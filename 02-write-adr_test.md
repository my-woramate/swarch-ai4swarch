**ภาษา:** ภาษาไทย | [English](exercise/02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ฉบับกรอกข้อมูลสมบูรณ์)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`exercise/outputs/2a-adr.md`](exercise/outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: send the order confirmation email through MailFast, a new external email provider, which replaces the mock emailservice logic;
award loyalty points through a new loyaltyservice (to be built);
record an "order placed" analytics event for the marketing team.
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`exercise/outputs/2b-adr.md`](exercise/outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: send the order confirmation email through MailFast, a new external email provider, which replaces the mock emailservice logic;
award loyalty points through a new loyaltyservice (to be built);
record an "order placed" analytics event for the marketing team.
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ (ไม่มีตัวเลขจริง เช่น 200 orders/s, 50 emails/s, 500 ms) | ใช้ครบถ้วน (อ้างอิง latency p95 ≤ 500 ms, peak 200 orders/s, MailFast limit 50 emails/s และ failure 0.5%) |
| ทางเลือกที่พิจารณา (ระบุ) | Event-Driven Architecture, Synchronous API Orchestration, Outbox Pattern | Direct gRPC, RabbitMQ (Message Broker), Apache Kafka |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | กล่าวถึงแบบกว้างๆ ในเชิงทฤษฎีว่าควรทำ idempotency และ rate limiter | กำหนดชัดเจนให้ consumer ใช้ Token Bucket/prefetch คุมอัตราส่ง MailFast ≤ 50/s และใช้ order_id เป็น Idempotency key ใน loyaltyservice |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | มีการสมมุติตัวเลขและเครื่องมือที่ไม่ได้กำหนดไว้ในโจทย์ | ไม่มี ยึดตามข้อเท็จจริงใน scenario, stack Go และระบบ Kubernetes ของเดิม |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ให้ใช้ order_id ด้วย Idempotency key ใน loyaltyservice | เพื่อให้มั่นใจว่าจะไม่มีการแจกแต้มซ้ำซ้อน (exactly-once) แม้คิวจะส่ง retry |
| 2 | กำหนด RabbitMQ ให้ใช้ Quorum Queues บน K8S | เพื่อป้องกัน message loss และเพิ่มความมั่นคง High Availability (HA) หากมีโหนดล่ม |
| 3 | เพิ่ม DLQ (Dead Letter Queue) และระบบ Alert ให้กับคิวของอีเมล | เพื่อรองรับกรณีระบบ MailFast ล่มเกินจำนวนครั้งที่ retry ให้ทีมตรวจสอบและแก้ไขได้ |
