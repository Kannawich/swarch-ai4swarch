> Agent: Claude Code (Anthropic CLI), model Claude Opus 5.5 (`claude-opus-5-5`). Run `a` (zero-shot), `context: none`

# ADR-0002: ใช้ event-driven publish/subscribe สำหรับงานหลังสั่งซื้อ (post-order actions)

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ 3 อย่างตาม requirement:

| # | งานหลังสั่งซื้อ | ลักษณะ |
|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** | external provider รายใหม่ มาแทน logic จำลอง (mock) เดิมใน `emailservice` |
| 2 | ให้คะแนนสะสม (loyalty points) ผ่าน **`loyaltyservice`** | service ใหม่ ยังต้องสร้าง |
| 3 | บันทึก analytics event "order placed" ให้ทีม marketing | ปลายทางยังไม่ระบุ |

marketing คาดว่าปีหน้าจะมีงานหลังสั่งซื้อเพิ่ม ได้แก่ SMS, fraud scoring และการอัปเดต recommendations

ปัญหาที่ต้องตัดสินใจ: ระบบที่สั่งซื้อควรส่งต่องานเหล่านี้อย่างไร โดยต้องรองรับ 3 ข้อนี้

- ต้องเพิ่มงานใหม่ได้ง่าย
- ถ้า external provider (MailFast) ขัดข้อง การสั่งซื้อต้องไม่ล้มตาม
- งานแต่ละอย่างต้องไม่กระทบกัน

ข้อจำกัดของ ADR นี้: เขียนจาก requirement อย่างเดียว ไม่ได้อ่านโค้ดหรือ manifest ของระบบ ข้อมูลต่อไปนี้จึงเป็น **Not evidenced in the provided files**:

- service ไหนเป็นผู้สั่งซื้อ
- โปรโตคอลที่ใช้ระหว่าง service
- มี order database หรือไม่
- ปริมาณคำสั่งซื้อต่อวินาที

## Decision drivers

1. **Modifiability / extensibility:** ต้องเพิ่มงานหลังสั่งซื้อใหม่ (SMS, fraud scoring, recommendations) ได้ โดยไม่ต้องแก้ flow การสั่งซื้อทุกครั้ง
2. **Availability ของการสั่งซื้อ:** MailFast, `loyaltyservice` หรือระบบ analytics ล่ม ต้องไม่ทำให้ลูกค้าสั่งซื้อไม่ได้
3. **Performance (latency ของการสั่งซื้อ):** ลูกค้าไม่ควรต้องรองานที่ไม่จำเป็นต่อการยืนยันคำสั่งซื้อ
4. **Reliability ของแต่ละงาน:** อีเมลและคะแนนต้องไม่หาย และไม่ซ้ำ (เช่น ได้คะแนนสองครั้ง)
5. **Cost / operability:** ทีมต้องดูแลโครงสร้างพื้นฐานที่เพิ่มขึ้นได้ (ทักษะและงบประมาณยังไม่ระบุ ดู Open questions)
6. **Security / privacy:** มีการส่งอีเมลและข้อมูลคำสั่งซื้อของลูกค้าออกไปยัง external provider (MailFast)

## Considered options

1. **A: Synchronous orchestration.** service ที่สั่งซื้อเรียก MailFast (ผ่าน `emailservice`), `loyaltyservice` และ analytics ตามลำดับ ภายใน request เดียวกับการสั่งซื้อ
2. **B: Event-driven publish/subscribe ผ่าน message broker.** service ที่สั่งซื้อ publish event `OrderPlaced` ครั้งเดียว ผู้รับแต่ละราย subscribe และทำงานของตัวเองแบบ asynchronous ได้แก่
   - `emailservice` (เปลี่ยนเป็น adapter ที่เรียก MailFast)
   - `loyaltyservice`
   - consumer สำหรับ analytics
3. **C: Dedicated post-order orchestrator service.** service ที่สั่งซื้อส่งคำขอเดียวไปยัง service ใหม่ (เช่น `postorderservice`) ซึ่งเป็นผู้เรียกแต่ละงานต่อเอง (พร้อม retry) แบบ asynchronous เมื่อเทียบกับการสั่งซื้อ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous orchestration | • เรียบง่าย ไม่ต้องเพิ่มโครงสร้างพื้นฐาน<br>• ลูกค้ารู้ผลทันที (เช่น ส่งอีเมลสำเร็จหรือไม่)<br>• debug และ trace ง่าย flow อยู่ที่เดียว | • latency ของการสั่งซื้อเท่ากับผลรวมของทุกงาน<br>• MailFast หรือ `loyaltyservice` ล่มแล้วกระทบการสั่งซื้อ (availability ลดลง)<br>• ทุกงานใหม่ต้องแก้ service ที่สั่งซื้อ (coupling สูง, modifiability ต่ำ)<br>• ถ้าสั่งซื้อสำเร็จแต่บางงานล้ม ต้องเขียน logic ชดเชยเอง |
| B: Event-driven pub/sub | • งานใหม่ (SMS, fraud scoring) เพิ่มได้แค่ subscribe โดยไม่ต้องแก้ผู้ publish (modifiability สูง)<br>• การสั่งซื้อไม่ต้องรอ MailFast หรือ `loyaltyservice` (latency ต่ำ, availability สูง)<br>• แต่ละ consumer scale และ retry ได้อิสระ broker เก็บข้อความไว้ระหว่างที่ consumer ล่ม<br>• analytics ได้ event stream ที่ marketing ใช้ต่อได้ | • ต้องติดตั้งและดูแล message broker (cost, operability)<br>• เป็น eventual consistency ลูกค้าอาจเห็นคะแนนช้ากว่าการสั่งซื้อ<br>• ส่งแบบ at-least-once จึงต้องทำ idempotency กันอีเมลหรือคะแนนซ้ำ<br>• trace ข้าม service ยากขึ้น ต้องมี observability เพิ่ม<br>• dual-write: บันทึกคำสั่งซื้อสำเร็จแต่ publish ไม่สำเร็จ อาจต้องใช้ outbox pattern |
| C: Post-order orchestrator | • ผู้สั่งซื้อรู้จักแค่ service เดียว (coupling ต่ำกว่า A)<br>• flow ของงานหลังสั่งซื้อรวมอยู่ที่เดียว อ่านง่าย<br>• กำหนดลำดับหรือเงื่อนไขระหว่างงานได้ (เช่น fraud scoring ก่อนให้คะแนน) | • ทุกงานใหม่ยังต้องแก้ orchestrator (modifiability ปานกลาง)<br>• orchestrator เป็น single point of failure ใหม่ ถ้าไม่มี queue หรือ storage ที่ทนทาน งานจะหายเมื่อ orchestrator ล่ม<br>• ต้องสร้างและดูแล service เพิ่ม และอาจต้องมี state หรือ queue ของตัวเองอยู่ดี |

## Decision

เลือก **Option B: Event-driven publish/subscribe ผ่าน message broker**

- service ที่สั่งซื้อจะ publish event `OrderPlaced` ครั้งเดียวหลังบันทึกคำสั่งซื้อสำเร็จ event มีข้อมูลดังนี้: order id, user id, email, รายการสินค้า, ยอดเงิน และ timestamp
- consumer เริ่มต้นมี 3 ราย และแต่ละรายมี subscription หรือ queue ของตัวเอง:
  - `emailservice`: เปลี่ยนจาก logic จำลองเป็น adapter ที่เรียก MailFast
  - `loyaltyservice`: ให้คะแนนสะสม
  - consumer สำหรับ analytics: ส่ง event "order placed" ไปยังระบบของ marketing
- consumer ทุกตัวต้องเป็น **idempotent** โดยใช้ order id เป็น key กันการทำงานซ้ำ
- ข้อความที่ประมวลผลไม่สำเร็จหลัง retry จะไปอยู่ใน **dead-letter queue**
- ยังไม่เลือกเทคโนโลยีของ broker (เช่น RabbitMQ, Kafka หรือ managed cloud pub/sub) จนกว่าจะได้คำตอบเรื่อง volume, ทักษะทีม และงบประมาณ (ดู Open questions)

เหตุผล: driver อันดับ 1 คือ modifiability เพราะมีงานใหม่อย่างน้อย 3 อย่างรออยู่ในปีหน้า B ตอบโจทย์ข้อนี้ดีที่สุด เพิ่มงานได้โดยไม่แตะ flow การสั่งซื้อ B ยังแยก availability ของการสั่งซื้อออกจาก MailFast ซึ่งเป็น external dependency ที่ทีมควบคุมไม่ได้ (driver 2–3) ส่วนต้นทุนที่เพิ่มขึ้น (broker, idempotency, observability) จ่ายครั้งเดียวแล้วใช้ซ้ำกับงานใหม่ทุกงาน ส่วน A ทำให้การสั่งซื้อเปราะบางขึ้นทุกครั้งที่เพิ่มงาน และ C ยังต้องแก้ศูนย์กลางทุกครั้งที่เพิ่มงาน

## Consequences

- **Positive:**
  - เพิ่ม SMS, fraud scoring และ recommendations update ได้ด้วยการเพิ่ม consumer ใหม่ ไม่ต้องแก้ผู้ publish (quality attribute: modifiability)
  - MailFast, `loyaltyservice` หรือ analytics ล่ม ไม่ทำให้การสั่งซื้อล้ม broker เก็บ event ไว้ส่งภายหลัง (quality attribute: availability)
  - latency ของการสั่งซื้อไม่ขึ้นกับเวลาตอบของ MailFast หรือ `loyaltyservice` (quality attribute: performance)
  - consumer แต่ละตัว scale แยกกันตามโหลดของตัวเองได้ (quality attribute: scalability)
  - การเปลี่ยน email provider ในอนาคตกระทบแค่ adapter ใน `emailservice` (quality attribute: modifiability)
- **Negative:**
  - ต้องติดตั้ง ดูแล monitor และ backup message broker เพิ่ม (quality attribute: cost, operability)
  - ผลของงานหลังสั่งซื้อเป็น eventual consistency เช่น ลูกค้าอาจยังไม่เห็นคะแนนทันที (quality attribute: usability, consistency)
  - debug flow ข้าม service ยากขึ้น ต้องมี correlation id และ distributed tracing (quality attribute: testability, observability)
  - schema ของ event `OrderPlaced` กลายเป็น contract ที่ consumer หลายรายพึ่งพา จะเปลี่ยนต้องมี versioning (quality attribute: modifiability)
- **Risks and mitigations:**
  - *อีเมลหรือคะแนนซ้ำจาก at-least-once delivery:* consumer ใช้ order id เป็น idempotency key
  - *event หายเพราะ dual-write (บันทึกคำสั่งซื้อแล้วแต่ publish ไม่สำเร็จ):* ใช้ transactional outbox หรือ publish พร้อม retry แต่ต้องรู้ก่อนว่ามี order store หรือไม่ (ดู Open questions)
  - *MailFast ขัดข้องนาน หรือโดน rate limit:* retry แบบ exponential backoff, dead-letter queue และ alert เมื่อ queue ค้างเกินเกณฑ์
  - *ข้อมูลส่วนบุคคล (อีเมล ที่อยู่) รั่วไปยัง MailFast หรือ analytics:* ส่งเฉพาะ field ที่จำเป็น เข้ารหัสระหว่างทาง และตรวจ data processing agreement กับ MailFast
  - *broker เป็น single point of failure:* ใช้ broker แบบ replicated หรือ managed service และวัด availability ของ broker แยก

## Validation

- **Availability scenario:** ปิด MailFast (หรือ stub ให้ตอบ error) และปิด `loyaltyservice` แล้วรัน load test อัตราการสั่งซื้อสำเร็จต้องไม่ลดลง เมื่อเปิดกลับมา อีเมลและคะแนนต้องถูกส่งครบ
- **Performance:** วัด latency ของการสั่งซื้อ (p95) ก่อนและหลังเปลี่ยน ค่าเป้าหมายยังไม่มีตัวเลข (ดู Open questions)
- **End-to-end lag:** วัดเวลาจากสั่งซื้อจนถึงส่งอีเมลสำเร็จ (p95) และขนาดของ dead-letter queue
- **Correctness:** ส่ง event ซ้ำโดยตั้งใจ แล้วตรวจว่าไม่มีอีเมลหรือคะแนนซ้ำ (duplicate rate = 0)
- **Modifiability scenario:** เพิ่ม consumer จำลองสำหรับ SMS โดยไม่แก้โค้ดของผู้ publish นับจำนวนไฟล์และ service ที่ต้องแก้ เป้าหมายคือไม่แก้ผู้ publish เลย
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มงาน SMS หรือ fraud scoring ปีหน้า (ยังไม่มีวันที่แน่นอน)

## Open questions

ข้อมูลเหล่านี้ไม่มีใน requirement จึงไม่ได้สมมติค่าขึ้นเอง:

1. **ผู้สั่งซื้อและ data store:** service ไหนเป็นผู้สั่งซื้อ และมี order database ให้ทำ outbox หรือไม่ ใช้โปรโตคอลอะไรระหว่าง service
2. **Volume:** ปริมาณคำสั่งซื้อปกติและช่วง peak (orders/วินาที) ใช้เลือก broker และ sizing
3. **Latency targets:** เป้า latency ของการสั่งซื้อ และอีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาทีหรือนาที
4. **MailFast:** API แบบไหน (REST, SMTP), rate limit, SLA, ค่าใช้จ่ายต่ออีเมล, มี idempotency key หรือ webhook แจ้งสถานะหรือไม่
5. **Loyalty rules:** สูตรคำนวณคะแนน, ต้องหักคะแนนคืนเมื่อยกเลิกหรือคืนสินค้าหรือไม่, ลูกค้าต้องเห็นคะแนนทันทีหรือไม่
6. **Analytics:** ปลายทางของ event คืออะไร (data warehouse หรือ analytics tool) และ schema ที่ marketing ต้องการ
7. **Fraud scoring ปีหน้า:** ต้อง *ขวาง* คำสั่งซื้อก่อนยืนยันหรือไม่ ถ้าใช่จะเป็นงาน synchronous ที่ pub/sub ไม่ครอบคลุม ต้องมี ADR แยก
8. **ทีมและงบ:** ทีมมีประสบการณ์กับ message broker ตัวไหน งบสำหรับ infrastructure หรือ managed service เท่าไร
9. **Compliance:** ข้อกำหนดด้าน privacy (เช่น PDPA หรือ GDPR) ในการส่งอีเมลและข้อมูลลูกค้าให้ MailFast และ analytics
10. **Deciders:** ใครเป็นผู้อนุมัติ ADR นี้
