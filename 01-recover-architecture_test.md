**ภาษา:** ภาษาไทย | [English](exercise/01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ฉบับกรอกข้อมูลสมบูรณ์)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`exercise/outputs/1A-architecture.md`](exercise/outputs/1A-architecture.md)
- Run B: [`exercise/outputs/1B-architecture.md`](exercise/outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → paymentservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (PAYMENT_SERVICE_ADDR), `system/online-boutique/protos/demo.proto` | ✅ | ตรวจสอบใน `system/online-boutique/src/checkoutservice/main.go` พบฟังก์ชัน `chargeCard` เรียก `PaymentServiceClient.Charge` จริง |
| checkoutservice → emailservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (EMAIL_SERVICE_ADDR), `system/online-boutique/protos/demo.proto` | ✅ | ตรวจสอบใน `system/online-boutique/src/checkoutservice/main.go` พบฟังก์ชัน `sendOrderConfirmation` เรียก `EmailServiceClient.SendOrderConfirmation` จริง |
| checkoutservice → cartservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (CART_SERVICE_ADDR), `system/online-boutique/protos/demo.proto` | ✅ | ตรวจสอบใน `system/online-boutique/src/checkoutservice/main.go` พบการเรียก `CartServiceClient.GetCart` และ `EmptyCart` จริง |
| checkoutservice → shippingservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (SHIPPING_SERVICE_ADDR), `system/online-boutique/protos/demo.proto` | ✅ | ตรวจสอบใน `system/online-boutique/src/checkoutservice/main.go` พบการเรียก `ShippingServiceClient.GetQuote` และ `ShipOrder` จริง |
| cartservice → redis-cart | `system/online-boutique/kubernetes-manifests/cartservice.yaml` (REDIS_ADDR="redis-cart:6379") | ✅ | ใน manifest ของ cartservice มีการตั้งค่า environment variable ชี้ไปยัง Service redis-cart พอร์ต 6379 จริง |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ใน manifest ไม่ได้ระบุภาษาโปรแกรมหรือ runtime ของแต่ละ service และไม่เห็นพฤติกรรมการจัดการข้อผิดพลาดภายในโค้ด เช่น ไม่รู้ว่าหาก emailservice ล้มเหลว checkoutservice จะแค่เตือน warning และปล่อยผ่านโดยไม่ทำให้ order ล้มเหลว

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด:
   - agent ไม่สามารถรู้ทิศทางการเรียกใช้งาน (dependencies) ระหว่าง service, ไม่ค้นพบเว็บ frontend, data store (redis) หรือ load generator จากไฟล์ `.proto` เพียงไฟล์เดียว เพราะ `.proto` นิยามเฉพาะ interface contract ของแต่ละ service แต่ไม่ได้ระบุว่า service ใดเป็น caller หรือ callee และไม่ได้ระบุ deployment topology
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ:
   - การเข้าใจผิดว่า emailservice เป็น synchronous dependency ที่สำคัญและการันตีการส่งอีเมลสำเร็จ ทั้งที่ความจริงโค้ด checkoutservice แค่ log warning แล้วปล่อยผ่าน ทำให้เข้าใจระดับ reliability และ transaction boundary ของระบบผิดไป
