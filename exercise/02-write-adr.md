**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) |ไม่ใช้เลย เพราะไม่ได้อ่าน scenario เรื่องที่ไม่รู้ย้ายไปไว้ใน Open questions|ใช้ครบ: p95 ไม่เกิน 500 ms, peak 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5% / 50 emails/s, แต้มภายใน 1 นาทีและได้ครั้งเดียว และคำนวณ backlog 200 − 50 = 150 emails/s |
| ทางเลือกที่พิจารณา (ระบุ) |3 แบบ: (A) synchronous fan-out (B) event-driven pub/sub ผ่าน broker ← เลือก (C) orchestrator service ใหม่ โดยไม่ระบุว่าใช้ broker ตัวไหน |4 แบบ: (1) sync gRPC (2) async ภายใน process (goroutine) (3) RabbitMQ self-hosted ← เลือก (4) Kafka self-hosted |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร |แต้มใช้ order ID เป็น idempotency key (พูดแบบกว้าง ๆ) ส่วน rate limit ไม่รู้ตัวเลข มีแค่ retry + backoff + dead-letter queue |แต้มกันซ้ำด้วย order_id ใน loyaltyservice ส่วนอีเมลจำกัดอัตราไม่เกิน 50/s โดยให้ส่วนที่เกินรอในคิว แต่ยังมีความเสี่ยงที่ event หายถ้า publish ไม่สำเร็จ เพราะ checkout ไม่มี DB สำหรับ outbox |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) |แทบไม่มี แต่ยกตัวอย่างชื่อ broker (RabbitMQ, Kafka, Pub/Sub) และใส่ protocol HTTPS ให้ MailFast ใน diagram เอง |ใส่ภาษา Go / Node.js ให้ frontend, payment และ shipping ใน diagram (ติด [Inferred] ไว้) และข้อความ "อีเมลช้าเป็นนาทีถึงชั่วโมง" ที่ประมาณเอง เพราะ scenario ไม่ได้บอกว่าช่วงพีกยาวแค่ไหน ส่วนเลขบรรทัดใน main.go ที่อ้าง ถูกต้อง (ตรวจเอง: บรรทัด 233 มี uuid.NewUUID() และ log.Warnf อยู่ที่บรรทัด 274 ในช่วง 273–277)
และ Locust ก็มีที่มาจริงจาก ADR-0001 |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 |จาก ai เป็นอภิมุข สาโรวาท(ผมเอง)|ADR ต้องมีคนรับผิดชอบการตัดสินใจ |
| 2 |ภาษา เป็น no evidence |ไฟล์ที่ AI อ่านใน Run b ไม่ได้บอกภาษาของ frontend, payment และ shipping |
| 3 |เพิ่ม new db ใน architecture สำหรับ orders+ transactional outbox |AI เลือกแค่ retry + alert ซึ่งยังทำให้แต้มหายได้ แต่ scenario บอกว่าแต้มต้องได้ภายใน 1 นาทีและห้ามหาย |
