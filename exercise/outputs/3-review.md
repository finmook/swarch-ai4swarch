Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 (`docs/adr/0002-post-order-events.md`)

ไฟล์ที่ใช้: ADR นี้ และ `docs/scenario.md` เท่านั้น การอ้างอิงบรรทัดใน `main.go` ที่ ADR ใส่ไว้ ผู้รีวิวตรวจสอบไม่ได้ (อยู่นอกขอบเขตไฟล์ที่อนุญาต) จึงถือเป็นข้อกล่าวอ้างของผู้เขียน

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 (performance) | ลูกค้า | เรียก `PlaceOrder` | peak 11.11 ที่ **200 orders/s**, MailFast p95 1.5 s / p99 8 s | `checkoutservice` บันทึก order + event แล้วตอบ client โดยไม่รอ post-order actions | p95 `PlaceOrder` **≤ 500 ms** (ปัจจุบันประมาณ 300 ms) |
| QA2 (availability) | MailFast / `loyaltyservice` / analytics / RabbitMQ | ล่มหรือตอบ error | ช่วง peak | order ยังสำเร็จ; event ไม่หาย และถูก process เมื่อระบบกลับมา | order fail เพราะสาเหตุนี้ **0 %**; event หาย **0** |
| QA3 (correctness) | RabbitMQ | ส่ง `OrderPlaced` ซ้ำ (redelivery) | ช่วง peak, consumer restart | `loyaltyservice` ให้แต้มครั้งเดียว | แต้มเข้า **ภายใน 1 นาที**, duplicate **0** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | **Decision ขัดกันเอง:** bullet แรกบอก publish ตรงไป RabbitMQ แล้ว "retry สั้น ๆ + log error" แต่ bullet สุดท้ายบอก "จะใช้ order DB + transactional outbox" และ diagram วาดทั้งสองเส้นทาง (`checkout → mq` และ `checkout → orderdb`) | ผู้ implement ไม่รู้ว่าต้องสร้างอะไร; ถ้าใช้ทางแรก event หายเมื่อ RabbitMQ ล่ม ซึ่งผิด QA2 และ requirement loyalty "exactly once" |
| Risk | **Outbox ยังไม่ปิดช่องว่างทั้งหมด:** ตัดเงิน (`chargeCard`) เกิดก่อนเขียน DB; ถ้า pod ตายระหว่างนั้น เงินถูกตัดแต่ไม่มี order/event | loyalty และอีเมลหายสำหรับ order ที่จ่ายเงินแล้ว; ADR ไม่กล่าวถึง |
| Risk | **Order DB กลายเป็น dependency แบบ synchronous ใหม่ของ checkout** (technology "Not decided") ถ้า DB ล่ม order จะ fail หรือไม่? ADR ไม่บอก | อาจย้ายปัญหา availability จาก MailFast มาที่ DB; ขัด driver 6 (เพิ่ม 2 stateful components: RabbitMQ + DB) สำหรับทีม 4 คน |
| Risk | **อีเมลซ้ำ:** at-least-once + retry ไป MailFast แต่ไม่รู้ว่า MailFast รองรับ idempotency key (Open questions ยอมรับเอง) | fail 0.5 % ที่ timeout แต่จริง ๆ ส่งสำเร็จ จะทำให้ลูกค้าได้อีเมลซ้ำ; ADR dedupe เฉพาะ loyalty |
| Sensitivity point | **ระยะเวลา peak × ส่วนต่าง 150 emails/s:** peak 1 ชม. = backlog ประมาณ 540,000 ฉบับ ระบายที่ 50/s ใช้อีก **ประมาณ 3 ชม.** (และนานกว่านั้นเพราะ order ใหม่ยังเข้ามา) | delay ของอีเมลและขนาด queue/disk ของ RabbitMQ ขึ้นกับตัวเลขนี้โดยตรง; ADR ยังไม่ได้ sizing |
| Sensitivity point | **Rate limit 50/s ต้องเป็น global** ข้ามทุก replica ของ `emailservice` (ADR ไม่ได้บอกว่าจะทำอย่างไร) | ถ้า scale เป็น N pods แต่ละ pod จำกัด 50/s จะชน rate limit ของ MailFast → error/retry storm |
| Sensitivity point | **Outbox relay polling interval + loyalty consumer concurrency/prefetch** | กำหนดว่า lag order → แต้มอยู่ใน 1 นาทีหรือไม่ (QA3) |
| Trade-off point | **Transactional outbox:** เพิ่ม reliability (ไม่หาย event) แต่เพิ่ม latency ของ `PlaceOrder` (DB write ต่อ order), cost และภาระ operations | ADR ยอมรับไว้ใน Consequences แต่ไม่มีตัวเลข latency ของ DB write เทียบกับ budget 200 ms |
| Trade-off point | **Async email:** performance/availability ของ checkout ดีขึ้น แต่ใช้ความทันเวลาของอีเมล (user-perceived latency) แลก | ยอมรับได้ถ้ามี SLA อีเมล ซึ่งยังเป็น open question |
| Non-risk | **แยก queue ต่อ consumer** (email/loyalty/analytics) | backlog อีเมลช่วง 11.11 ไม่ทำให้ loyalty เกิน 1 นาที; ตรงกับ QA3 |
| Non-risk | **เลือก RabbitMQ แทน Kafka** ที่ 200 msg/s | ทีมมี 2 คนเคยใช้ RabbitMQ และเป็น open-source self-hosted (`docs/scenario.md`); per-queue DLQ/throttle เหมาะกับงานนี้ |
| Non-risk | **Dedupe loyalty ด้วย `order_id`** แบบ unique constraint | วิธีมาตรฐานที่ทำให้ได้ผลแบบ exactly once บน at-least-once delivery (ต้องให้ dedupe และการเพิ่มแต้มอยู่ใน transaction เดียวกัน) |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) outbox อย่างเดียวโดยไม่มี broker (consumer poll จาก DB table) ซึ่งตอบ driver 6 ได้ดีกว่าการเพิ่มทั้ง DB และ RabbitMQ; (b) NATS JetStream (open-source, เบากว่า Kafka); (c) รูปแบบ hybrid: loyalty ผ่าน outbox ที่ reliable, อีเมล/analytics ผ่าน best-effort publish
- **Consequences ที่ไม่ได้ระบุ:** schema/versioning ของ `OrderPlaced` เมื่อมี consumer เพิ่มปีหน้า (modifiability); ข้อมูลส่วนบุคคล (email) อยู่ใน message/DLQ; ขนาด disk ของ RabbitMQ สำหรับ backlog
- **Claim ที่อ่อน/น่าจะผิด:**
  - "Option 3 เป็นตัวเลือกเดียวที่ผ่าน driver 1–4" แต่ Decision bullet แรกเอง (publish fail → log) **ไม่ผ่าน** driver 3
  - "retry ครอบคลุม MailFast fail 0.5 %" ไม่จริงทั้งหมดถ้าไม่มี idempotency (เกิดอีเมลซ้ำ)
  - "p95 ใกล้ 300 ms เดิม" เป็น [Inferred] ที่ไม่ได้นับ DB write ของ outbox
  - Consequence ข้อ "ทีมต้องดูแล DB อีกตัว" ปรากฏโดยไม่มี Decision ชัดเจนว่าเลือก DB อะไร
- **Validation ขาด:** ไม่มี test "ปิด RabbitMQ / ปิด order DB ระหว่าง peak" และไม่มีการตรวจว่า event หาย 0 หลัง `chargeCard` สำเร็จ

## 4. Verdict

**Accept with changes.** ทิศทาง (async ผ่าน RabbitMQ, queue แยกต่อ consumer) เหมาะกับระบบนี้ แต่ Decision ยังไม่สอดคล้องกันเองในจุดที่สำคัญที่สุด

1. **ตัดสินใจให้ชัดเรื่อง outbox:** เลือก DB (technology + การ deploy บน K8s), ลบเส้นทาง "publish ตรง + log error" ออก, ระบุว่า DB ล่มแล้ว order ทำอย่างไร และจัดการช่องว่างระหว่าง `chargeCard` กับการเขียน DB
2. **Sizing และ SLA ของอีเมล:** คำนวณ backlog ตามระยะเวลา peak, กำหนด global rate limiter 50/s ข้าม replica, และกำหนดวิธีกันอีเมลซ้ำ (idempotency ของ MailFast หรือ dedupe ฝั่ง `emailservice`)
3. **เพิ่ม Validation สำหรับ failure ของ infrastructure ใหม่:** kill RabbitMQ และ order DB ระหว่าง load 200 orders/s แล้ววัด p95 ≤ 500 ms, order fail 0 %, event หาย 0, loyalty lag ≤ 1 นาที
