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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ เพราะไม่ได้อ่านไฟล์ scenario|ใช้ครบ ทั้ง p95 <= 500 ms, 200 orders/s, MailFast p95 1.5 s / p99 8 s / ล้ม 0.5 % / 50 emails/s|
| ทางเลือกที่พิจารณา (ระบุ) |3 ทาง: A sync orchestration, B event-driven pub/sub, C post-order orchestrator เลือก B|4 ทาง: sync gRPC, goroutines ใน process, RabbitMQ, Kafka เลือก RabbitMQ|
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร |มีการพูดถึง และมีการยกเรื่อง event หาย มีการพูดถึง retry/backoff แต่ไม่ได้พูดถึงตัวเลข 50/s| ถ้า publish ล้มแล้ว event หาย ซึ่ง**ขัดกับ** exactly once ที่อ้างไว้ (บรรทัด 81 กับ 90)<br>rate limit: ตั้งเป้า ≤ 50/s รวมทุก replica (บรรทัด 83) แต่ไม่บอกกลไก|
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี ไม่มีตัวเลขแต่งเอง แต่**สมมติว่ามีการ "บันทึกคำสั่งซื้อ"** ทั้งที่โค้ดจริงไม่มี| มี: "ไม่มีใครในทีมเคยใช้ Kafka" (บรรทัด 63) ไม่มีใน scenario, prefetch คุม rate limit ได้|

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | Decision: เปลี่ยน "จำกัดอัตรา ≤ 50/s รวมทุก replica" ให้ระบุกลไกชัดเจน (ใช้ email consumer 1 replica พร้อม token bucket ≤ 50 emails/s) ใช้หน่วย emails/s ให้สม่ำเสมอ | AI ไม่บอกว่าหลาย pod จะรู้ยอดรวมกันได้อย่างไร ถ้ามี 3 pod ที่แต่ละตัวจำกัด 50/s รวมกันจะส่งได้ 150 emails/s เกิน rate limit ของ MailFast |
| 2 | Decision: เปลี่ยน "ถ้า publish ไม่สำเร็จให้ log แล้วคืนผลสำเร็จ" เป็นการบันทึก `OrderPlaced` ลง outbox table ก่อนคืนผล แล้วให้ relay retry publish ไป RabbitMQ ถ้าเขียน outbox ไม่สำเร็จให้คืน error | ร่างของ AI อ้างว่าผ่าน driver 3 (loyalty exactly once) แต่ถ้า publish ล้ม event จะหาย ลูกค้าไม่ได้คะแนน |
| 3 | Context: แก้ลำดับการเรียกใน `PlaceOrder` ให้ครบ โดยเพิ่มการดึงตะกร้า ราคาสินค้า แปลงสกุลเงิน และขอค่าส่ง ก่อน `chargeCard` | หัวข้อเขียนว่า "จากโค้ด" แต่ลำดับที่เขียนตรงกับ scenario ที่ย่อไว้ ในโค้ดจริงมีการเรียกก่อนหน้านั้นอีก |
