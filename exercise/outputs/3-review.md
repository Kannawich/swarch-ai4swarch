> Agent: Claude Code (Anthropic CLI), model Claude Opus 5.5 (`claude-opus-5-5`). รีวิว `docs/adr/0002-final_adr.md` โดยใช้ `docs/scenario.md` เป็นข้อมูลอ้างอิง

# ATAM review: ADR-0002 (RabbitMQ `OrderPlaced`)

ข้ออ้างที่ ADR ยกมาจาก `main.go` (เลขบรรทัด, ไม่มี timeout, ไม่มี DB) ตรวจสอบไม่ได้ เพราะ task นี้อนุญาตให้อ่านแค่ ADR และ scenario จึงถือว่าเป็นข้อมูลที่ ADR อ้างไว้เท่านั้น

## 1. Quality-attribute scenarios

| # | Source · Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| S1 Performance | ลูกค้า · `PlaceOrder` 200 ครั้ง/วินาที | sale 11.11, MailFast p95 1.5 s / p99 8 s | ยืนยัน order โดยไม่รองานหลังสั่งซื้อ | p95 ของ `PlaceOrder` ≤ 500 ms (publish ใช้ได้ไม่เกิน headroom ประมาณ 200 ms) |
| S2 Availability + correctness | ความล้มเหลว · `loyaltyservice` หรือ RabbitMQ node ล่มระหว่างสั่งซื้อ | peak | order สำเร็จ, event ไม่หาย, ให้คะแนนหลัง recover | order สำเร็จ 100 %, คะแนนภายใน 60 s, คะแนนต่อ `order_id` = 1 (ไม่ใช่ 0 และไม่ใช่ 2) |
| S3 Rate limit | checkout · อีเมล 200 ฉบับ/วินาที เทียบกับ limit 50/s และล้ม 0.5 % | peak | เก็บ buffer, throttle, retry | rate-limit error จาก MailFast = 0, อีเมลหาย = 0, ระบายที่ 50/s (backlog โต 150/s) |

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | ADR ขัดกันเองเรื่อง outbox: Decision (บรรทัด 81) ให้เขียน outbox table เมื่อ publish ล้ม แต่ Context (23) และ Risks (112) บอกว่า `checkoutservice` ไม่มี DB และ Open question 3 ยังไม่ได้ตัดสินใจ | ถ้าไม่มี outbox แล้ว broker ล่ม หรือ pod ตายหลัง `chargeCard` แต่ก่อน publish จะได้ loyalty = **0 ครั้ง** ซึ่งผิด S2 (exactly once) และอีเมลหายด้วย. quorum queues ช่วยไม่ได้ เพราะ event ยังไปไม่ถึง broker |
| Risk | email consumer ถูกกำหนดเป็น **1 replica** (บรรทัด 83) แต่ประโยคเดียวกันเขียนว่า "รวมทุก replica" | เป็น single point of failure: ระหว่าง restart หรือ deploy อีเมลหยุดส่ง ทั้งที่ backlog กำลังโต 150/s |
| Sensitivity point | concurrency หรือ prefetch ของ email consumer: ถ้าจะส่ง 50/s เมื่อ MailFast p95 = 1.5 s ต้องมี in-flight ประมาณ 50 × 1.5 = 75 ข้อความ และช่วง p99 (8 s) ต้องมีถึงประมาณ 400 **[Inferred]** (Little's law) | ADR ไม่ระบุค่านี้ ถ้า prefetch ต่ำ throughput จริงจะต่ำกว่า 50/s มาก และคิวจะไม่มีวันระบายหมด |
| Sensitivity point | publish timeout ภายใน headroom 200 ms (Open question 4) | ตั้งสูงไปจะผิด S1 ตั้งต่ำไปจะได้ confirm ไม่ทันและ fallback บ่อย ซึ่งตอนนี้ fallback ยังไม่มีอยู่จริง (ดู Risk แรก) |
| Sensitivity point | ระยะเวลา peak (Open question 1) กำหนดขนาด backlog และ disk ของ RabbitMQ: 150 ข้อความ/วินาทีต่อทุกนาทีของ peak | ถ้า disk เต็ม RabbitMQ จะปิดรับ publish (flow control) ทุก order จะตกไปที่ fallback พร้อมกันหมด **[Inferred]** |
| Trade-off point | outbox store ใน `checkoutservice` | reliability (ไม่หาย) ↑ เทียบกับ cost/operability (stateful store ใหม่) ↓ และ latency ของ `PlaceOrder` ↑ |
| Trade-off point | RabbitMQ queue แทน log (Kafka) | modifiability และ team fit ↑ แต่ analytics และ consumer ใหม่ปีหน้า **replay ย้อนหลังไม่ได้** ↓ |
| Non-risk | queue แยกต่อ consumer และ `loyaltyservice` dedup ด้วย unique constraint บน `order_id` ใน transaction เดียวกับการเพิ่มคะแนน | ได้ effectively-once จาก at-least-once และอีเมลที่ค้างไม่ทำให้ loyalty ช้าจนเกิน 1 นาที |
| Non-risk | ตัดตัวเลือก goroutine และ sync gRPC ออก โดยอ้างตัวเลขจาก scenario (p95 1.5 s, 50/s เทียบกับ 200/s) | เหตุผลชัดและตรวจสอบได้ |
| Non-risk | แผน Validation มีตัวชี้วัดที่วัดได้ (diff = 0 บรรทัด, คะแนน = 1, ภายใน 60 s, ไม่มี 429) | ตรงกับ S1–S3 |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** RabbitMQ Streams สำหรับ analytics (ใช้ broker ตัวเดิมแต่ replay ได้) และ shared rate limiter ข้าม replica (token bucket กลาง) แทนการบังคับให้มี 1 replica
- **Consequences ที่ไม่ได้ระบุ:**
  - ไม่มีเป้าหมายเวลาส่งอีเมล ตัวอย่างในบรรทัด 108 ที่ระบาย 30 นาทีหลัง peak ยังไม่มีเกณฑ์ว่ารับได้หรือไม่
  - loyalty consumer ต้องรับได้ ≥ 200 msg/s และมีมากกว่า 1 replica แต่ไม่ได้ระบุ
  - fanout หรือ topic ยังไม่ได้เลือก
- **Claims ที่ไม่มีหลักฐานหรือน่าจะผิด:**
  - "p95 ประมาณ 1.8 s" ได้จากการเอา p95 สองค่ามาบวกกัน ซึ่งไม่ใช่วิธีรวม percentile ที่ถูกต้อง แต่ข้อสรุปว่าผิดเป้าก็ยังจริง
  - "ได้ผล exactly once" เป็นจริงแค่ฝั่ง duplicate ส่วนฝั่ง event หายยังไม่ได้แก้ (Risk แรก)
  - mitigation "log `order_id` เพื่อ reconcile" ไม่มีกระบวนการ reconcile และไม่มีเจ้าของ

## 4. Verdict: **Accept with changes**

1. **ตัดสินใจเรื่อง outbox** (Open question 3) แล้วแก้บรรทัด 81, 112 และ Risks ให้ตรงกัน ถ้าไม่ทำ outbox ต้องเขียนให้ชัดว่ายอมรับ loyalty = 0 ได้ในกรณีใด
2. **ออกแบบ email consumer ใหม่:** กำหนด prefetch หรือ concurrency (≥ 75 in-flight) และใช้ shared limiter ให้รันได้ ≥ 2 replica แทน 1 replica ที่หยุดส่งทุกครั้งที่ restart
3. **กำหนดตัวเลขที่ขาด:** publish timeout, peak duration (เพื่อคำนวณ disk), เป้าเวลาส่งอีเมล และจำนวน replica หรือ throughput ของ loyalty consumer แล้วใส่ทั้งหมดลงใน load test
