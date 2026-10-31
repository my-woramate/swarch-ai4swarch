<!-- Agent: Gemini CLI / Antigravity | Model: Gemini 3.8 Flash -->

# Architecture Review: ADR-0002 (ATAM Style)

- **Review Target:** `docs/adr/0002-use-rabbitmq-for-post-order-processing.md`
- **Reviewer:** Architecture Review Board (Chief Analyst Role)
- **Evaluation Method:** ATAM (Architecture Tradeoff Analysis Method)

---

## 1. Quality-attribute scenarios

1. **Scenario 1 (Peak Load & Checkout Latency):**
   - *Source:* ผู้ใช้งานจำนวนมากพร้อมกัน
   - *Stimulus:* ปริมาณคำสั่งซื้อพุ่งแตะช่วงพีก **200 orders/s** ในช่วงเทศกาล 11.11
   - *Environment:* ระบบทำงานบน Kubernetes ภายใต้โหลดสูงสุด
   - *Response:* `checkoutservice` บันทึกการชำระเงิน ตัดยอด และส่ง event `OrderPlaced` เข้า RabbitMQ แบบ asynchronous โดยไม่บล็อกรอการส่งอีเมลหรือการแจกแต้มสะสม
   - *Response measure:* p95 latency ของ `PlaceOrder` คงอยู่ที่ **≤ 500 ms** (คงระดับ ~300 ms ตามเกณฑ์เดิม) และไม่มีข้อผิดพลาดของระบบสั่งซื้อหลัก

2. **Scenario 2 (External Rate Limiting & Fault Tolerance):**
   - *Source:* Event `OrderPlaced` ที่สะสมในคิวของ `emailservice`
   - *Stimulus:* ปริมาณ event หลั่งไหลเข้ามา 200 events/s เข้าสู่คิวส่งอีเมล ขณะที่ MailFast มีข้อจำกัดอัตราการส่งที่ **50 emails/s**, latency p95 อยู่ที่ **1.5 s** (p99 ถึง **8 s**) และมีอัตราล้มเหลวเฉลี่ย **0.5%**
   - *Environment:* บริการภายนอกมีความหน่วงสูงและมีข้อจำกัดด้าน rate limit
   - *Response:* Consumer ของ `emailservice` ควบคุมอัตราการดึงและยิง API ไม่ให้เกิน 50 req/s พร้อมส่งข้อความที่ล้มเหลวเข้ากระบวนการ retry แบบ exponential backoff และตัดเข้า Dead Letter Queue (DLQ) เมื่อล้มเหลวถาวร
   - *Response measure:* อัตราการเรียก MailFast ไม่เกิน 50 emails/s ตลอดเวลา, ไม่เกิดข้อผิดพลาด HTTP 429 Too Many Requests, และไม่มีข้อความคำสั่งซื้อสูญหาย

3. **Scenario 3 (Data Consistency & Loyalty Timeliness):**
   - *Source:* การสั่งซื้อเสร็จสมบูรณ์
   - *Stimulus:* Event `OrderPlaced` ถูกส่งไปยัง `loyaltyservice` โดยอาจมีการส่งข้อความซ้ำจากการ retry ในระดับ network หรือ broker
   - *Environment:* สภาวะโหลดสูงและมีความเป็นไปได้ของ network jitter
   - *Response:* `loyaltyservice` ตรวจสอบความซ้ำซ้อนด้วย `order_id` และคำนวณแต้มสะสมให้บัญชีผู้ใช้
   - *Response measure:* ผู้ใช้ได้รับแต้มสะสม**ภายใน 1 นาที**หลังการสั่งซื้อ และได้รับแต้มเพียงครั้งเดียวเท่านั้น (strictly **exactly once**)

---

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | การเลือกสถาปัตยกรรม Asynchronous Event-Driven ด้วย RabbitMQ แยกงานหลังการสั่งซื้อ (MailFast, Loyalty, Analytics) ออกจากกระบวนการหลักของ `checkoutservice` | เป็นการตัดสินใจที่ถูกต้องอย่างยิ่ง ช่วยตัดผลกระทบจากความหน่วงระดับวินาที (1.5–8 s) และอัตรา failure ของ MailFast ออกจากเส้นทางการสั่งซื้อ ทำให้รักษา SLA latency (p95 ≤ 500 ms) และการันตี Availability ของระบบสั่งซื้อหลักได้อย่างเด็ดขาด |
| **Risk** | RabbitMQ กลายเป็น Single Point of Failure (SPOF) ตัวใหม่สำหรับกระบวนการส่ง event หาก RabbitMQ cluster ล่ม และ `checkoutservice` ไม่มี local fallback หรือ outbox mechanism | อาจทำให้ `checkoutservice` เกิด error หรือหากตัดใจทิ้ง event ไป คำสั่งซื้อจะสำเร็จโดยที่ลูกค้าไม่ได้อีเมลยืนยันและไม่ได้แต้มสะสม ซึ่งกระทบต่อความน่าเชื่อถือของธุรกิจอย่างรุนแรง |
| **Sensitivity point** | การตั้งค่า Concurrency / Prefetch count และ Rate limiter ของ `emailservice` consumer | ค่าพารามิเตอร์นี้มีความอ่อนไหวสูงมาก หากตั้งค่าสูงเกินไป คิวจะปล่อย request ยิงไปเกิน 50 req/s ทำให้ MailFast ปฏิเสธ (429) แต่หากตั้งค่าต่ำเกินไป ข้อความในคิวช่วงพีก 200 orders/s จะสะสมตกค้างจนผู้ใช้ได้รับอีเมลยืนยันล่าช้าเป็นชั่วโมง |
| **Sensitivity point** | Throughput และเวลาประมวลผลของ `loyaltyservice` ในช่วงโหลด 200 orders/s | ความต้องการทางธุรกิจระบุว่าต้องให้แต้มสะสม "ภายใน 1 นาที" หากคิวของ `loyaltyservice` มี latency การประมวลผลหรือคอขวดที่ database จนระบายข้อความไม่ทันภายใน 60 วินาที จะถือว่าละเมิด SLA ของธุรกิจทันที |
| **Trade-off point** | การเลือกระหว่าง Decoupling/Extensibility กับ Operational Complexity | การใช้ RabbitMQ ทำให้ระบบมีความยืดหยุ่นสูง (Modifiability) สามารถเพิ่ม SMS หรือ fraud scoring ในปีหน้าได้ง่าย แต่ต้องแลกมาด้วยความซับซ้อนในการติดตั้ง ดูแลรักษา และมอนิเตอร์ stateful cluster บน Kubernetes สำหรับทีมงานขนาดเล็กเพียง 4 คน |

---

## 3. Missing or weak

1. **ขาดรายละเอียด Transactional Outbox Pattern หรือ Local Persistence:** ADR ระบุว่า `checkoutservice` จะ publish event ไปยัง RabbitMQ ทันที แต่ไม่ได้กล่าวถึงกรณีที่การชำระเงินตัดบัตรสำเร็จแล้วแต่การเชื่อมต่อไปยัง RabbitMQ ขัดข้อง หากไม่มี outbox pattern หรือ local retry buffer คำสั่งซื้อนั้นจะเสี่ยงต่อการเกิด event สูญหายถาวร
2. **ไม่ระบุเทคโนโลยี Data Store สำหรับ `loyaltyservice`:** ADR ระบุเรื่องการใช้ `order_id` ทำ idempotency check แต่ไม่ได้ระบุเทคโนโลยีฐานข้อมูล (เช่น Redis หรือ Relational DB) และไม่ได้คำนวณว่า database ดังกล่าวจะรองรับ write throughput 200 IOPS พร้อม constraint check ได้ทันเวลา 1 นาทีหรือไม่
3. **การจัดการ Queue Backlog และ Alerting ในระดับปฏิบัติการ:** ในช่วงพีก 200 orders/s คิวอีเมลจะมีข้อความสะสมอย่างรวดเร็ว (เนื่องจากระบายได้ 50 emails/s) ADR ไม่ได้ระบุขีดจำกัดขนาดของ Persistent Volume บน Kubernetes และขาดแนวทางการแจ้งเตือนเมื่อเกิดคิวตกค้างสะสมเกินเกณฑ์ปกติ

---

## 4. Verdict

**Verdict:** **Accept with changes**

การตัดสินใจเลือกใช้ RabbitMQ เพื่อทำ Event-Driven Architecture มีความสมเหตุสมผลและตอบโจทย์ปัญหาคอขวดด้าน latency และ rate limit ได้อย่างตรงจุด คณะกรรมการเห็นชอบในหลักการโดยมีเงื่อนไขให้ปรับปรุงเอกสารและแบบการติดตั้ง 3 ประการดังนี้:

1. **เพิ่มกลไก Local Fallback / Outbox ใน `checkoutservice`:** ออกแบบแนวทางสำรองในกรณีที่ RabbitMQ ไม่สามารถรับข้อความได้ชั่วขณะ เพื่อรับประกันว่าไม่มี event คำสั่งซื้อที่ชำระเงินแล้วสูญหาย
2. **ระบุสเปก Data Store และการทดสอบเวลา 1 นาทีของ `loyaltyservice`:** ระบุฐานข้อมูลที่จะใช้ทำ Idempotency key และทำการทดสอบ stress test ยืนยันว่าการมอบแต้มสะสมจะประมวลผลเสร็จสิ้นภายในกรอบเวลา 60 วินาทีที่กำหนดในช่วงพีก
3. **กำหนดกลไก Rate Limiting ที่ชัดเจนใน `emailservice`:** ระบุเทคนิค rate limiting ที่ระดับ application consumer (เช่น Token Bucket algorithm ร่วมกับ RabbitMQ QoS Prefetch) เพื่อรับประกันอย่างเคร่งครัดว่าอัตราการเรียก MailFast จะไม่มีทางเกิน 50 requests/s