**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

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

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → paymentservice | checkoutservice.yaml (PAYMENT_SERVICE_ADDR) | ✅ | พบใน checkoutservice.yaml บรรทัด 61–62 และ main.go บรรทัด 116 อ่านค่านี้จริง |
| frontend → adservice | frontend.yaml (AD_SERVICE_ADDR) | ✅ | พบใน frontend.yaml บรรทัด 81–82 และ frontend/main.go บรรทัด 138 อ่านค่านี้จริง |
| recommendationservice → productcatalogservice | recommendationservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR) | ✅ | พบใน recommendationservice.yaml บรรทัด 60–61 และ recommendation_server.py บรรทัด 131 อ่านค่านี้ (service นี้เขียนด้วย Python) |
| cartservice → redis-cart | cartservice.yaml (REDIS_ADDR) | ✅ | พบใน cartservice.yaml บรรทัด 50–51 และ src/cartservice/src/Startup.cs บรรทัด 29 อ่านค่านี้ (service นี้เขียนด้วย C#) |
| loadgenerator → frontend | loadgenerator.yaml (FRONTEND_ADDR) | ❓ | มี FRONTEND_ADDR ใน loadgenerator.yaml บรรทัด 69, 82 และใช้เรียก frontend ที่บรรทัด 48–49 แต่ kustomization.yaml บรรทัด 24 comment ไว้ จึงไม่ได้ deploy ตามปกติ เป็นเครื่องมือทดสอบ ไม่ควรอยู่ในแผนภาพเหมือนส่วนหนึ่งของระบบ |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): (1) dependency ที่ขาด: frontend → shoppingassistantservice (frontend.yaml บรรทัด 83–84) ไม่อยู่ในตารางและแผนภาพ AI พูดถึงแค่ใน Gaps (2) container ที่น่าสงสัย: loadgenerator ถูกวาดเป็นส่วนของระบบ แต่ kustomization.yaml บรรทัด 24 comment ไว้ จึงไม่ได้ deploy ตามปกติ


## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
Run A อ่านแค่ demo.proto จึงพบ dependency 0 ตัว ส่วน Run B ที่ได้อ่าน manifest ด้วยพบ 17 ตัว
เพราะไฟล์ .proto บอกแค่ว่าแต่ละ service มีรายละเอียด function ยังไง แต่ไม่ได้บอกว่า service ไหนเรียกอันไหน
นอกจากนี้ Run A ยังไม่รู้จัก data store(redis-cart) และ frontend/web container เลย เพราะทั้งสองไม่ได้มีใน .proto file

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
ข้อผิดพลาดที่อันตรายที่สุดคือ loadgenerator
เพราะถ้าสมาชิกใหม่เชื่อแผนภาพนี้ ตอน deploy ขึ้นของจริง เขาเปิด loadgenerator ด้วย ทำให้มี traffic ปลอมยิงเข้า frontend ตลอดเวลา
แต่ความจริง kustomization.yaml บรรทัด 24 comment loadgenerator ไว้ จึงไม่ได้ถูก deploy เป็นส่วนของระบบ


