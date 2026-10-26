**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**อภิมุข สาโรวาท / รหัสนิสิต: 6631358621**
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: Claude Code
**โมเดล:** Opus 5.5
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A |Claude Code / Opus 5.5 | `prompts/1-…` run A |AGENTS.md, prompts/1-recover-architecture.md, system/online-boutique/protos/demo.proto |อยู่ในขอบเขต |3 | |
| 2 | 1-B | Claude Code / Opus 5.5 | `prompts/1-…` run B | prompts/1-recover-architecture.md, demo.proto, ทุกไฟล์ใน kubernetes-manifests/ (kustomization.yaml, README.md และ yaml ของ 11 service) | อยู่ในขอบเขต | 4 | รันผ่าน subagent (session ใหม่) |
| 3 | 2-a |Claude Code / Opus 5.5 | `prompts/2-…` run a |AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md |อยู่ในขอบเขต |3 | |
| 4 | 2-b |Claude Code / Opus 5.5 | `prompts/2-…` run b |prompts/2-write-adr.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go |อยู่ในขอบเขต |4 | |
| 5 | 3 | Claude Code / Opus 5.5 | `prompts/3-…` adr: docs/adr/0002-post-order-events.md, Thai | AGENTS.md, prompts/3-review-adr.md, docs/adr/0002-post-order-events.md, docs/scenario.md | อยู่ในขอบเขต | 4 | รันผ่าน subagent (session ใหม่) |
| 6 | | | | | | | |
| 7 | | | | | | | |
