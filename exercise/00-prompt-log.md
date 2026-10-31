**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** วรเมธ พงศาประเสริฐพร 
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: Gemini CLI (AGY)
**โมเดล:** Gemini 3.8 Flash
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: Gemini AI Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | | `prompts/1-…` run A | - `prompts/1-recover-architecture.md`<br> - `system/online-boutique/protos/demo.proto`| ใช่ | 2| ไม่ค้นพบ Frontend, Data Store, Runtime, Load Generator|
| 2 | 1-B | | `prompts/1-…` run B | - `prompts/1-recover-architecture.md`<br> -`system/online-boutique/protos/demo.proto`<br> -`system/online-boutique/kubernetes-manifests/kustomization.yaml`<br> - `system/online-boutique/kubernetes-manifests/frontend.yaml`<br> - `system/online-boutique/kubernetes-manifests/cartservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/recommendationservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/shippingservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/currencyservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/paymentservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/emailservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/adservice.yaml`<br> - `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`| ใช่ | 4.5 |พบทุก Services เจอ Frontend deployment เจอ Data Store Redis Cart เจอ Runtime Dependencies และ Load Generator |
| 3 | 2-a | | `prompts/2-…` run a | `prompts/2-write-adr.md- docs/adr/template.md` | ใช่ | 2.5 | ได้โครง ADR แต่มีแนวคิดที่กว้างไป ไม่มีตัวเลขจริงเพื่อวัดประสิทธิภาพของระบบ เช่น MailFest หรือ peak load |
| 4 | 2-b | | `prompts/2-…` run b | `prompts/2-… run b (context: scenario, adr-0001, checkoutservice)` | ใช่ | 5 | ADR มีความสมจริง ดูสามารถนำไปใช้ได้จริง มีการวิเคราะห์และนำตัวเลขจากโจทย์มาให้เห็นจริง เช่น peak 200 orders/s หรือ MailFast rate limit 50/s รวมถึง latency <= 500 ms|
| 5 | 3 | | `prompts/3-…` | - `prompts/3-review-adr.md`<br>- `docs/adr/0002-use-rabbitmq-for-post-order-processing.md`<br>- `docs/scenario.md`| ใช่ | 4.5 | รีวิวได้ครบถ้วน มีการระบุไว้ทั้ง Scenarios, Risk หรือเรื่อง rate limit ที่สามารถนำไปประชุมหารือเพื่อพัฒนาระบบต่อได้จริง |
| 6 | | | | | | | |
| 7 | | | | | | | |
