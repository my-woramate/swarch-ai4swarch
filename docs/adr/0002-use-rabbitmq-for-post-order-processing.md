# ADR-0002: Use RabbitMQ as a Message Broker for Asynchronous Post-Order Processing

- **Status:** Proposed
- **Date:** 2026-10-29
- **Deciders:** Software Architecture Team

## Context and problem

ในระบบ Online Boutique กระบวนการสั่งซื้อปัจจุบัน (`checkoutservice.PlaceOrder` ใน `system/online-boutique/src/checkoutservice/main.go`) ทำงานแบบ synchronous ผ่าน gRPC เรียงตามลำดับ ได้แก่ เรียก `paymentservice`, `shippingservice` และปิดท้ายด้วยการเรียก `emailservice.SendOrderConfirmation` ซึ่งปัจจุบันเป็น mock service โดยใช้เวลาประมวลผลคำสั่งซื้อรวมอยู่ที่ p95 ประมาณ 300 ms และหากส่งอีเมลล้มเหลว `checkoutservice` จะเพียงบันทึก log warning โดยคำสั่งซื้อยังถือว่าสำเร็จ

ความต้องการใหม่ (R-12) กำหนดให้เมื่อการสั่งซื้อเสร็จสมบูรณ์ ระบบต้องรองรับงาน 3 ส่วน ได้แก่:
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็น external email provider รายใหม่เพื่อทดแทน mock logic ของ `emailservice`
2. มอบแต้มสะสม (loyalty points) ผ่านบริการใหม่ **`loyaltyservice`** (กำลังจะพัฒนา)
3. บันทึก analytics event "order placed" สำหรับทีมการตลาด

นอกจากนี้ ทีมการตลาดยังมีแผนจะเพิ่มการทำงานหลังสั่งซื้อเพิ่มเติมในปีหน้า (เช่น SMS, fraud scoring, และ recommendations update)

ปัญหาคือระบบต้องรองรับเงื่อนไขทางธุรกิจและข้อจำกัดทางเทคนิค (ตาม `docs/scenario.md`) ดังต่อไปนี้:
- **Checkout latency:** p95 ของ `PlaceOrder` ต้องคงไว้ที่ **≤ 500 ms** (ปัจจุบันอยู่ที่ ~300 ms)
- **Peak load:** รองรับปริมาณคำสั่งซื้อสูงสุด **200 orders/s** ในช่วงเทศกาล 11.11
- **MailFast constraints:** ผู้ให้บริการภายนอกมี p95 สูงถึง **1.5 s** และ p99 ถึง **8 s**, มีอัตราความล้มเหลวเฉลี่ย **0.5%**, และจำกัด rate limit อยู่ที่ **50 emails/s** (ซึ่งต่ำกว่า peak load 200 orders/s มาก)
- **Loyalty points:** ต้องมอบแต้มสะสมให้ผู้ใช้**ภายใน 1 นาที**หลังสั่งซื้อ และต้องให้แบบ **exactly once** (ห้ามแจกแต้มซ้ำซ้อน)
- **Availability:** คำสั่งซื้อจะต้อง**ไม่ล้มเหลว**เด็ดขาด เพียงเพราะระบบ email, loyalty หรือ analytics ขัดข้อง
- **Team & Operations:** ทีมงานมีนักพัฒนา 4 คน มีความเชี่ยวชาญใน Go และ gRPC โดยมี 2 คนเคยใช้งาน **RabbitMQ** มาก่อน ระบบทั้งหมดรันอยู่บน Kubernetes และมีนโยบายเลือกใช้ส่วนประกอบที่เป็น **open-source, self-hosted** มากกว่า managed service บนคลาวด์

## Decision drivers

1. **Availability:** กระบวนการสั่งซื้อหลักต้องไม่ล่ม หากระบบ post-order บริการใดบริการหนึ่งล่มหรือตอบสนองช้า
2. **Performance (Latency & Throughput):** p95 ของ `PlaceOrder` ต้องไม่เกิน 500 ms และต้องสามารถ buffer โหลดช่วงพีก 200 orders/s เพื่อไม่ให้เกิน rate limit 50 emails/s ของ MailFast
3. **Modifiability / Extensibility:** สามารถเพิ่มผู้รับ event (SMS, fraud scoring, recommendations) ในปีหน้าได้ง่าย โดยไม่ต้องแก้โค้ดใน `checkoutservice`
4. **Reliability & Consistency:** ต้องรองรับการทำงานแบบ retry เมื่อ MailFast ล้มเหลว และรองรับ idempotent processing เพื่อให้แต้มสะสมแบบ exactly once ภายใน 1 นาที
5. **Operability & Team Skill:** ระบบต้องสามารถติดตั้งแบบ self-hosted บน Kubernetes ได้โดยตรง และเหมาะสมกับทีมขนาด 4 คนที่มีทักษะ Go, gRPC และมีประสบการณ์ RabbitMQ

## Considered options

1. **Direct Synchronous / Asynchronous gRPC Calls from `checkoutservice`:** ให้ `checkoutservice` เรียก `emailservice`, `loyaltyservice`, และ analytics service โดยตรงผ่าน gRPC (อาจเรียกแบบ sync หรือแยก goroutine ภายใน service)
2. **Message Broker with Publish/Subscribe using RabbitMQ:** ใช้ RabbitMQ เป็น message broker แบบ self-hosted บน Kubernetes โดย `checkoutservice` ทำหน้าที่ publish ข้อความ `OrderPlaced` เข้า AMQP Exchange (fanout หรือ topic) จากนั้นแต่ละ consumer queue (email, loyalty, analytics) จะดึงข้อความไปประมวลผลแยกกันแบบ asynchronous
3. **Distributed Event Streaming Platform using Apache Kafka:** ติดตั้ง Apache Kafka (และ Zookeeper หรือ KRaft) บน Kubernetes เพื่อทำ persistent event stream สำหรับ `OrderPlaced` event

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option 1: Direct gRPC Calls** | • ไม่ต้องติดตั้งและดูแลระบบ infrastructure เพิ่มเติม<br>• ทีมคุ้นเคยกับ gRPC เป็นอย่างดี<br>• สถาปัตยกรรมเรียบง่าย ไม่มีความซับซ้อนของ distributed queue | • ไม่ผ่านเกณฑ์ Performance: Latency ของ MailFast (p95 1.5 s, p99 8 s) จะทำให้ `PlaceOrder` latency เกิน 500 ms ทันทีหากเรียกแบบ sync<br>• หากใช้ goroutine แบบ fire-and-forget ใน memory ข้อมูลจะสูญหายทันทีหาก pod ของ `checkoutservice` restart หรือ crash<br>• ไม่สามารถควบคุม rate limit 50 emails/s ของ MailFast ในช่วง peak 200 orders/s ได้ ส่งผลให้เกิดข้อผิดพลาด 429 Too Many Requests จำนวนมาก<br>• ขาด Modifiability: การเพิ่ม service ใหม่ในปีหน้าต้องกลับมาแก้ไขโค้ดและ deploy `checkoutservice` ซ้ำ |
| **Option 2: RabbitMQ (Message Broker)** | • Latency ต่ำมากในการ publish event เข้า RabbitMQ (~1–5 ms) ช่วยรักษา p95 ของ `PlaceOrder` ให้ต่ำกว่า 500 ms ได้สบาย<br>• ทำหน้าที่เป็น buffer ดูดซับ traffic พีก 200 orders/s และช่วยควบคุม rate limit ให้ consumer ทยอยส่ง MailFast ไม่เกิน 50 emails/s ได้อย่างสมบูรณ์<br>• มี retry mechanism และ Dead Letter Queue (DLQ) สำหรับ 0.5% ที่ MailFast ล้มเหลว<br>• Decoupling สมบูรณ์ (Modifiability): ปีหน้าสามารถเพิ่ม consumer สำหรับ SMS, fraud scoring ได้ทันทีโดยผูกคิวใหม่เข้า exchange เดิม ไม่ต้องแตะ `checkoutservice`<br>• ทีมงาน 2 ใน 4 คนมีประสบการณ์ RabbitMQ อยู่แล้ว และเป็นซอฟต์แวร์ open-source น้ำหนักเบาที่รันบน Kubernetes ได้สะดวก | • เพิ่มภาระในการ deploy และดูแล stateful pod ของ RabbitMQ บน Kubernetes<br>• ต้องจัดการกลไก Idempotency (Deduplication) ที่ฝั่ง `loyaltyservice` เพื่อป้องกันปัญหาข้อความซ้ำจากการ retry (at-least-once delivery) เพื่อให้แต้มสะสมแบบ exactly-once ตามเงื่อนไข |
| **Option 3: Apache Kafka** | • รองรับ throughput ปริมาณมหาศาล และเก็บ event stream ไว้ได้ยาวนาน (event replay)<br>• รองรับ Pub/Sub แบบ decoupled เช่นเดียวกัน | • มีความซับซ้อนในการติดตั้ง ดูแลรักษา และใช้ทรัพยากรสูงเกินจำเป็น (Overengineering) สำหรับปริมาณงาน 200 orders/s<br>• ไม่มีสมาชิกในทีมคนใดระบุว่ามีประสบการณ์กับ Kafka มาก่อน (ทีมมีเพียง 4 คน)<br>• การควบคุม rate limit ราย consumer และการทำ message-level retry/DLQ ใน Kafka ทำได้ซับซ้อนกว่า AMQP queue ของ RabbitMQ |

## Decision

เลือก **Option 2: ใช้ RabbitMQ เป็น Message Broker** สำหรับกระจาย event หลังการสั่งซื้อแบบ Asynchronous Event-Driven Architecture

**แนวทางการออกแบบและการทำงาน:**
1. **Event Publishing:** เมื่อ `checkoutservice.PlaceOrder` บันทึกการชำระเงินและจัดการสินค้าสำเร็จแล้ว จะแปลงข้อมูลคำสั่งซื้อเป็น event `OrderPlaced` (มี `order_id`, `user_id`, รายการสินค้า, ข้อมูลที่อยู่ และยอดรวม) แล้ว publish ไปยัง RabbitMQ Exchange แบบ fanout หรือ topic ผ่าน AMQP protocol
2. **Buffer & Decoupling:** `checkoutservice` จะไม่ต้องรอผลการทำงานของ external provider หรือ service ปลายทาง ทำให้ `PlaceOrder` p95 ยังคงอยู่ที่ ~300 ms (ไม่เกินข้อกำหนด 500 ms)
3. **Queue & Consumers:**
   - **`emailservice` queue:** เป็น consumer ที่ดึง event ไปแปลงและเรียก MailFast API โดยตั้งค่า worker pool / prefetch count ให้จำกัดอัตราการส่งไม่เกิน **50 emails/s** เพื่อไม่ให้ชน rate limit ของ MailFast และหากเรียกไม่สำเร็จ (ตาม failure rate 0.5%) ให้ส่งเข้ากระบวนการ retry ด้วย exponential backoff และ DLQ
   - **`loyaltyservice` queue:** ดึง event เพื่อคำนวณและมอบแต้มสะสม โดย consumer จะประมวลผลทันทีซึ่งใช้เวลาไม่กี่มิลลิวินาที จึงการันตีว่าเสร็จสิ้น**ภายใน 1 นาที**หลังการสั่งซื้อแน่นอน พร้อมทั้งออกแบบให้ใช้ `order_id` ทำ Idempotency check (บันทึกลง local DB / Redis) เพื่อรับประกันว่าแต้มจะถูกให้เพียงครั้งเดียว (exactly-once semantics ในระดับ application)
   - **`analytics` queue:** ดึง event เพื่อบันทึกลงระบบวิเคราะห์ของทีมการตลาด
4. **Operations:** ติดตั้ง RabbitMQ Cluster Operator บน Kubernetes ในคลัสเตอร์เดิม สอดคล้องกับแนวทาง open-source และ self-hosted ขององค์กร

## Consequences

- **Positive (Availability):** ความพร้อมใช้งานของระบบสั่งซื้อไม่ขึ้นอยู่กับ MailFast, `loyaltyservice`, หรือ analytics หากระบบเหล่านี้ล่ม คำสั่งซื้อของลูกค้ายังคงสำเร็จ 100%
- **Positive (Performance):** รักษา latency ของ `PlaceOrder` ให้ต่ำกว่า 500 ms (latency ของการ publish message อยู่ที่ระดับ millisecond) และรองรับ peak load 200 orders/s ได้อย่างปลอดภัย
- **Positive (Modifiability):** ยืดหยุ่นสูงต่อความต้องการในอนาคต ปีหน้าทีมการตลาดสามารถสร้าง service ใหม่ (SMS, fraud scoring, recommendations) แล้วเพิ่มคิวรับฟัง event `OrderPlaced` ได้ทันทีโดยไม่ต้องแก้ไขหรือกระทบ `checkoutservice`
- **Positive (Operability & Cost):** RabbitMQ เป็น open-source น้ำหนักเบา รันบน Kubernetes ได้โดยตรง ไม่มีค่าใช้จ่าย managed service รายเดือน และมีทีมงานครึ่งหนึ่งที่มีประสบการณ์ใช้งานมาก่อน
- **Negative (Operational Complexity):** ต้องดูแลรักษา stateful component เพิ่มขึ้นอีกหนึ่งตัว (RabbitMQ cluster, persistent storage บน Kubernetes, monitoring, alerting) (quality attribute: operability)
- **Negative (Eventual Consistency & Complexity):** การทำงานกลายเป็นแบบ eventual consistency ผู้ใช้อาจไม่ได้รับอีเมลยืนยันในทันทีระดับวินาที (แม้ว่าอีเมลจะส่งตามหลังมา) และนักพัฒนาต้องเขียนโค้ดรองรับ idempotency และ dead-letter handling (quality attribute: testability, reliability)
- **Risks and mitigations:**
  - *Risk:* หาก RabbitMQ broker ล่ม ข้อความอาจค้างหรือไม่สามารถ publish ได้
  - *Mitigation:* ติดตั้ง RabbitMQ แบบ High Availability (HA) คลัสเตอร์ (เช่น Quorum Queues) บน Kubernetes และพิจารณาใช้ Local Outbox Pattern หรือ retry logic ฝั่ง `checkoutservice` ก่อน fallback
  - *Risk:* การส่งข้อความซ้ำจากการ retry ของ RabbitMQ ทำให้แจกแต้มสะสมซ้ำซ้อน
  - *Mitigation:* บังคับใช้ unique constraint บน `order_id` ภายใน `loyaltyservice` database เพื่อทำ Idempotent Consumer อย่างรัดกุม

## Validation

1. **Latency & Throughput Load Test:** ใช้เครื่องมือทดสอบโหลด (เช่น Locust `loadgenerator`) ยิงโหลดที่ 200 orders/s ต่อเนื่องเป็นเวลา 15 นาที เพื่อตรวจสอบว่า:
   - p95 latency ของ `PlaceOrder` คงอยู่ที่ ≤ 500 ms
   - คิวใน RabbitMQ ดูดซับ event ได้โดยไม่เกิด message loss
2. **Rate Limit Verification:** ตรวจสอบว่า `emailservice` เรียก MailFast ไม่เกิน 50 req/s ตลอดช่วง peak load และไม่มีข้อผิดพลาด 429 จาก MailFast
3. **Resilience & SLA Test:**
   - จำลองสถานการณ์ MailFast ล่ม/ช้า (หน่วงเวลา 8 s และ inject error 0.5%) เพื่อยืนยันว่าไม่มีผลกระทบต่อ `PlaceOrder` และข้อความที่ผิดพลาดถูก retry สำเร็จ
   - ทดสอบจำลองการส่ง event ซ้ำไปยัง `loyaltyservice` และยืนยันว่าผู้ใช้ได้รับแต้มสะสมเพียงครั้งเดียว และกระบวนการเสร็จสิ้นภายใน 60 วินาที

## Open questions

1. **Storage/Database สำหรับ `loyaltyservice`:** ใน scenario ยังไม่ได้ระบุเทคโนโลยีฐานข้อมูลที่ `loyaltyservice` จะใช้เก็บแต้มสะสมและทำ idempotency key (จะใช้ Redis เดิมของระบบ หรือต้องการฐานข้อมูลแบบ Relational เช่น PostgreSQL)
2. **MailFast Error Handling Policy:** หาก retry ส่งอีเมลจนครบกำหนด (exhausted retries) แล้วข้อความตกไปที่ DLQ ทีมมีกระบวนการ alert และแก้ไข manual หรือไม่ อย่างไร
3. **Analytics Destination:** ทีมการตลาดต้องการให้ analytics event ส่งต่อไปยังปลายทางใด (เช่น BigQuery, Elasticsearch, หรือ Third-party webhook)
