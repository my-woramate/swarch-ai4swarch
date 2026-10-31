<!-- Agent: Gemini CLI / Antigravity | Model: Gemini 3.8 Flash -->

# ADR-0002: Use Event-Driven Architecture with a Message Broker for Post-Order Processing

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Software Architecture Team

## Context and problem

ระบบ Online Boutique มีความต้องการใหม่ (R-12) สำหรับกระบวนการหลังจากลูกค้าสั่งซื้อสินค้าเรียบร้อยแล้ว (post-order processing) โดยต้องดำเนินงาน 3 ประการ:
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast ซึ่งเป็น external email provider รายใหม่เพื่อทดแทน mock `emailservice` เดิม
2. มอบแต้มสะสม (loyalty points) ผ่าน service ใหม่ชื่อ `loyaltyservice` ที่กำลังจะพัฒนา
3. บันทึก analytics event "order placed" สำหรับทีมการตลาด

นอกจากนี้ ทีมการตลาดยังมีแผนจะเพิ่มการทำงานหลังการสั่งซื้อเพิ่มเติมในปีหน้า เช่น การส่ง SMS, การตรวจจับทุจริต (fraud scoring) และการอัปเดตระบบแนะนำสินค้า (recommendations update) 

ปัญหาคือเราควรออกแบบการเชื่อมต่อและการกระจายงานไปยังบริการเหล่านี้อย่างไร เพื่อไม่ให้กระทบต่อกระบวนการสั่งซื้อหลัก และรองรับการขยายตัวในอนาคตได้อย่างยืดหยุ่น

## Decision drivers

1. **Modifiability & Extensibility:** ความสะดวกในการเพิ่มบริการหลังการสั่งซื้อใหม่ ๆ (SMS, fraud scoring, recommendation) ในปีหน้า โดยไม่ต้องแก้ไขโค้ดของ `checkoutservice`
2. **Availability & Fault Isolation:** ความล้มเหลวหรือความล่าช้าของบริการภายนอก (MailFast) หรือบริการเสริม (loyalty, analytics) ต้องไม่ทำให้การสั่งซื้อของลูกค้าล้มเหลว
3. **Performance (Latency):** เวลาตอบสนองต่อผู้ใช้ในการสั่งซื้อสินค้าต้องรวดเร็ว ไม่ถูกหน่วงโดยกระบวนการหลังการสั่งซื้อ
4. **Reliability & Delivery Guarantees:** ต้องมั่นใจได้ว่าคำสั่งซื้อที่เกิดขึ้นจะไม่สูญหาย และได้รับการประมวลผลแต้มสะสมและอีเมลอย่างถูกต้อง

## Considered options

1. **Option A: Synchronous Calls (gRPC / REST) Orchestrated by CheckoutService**
   ให้ `checkoutservice` เรียกไปยัง MailFast (หรือ wrapper service), `loyaltyservice` และ analytics service โดยตรงแบบ synchronous ทีละบริการหลังรับคำสั่งซื้อ
2. **Option B: Event-Driven Architecture using an Asynchronous Message Broker (Publish-Subscribe)**
   เมื่อคำสั่งซื้อสำเร็จ ให้ `checkoutservice` เผยแพร่เหตุการณ์ (publish event) `OrderPlaced` ไปยัง Message Broker โดยมี MailFast adapter/service, `loyaltyservice` และ analytics service สมัครรับข้อมูล (subscribe) แยกกันเพื่อประมวลผลแบบ asynchronous
3. **Option C: In-Process Background Worker / Goroutine Dispatch**
   ให้ `checkoutservice` รันงานเบื้องหลังแบบ asynchronous ภายใน memory/process เดียวกันผ่าน background worker หรือ goroutine/thread เพื่อเรียกบริการภายนอก

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option A: Synchronous Calls** | • สถาปัตยกรรมเรียบง่าย ไม่ต้องเพิ่ม infrastructure ภายนอก<br>• ติดตามข้อผิดพลาดและ debug ได้ง่ายตาม call stack | • **availability ต่ำ:** หาก MailFast หรือบริการปลายทางล่ม อาจทำให้ checkout ล้มเหลวหรือค้าง<br>• **performance แย่:** latency ของ checkout เพิ่มขึ้นตามผลรวมของ downstream ทั้งหมด<br>• **modifiability ต่ำ:** เพิ่มฟีเจอร์ใหม่ปีหน้าต้องแก้ไขและ redeploy `checkoutservice` เสมอ |
| **Option B: Event-Driven Architecture (Message Broker)** | • **modifiability สูงมาก:** เพิ่ม subscriber ใหม่ (SMS, fraud scoring) ปีหน้าได้อิสระโดยไม่ต้องแตะ `checkoutservice`<br>• **availability & fault isolation ดีเยี่ยม:** downstream ล่ม ไม่กระทบการ checkout<br>• **performance สูง:** `checkoutservice` publish event แล้วตอบกลับผู้ใช้ได้ทันที<br>• มีคิวช่วย buffer โหลดเมื่อมี traffic พุ่งสูง | • **operational complexity เพิ่มขึ้น:** ต้องติดตั้ง ดูแลรักษา และมอนิเตอร์ระบบ Message Broker<br>• ต้องจัดการความสอดคล้องของข้อมูลแบบ eventual consistency และการ retry เมื่อเกิดข้อผิดพลาด |
| **Option C: In-Process Background Worker** | • latency ของ checkout ยังคงเร็ว<br>• ไม่ต้องติดตั้ง message broker เพิ่มเติม | • **reliability ต่ำมาก:** หากคอนเทนเนอร์ของ `checkoutservice` เกิด crash หรือ restart ระหว่างทำงาน งานที่อยู่ใน memory จะสูญหายทันที<br>• **modifiability จำกัด:** ยังคงผูกติดอยู่กับ lifecycle ของ `checkoutservice` |

## Decision

เลือก **Option B: Event-Driven Architecture using an Asynchronous Message Broker (Publish-Subscribe)**

**เหตุผล:**
เนื่องจากสอดคล้องกับ Decision Drivers มากที่สุด โดยเฉพาะด้าน **Modifiability** ที่ทีมการตลาดระบุชัดเจนว่าจะมีฟีเจอร์ใหม่ ๆ ตามมาอีกมากในปีหน้า การใช้ Event-Driven ผ่าน Publish-Subscribe ช่วยตัด coupling ระหว่าง `checkoutservice` กับ downstream consumers อย่างสมบูรณ์ นอกจากนี้ยังช่วยปกป้องระบบ checkout จากความล่าช้าและความล้มเหลวของบริการภายนอกอย่าง MailFast (ยกระดับ **Availability** และ **Performance**)

## Consequences

- **Positive:**
  - **Modifiability:** ทีมสามารถพัฒนาและเปิดใช้งาน consumer ใหม่ (SMS, fraud scoring, recommendation updates) ในปีหน้าได้อย่างอิสระโดยไม่ต้องแก้ไขหรือทดสอบ `checkoutservice` ซ้ำ
  - **Availability:** ปัญหาเครือข่ายหรือ downtime ของ MailFast หรือบริการเสริม จะไม่ขัดขวางการทำรายการสั่งซื้อของลูกค้า
  - **Performance:** เวลาตอบสนอง (latency) ของการสั่งซื้อไม่ถูกถ่วงด้วยเวลาการส่งอีเมลหรือการคำนวณแต้มสะสม
- **Negative:**
  - **Operational Cost & Complexity (Operability):** ต้องจัดหา ติดตั้ง ดูแล และบริหารจัดการคลัสเตอร์ Message Broker เพิ่มในระบบ
  - **Reliability:** ต้องพัฒนากลไกการจัดการข้อผิดพลาดเพิ่มเติม เช่น Dead Letter Queue (DLQ), การ retry แบบ exponential backoff, และ idempotent consumer เพื่อป้องกันการประมวลผลซ้ำ
- **Risks and mitigations:**
  - *Risk:* Message Broker ล่มทำให้ไม่สามารถ publish event ได้
    - *Mitigation:* ใช้รูปแบบ Transactional Outbox Pattern หรือ retry ในฝั่งผู้ส่ง พร้อมตั้งค่าคลัสเตอร์ของ Broker แบบ High Availability (HA)
  - *Risk:* Consumer ประมวลผลข้อความซ้ำจากการ retry ส่งผลให้ให้แต้มสะสมเบิ้ล
    - *Mitigation:* บังคับใช้ Idempotency Key (เช่น `order_id`) ใน `loyaltyservice` เพื่อตรวจสอบก่อนบันทึกแต้ม

## Validation

1. ทำการทดสอบ Integration Test และ Chaos Test โดยจำลองสถานการณ์ MailFast ล่ม หรือ latency สูง แล้วตรวจสอบว่า `checkoutservice` ยังคงยืนยันคำสั่งซื้อได้สำเร็จตามปกติ
2. จำลองการเพิ่ม subscriber จำลองตัวใหม่ เพื่อยืนยันว่าสามารถรับ event `OrderPlaced` ได้โดยไม่ต้องแก้ไขหรือรีสตาร์ต `checkoutservice`
3. ติดตั้ง metrics วัด message throughput, consumer lag และเวลาตั้งแต่ publish จนกระทั่ง consumer ประมวลผลเสร็จสิ้น

## Open questions

*(เนื่องจากข้อกำหนด R-12 ไม่ได้ระบุตัวเลขและบริบทแวดล้อม จึงจำเป็นต้องสอบถามข้อมูลเพิ่มเติมดังต่อไปนี้)*

1. **Throughput & Peak Load:** ปริมาณคำสั่งซื้อเฉลี่ยและช่วงเวลาที่มีโหลดสูงสุด (peak orders per second) มีตัวเลขประมาณเท่าใด เพื่อใช้เลือกและปรับขนาดของ Message Broker?
2. **Latency Budget:** เป้าหมายเวลาในการตอบสนอง (SLA/p95 latency) ของ `checkoutservice` มีขีดจำกัดไม่เกินกี่มิลลิวินาที?
3. **MailFast Constraints:** MailFast มีข้อจำกัดเรื่อง rate limit, latency ทั่วไป หรือค่าใช้จ่ายต่อ API call อย่างไร?
4. **Loyalty Points Consistency:** การให้แต้มสะสมจำเป็นต้องเป็น exactly-once อย่างเคร่งครัดหรือไม่ และมีกรอบเวลาที่ยอมรับได้ (เช่น ภายในกี่นาทีหลังสั่งซื้อ)?
5. **Team Skill & Operations:** ทีมงานมีความคุ้นเคยหรือเชี่ยวชาญเทคโนโลยี Message Broker ใดเป็นพิเศษ (เช่น RabbitMQ, Apache Kafka, NATS, หรือ Cloud Pub/Sub) และมีข้อจำกัดด้าน infrastructure ที่ใช้งานอยู่ในปัจจุบันหรือไม่?
