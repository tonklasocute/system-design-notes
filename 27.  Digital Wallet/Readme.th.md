# บทที่ 27: กระเป๋าเงินดิจิทัล (Digital Wallet)

## บทนำ
**แพลตฟอร์มการชำระเงิน (payment platform)** มักจะมี **บริการกระเป๋าเงิน (wallet service)** ซึ่งอนุญาตให้ลูกค้าสามารถเก็บเงินไว้ในแอปพลิเคชัน แล้วสามารถถอนออกได้ในภายหลัง

คุณยังสามารถใช้มันเพื่อจ่ายค่าสินค้าและบริการ หรือโอนเงินให้ผู้ใช้คนอื่นที่ใช้บริการ **กระเป๋าเงินดิจิทัล (digital wallet)** ได้ ซึ่งอาจเร็วกว่าและถูกกว่าการทำผ่านช่องทางการชำระเงินแบบปกติ (normal payment rails)

<div style="margin-left:3rem">
    <img src="./images/digital-wallet.png" alt="digital-wallet" width="500" />
</div>

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)
 * C: เราควรมุ่งเน้นแค่การโอนเงินระหว่างกระเป๋าเงินดิจิทัลเท่านั้นหรือไม่? เราควรรองรับ operation อื่น ๆ หรือไม่?
 * I: ให้เรามุ่งเน้นที่การโอนเงินระหว่างกระเป๋าเงินดิจิทัลก่อนในตอนนี้
 * C: ระบบต้องรองรับกี่ธุรกรรมต่อวินาที?
 * I: สมมติว่า 1 ล้าน TPS
 * C: กระเป๋าเงินดิจิทัลมีความต้องการด้านความถูกต้อง (correctness) ที่เข้มงวด เราสามารถสมมติได้หรือไม่ว่าการรับประกันแบบ transactional เพียงพอแล้ว?
 * I: ฟังดูดี
 * C: เราจำเป็นต้องพิสูจน์ความถูกต้อง (correctness) หรือไม่?
 * I: เราสามารถทำได้ผ่านการกระทบยอด (reconciliation) แต่นั่นตรวจจับได้แค่ความคลาดเคลื่อน ไม่ได้บอกสาเหตุที่แท้จริง (root cause) แทนที่จะทำอย่างนั้น เราต้องการความสามารถในการ replay ข้อมูลจากจุดเริ่มต้นเพื่อสร้างประวัติขึ้นใหม่
 * C: เราสามารถสมมติได้หรือไม่ว่าความต้องการด้านความพร้อมใช้งาน (availability) อยู่ที่ 99.99%?
 * I: ได้
 * C: เราจำเป็นต้องพิจารณาการแลกเปลี่ยนเงินตราต่างประเทศ (foreign exchange) หรือไม่?
 * I: ไม่ อยู่นอกขอบเขต

สรุปสิ่งที่เราต้องรองรับ:
 * รองรับการโอนยอดคงเหลือระหว่างสองบัญชี
 * รองรับ 1 ล้าน TPS
 * ความน่าเชื่อถือ (reliability) 99.99%
 * รองรับ transaction
 * รองรับการทำซ้ำได้ (reproducibility)

### **การประมาณการแบบคร่าว ๆ (Back-of-the-envelope estimation)**
ฐานข้อมูลเชิงสัมพันธ์แบบดั้งเดิม ที่ provision บน cloud สามารถรองรับได้ประมาณ 1,000 TPS

เพื่อให้ได้ 1 ล้าน TPS เราจะต้องใช้ node ฐานข้อมูล 1,000 ตัว แต่หากการโอนเงินแต่ละครั้งมีสองขา (leg) เราก็จะต้องรองรับจริง ๆ ที่ 2 ล้าน TPS

เป้าหมายในการออกแบบข้อหนึ่งของเราคือการเพิ่ม TPS ที่ node เดียวสามารถรองรับได้ เพื่อให้เราสามารถใช้ node ฐานข้อมูลน้อยลง

| TPS ต่อ node | จำนวน node |
|--------------|-------------|
| 100          | 20,000      |
| 1,000        | 2,000       |
| 10,000       | 200         |

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

### **การออกแบบ API (API Design)**
เราจำเป็นต้องมีเพียงหนึ่ง endpoint สำหรับการสัมภาษณ์นี้:
```
POST /v1/wallet/balance_transfer - โอนยอดคงเหลือจากกระเป๋าเงินหนึ่งไปยังอีกกระเป๋าเงินหนึ่ง
```

พารามิเตอร์ของ request - from_account, to_account, amount (เป็น string เพื่อไม่ให้สูญเสียความแม่นยำ), currency, transaction_id (idempotency key)

ตัวอย่าง response:
```
{
    "status": "success"
    "transaction_id": "01589980-2664-11ec-9621-0242ac130002"
}
```

### **โซลูชันการ shard แบบ in-memory (In-memory sharding solution)**
แอปพลิเคชันกระเป๋าเงินของเราดูแลรักษายอดคงเหลือในบัญชีของผู้ใช้ทุกคน

โครงสร้างข้อมูลที่ดีสำหรับแทนสิ่งนี้คือ `map<user_id, balance>` ซึ่งสามารถ implement โดยใช้ in-memory Redis store ได้

เนื่องจาก redis node เดียวไม่สามารถทนต่อ 1 ล้าน TPS ได้ เราจำเป็นต้อง partition redis cluster ของเราออกเป็นหลาย node

ตัวอย่าง algorithm การ partition:
```
String accountID = "A";
Int partitionNumber = 7;
Int myPartition = accountID.hashCode() % partitionNumber;
```

Zookeeper สามารถใช้จัดเก็บจำนวน partition และที่อยู่ของ redis node ต่าง ๆ ได้ เนื่องจากมันเป็น configuration storage ที่มีความพร้อมใช้งานสูง (highly-available)

สุดท้าย wallet service เป็นบริการแบบ stateless ที่รับผิดชอบในการดำเนินการโอนยอดคงเหลือ มันสามารถขยาย (scale) แนวนอนได้อย่างง่ายดาย:

<div style="margin-left:3rem">
    <img src="./images/wallet-service.png" alt="wallet-service" width="500" />
</div>

แม้ว่าโซลูชันนี้จะแก้ปัญหาด้านความสามารถในการขยาย (scalability) ได้ แต่มันไม่อนุญาตให้เราดำเนินการโอนยอดคงเหลือแบบ atomic ได้

### **Transaction แบบกระจาย (Distributed transactions)**
วิธีหนึ่งในการจัดการ transaction คือการใช้ two-phase commit protocol บนพื้นฐานของฐานข้อมูลเชิงสัมพันธ์แบบ sharded ทั่วไป:

<div style="margin-left:3rem">
    <img src="./images/distributed-transactions-relational-dbs.png" alt="distributed-transactions-relational-dbs" width="500" />
</div>

นี่คือวิธีที่ two-phase commit (2PC) protocol ทำงาน:

<div style="margin-left:3rem">
    <img src="./images/2pc-protocol.png" alt="2pc-protocol" width="500" />
</div>

 * coordinator (wallet service) ทำ operation อ่านและเขียนบนฐานข้อมูลหลายตัวตามปกติ
 * เมื่อแอปพลิเคชันพร้อม commit transaction แล้ว coordinator จะขอให้ฐานข้อมูลทุกตัว prepare
 * หากฐานข้อมูลทุกตัวตอบกลับว่า "yes" coordinator จะขอให้ฐานข้อมูล commit transaction
 * มิฉะนั้น ฐานข้อมูลทุกตัวจะถูกขอให้ยกเลิก (abort) transaction

ข้อเสียของแนวทาง 2PC:
 * ไม่มีประสิทธิภาพเนื่องจาก lock contention
 * coordinator เป็นจุดที่อาจเกิดความล้มเหลวได้ทั้งระบบ (single point of failure)

### **Transaction แบบกระจายด้วย Try-Confirm/Cancel (TC/C)**
TC/C เป็นรูปแบบหนึ่งของ 2PC protocol ซึ่งทำงานร่วมกับ transaction ชดเชย (compensating transaction):
 * coordinator ขอให้ฐานข้อมูลทุกตัวจอง (reserve) ทรัพยากรสำหรับ transaction
 * coordinator รวบรวมคำตอบจากฐานข้อมูล - หากตอบว่า yes ฐานข้อมูลจะถูกขอให้ try-confirm หากตอบว่า no ฐานข้อมูลจะถูกขอให้ try-cancel

ความแตกต่างที่สำคัญข้อหนึ่งระหว่าง TC/C และ 2PC คือ 2PC ดำเนินการ transaction เดียว ในขณะที่ TC/C มี transaction อิสระสองรายการ

นี่คือวิธีที่ TC/C ทำงานในแต่ละ phase:

| Phase | Operation | A                   | C                   |
|-------|-----------|---------------------|---------------------|
| 1     | Try       | เปลี่ยนยอดคงเหลือ: -$1 | ไม่ทำอะไร          |
| 2     | Confirm   | ไม่ทำอะไร          | เปลี่ยนยอดคงเหลือ: +$1 |
|       | Cancel    | เปลี่ยนยอดคงเหลือ: +$1 | ไม่ทำอะไร          |

Phase 1 - try:

<div style="margin-left:3rem">
    <img src="./images/try-phase.png" alt="try-phase" width="500" />
</div>

 * coordinator เริ่ม local transaction ในฐานข้อมูลของ A เพื่อลดยอดคงเหลือของ A ลง $1
 * ฐานข้อมูลของ C ได้รับคำสั่ง NOP ซึ่งไม่ทำอะไรเลย

Phase 2a - confirm:

<div style="margin-left:3rem">
    <img src="./images/confirm-phase.png" alt="confirm-phase" width="500" />
</div>

 * หากฐานข้อมูลทั้งสองตอบกลับว่า "yes" phase confirm จะเริ่มขึ้น
 * ฐานข้อมูลของ A ได้รับ NOP ในขณะที่ฐานข้อมูลของ C ได้รับคำสั่งให้เพิ่มยอดคงเหลือของ C ขึ้น $1 (local transaction)

Phase 2b - cancel:

<div style="margin-left:3rem">
    <img src="./images/cancel-phase.png" alt="cancel-phase" width="500" />
</div>

 * หากมี operation ใดใน phase 1 ล้มเหลว phase cancel จะเริ่มขึ้น
 * ฐานข้อมูลของ A ได้รับคำสั่งให้เพิ่มยอดคงเหลือของ A ขึ้น $1, ฐานข้อมูลของ C ได้รับ NOP

นี่คือการเปรียบเทียบระหว่าง 2PC และ TC/C:

|      | Phase แรก                                            | Phase ที่สอง: สำเร็จ              | Phase ที่สอง: ล้มเหลว                        |
|------|--------------------------------------------------------|------------------------------------|-------------------------------------------|
| 2PC  | transaction ยังไม่เสร็จสมบูรณ์                          | Commit/Cancel transaction ทั้งหมด     | ยกเลิก transaction ทั้งหมด                   |
| TC/C | transaction ทั้งหมดเสร็จสมบูรณ์แล้ว - ไม่ commit ก็ cancel | ดำเนิน transaction ใหม่หากจำเป็น | ย้อนกลับ (reverse) transaction ที่ commit ไปแล้ว |

TC/C ยังถูกเรียกว่า distributed transaction by compensation การดำเนินการระดับสูงถูกจัดการอยู่ใน business logic

คุณสมบัติอื่น ๆ ของ TC/C:
 * ไม่ยึดติดกับฐานข้อมูลใดฐานข้อมูลหนึ่ง (database agnostic) ตราบใดที่ฐานข้อมูลรองรับ transaction
 * รายละเอียดและความซับซ้อนของ distributed transaction ต้องถูกจัดการใน business logic

### **โหมดความล้มเหลวของ TC/C (TC/C Failure modes)**
หาก coordinator ตายกลางคัน มันจำเป็นต้องกู้คืนสถานะระหว่างกลาง (intermediary state)
สิ่งนี้สามารถทำได้โดยการดูแลรักษาตาราง phase status ซึ่งถูกอัปเดตแบบ atomic ภายใน database shard:

<div style="margin-left:3rem">
    <img src="./images/phase-status-tables.png" alt="phase-status-tables" width="500" />
</div>

ตารางนี้มีอะไรบ้าง:
 * ID และเนื้อหาของ distributed transaction
 * สถานะของ try phase - ยังไม่ส่ง, ส่งแล้ว, ได้รับ response แล้ว
 * ชื่อของ phase ที่สอง - confirm หรือ cancel
 * สถานะของ phase ที่สอง
 * flag out-of-order (จะอธิบายในภายหลัง)

ข้อควรระวังหนึ่งเมื่อใช้ TC/C คือมีช่วงเวลาสั้น ๆ ที่สถานะของบัญชีต่าง ๆ ไม่สอดคล้องกัน ในขณะที่ distributed transaction กำลังดำเนินอยู่:

<div style="margin-left:3rem">
    <img src="./images/unbalanced-state.png" alt="unbalanced-state" width="500" />
</div>

สิ่งนี้ไม่เป็นปัญหาตราบใดที่เราสามารถกู้คืนจากสถานะนี้ได้เสมอ และผู้ใช้ไม่สามารถใช้สถานะระหว่างกลางนี้เพื่อใช้จ่ายเงินได้
สิ่งนี้ถูกรับประกันด้วยการดำเนินการหักเงิน (deduction) ก่อนการเพิ่มเงิน (addition) เสมอ

| ตัวเลือกใน Try phase  | บัญชี A | บัญชี C |
|--------------------|-----------|-----------|
| ตัวเลือก 1           | -$1       | NOP       |
| ตัวเลือก 2 (ไม่ถูกต้อง) | NOP       | +$1       |
| ตัวเลือก 3 (ไม่ถูกต้อง) | -$1       | +$1       |

โปรดสังเกตว่าตัวเลือก 3 จากตารางข้างต้นไม่ถูกต้อง เนื่องจากเราไม่สามารถรับประกันการดำเนินการ transaction แบบ atomic ข้ามฐานข้อมูลที่แตกต่างกันได้ โดยไม่พึ่งพา 2PC

กรณีขอบ (edge-case) หนึ่งที่ต้องจัดการคือการดำเนินการแบบไม่เรียงลำดับ (out of order execution):

<div style="margin-left:3rem">
    <img src="./images/out-of-order-execution.png" alt="out-of-order-execution" width="500" />
</div>

เป็นไปได้ที่ฐานข้อมูลจะได้รับ operation cancel ก่อนที่จะได้รับ try กรณีขอบนี้สามารถจัดการได้ด้วยการเพิ่ม flag out-of-order ในตาราง phase status ของเรา
เมื่อเราได้รับ operation try เราจะตรวจสอบก่อนว่า flag out-of-order ถูกตั้งค่าไว้หรือไม่ หากใช่ จะส่งคืนความล้มเหลว

### **Transaction แบบกระจายด้วย Saga (Distributed transaction using Saga)**
อีกแนวทางที่นิยมใช้คือการใช้ Saga - มาตรฐานสำหรับการ implement distributed transaction ในสถาปัตยกรรมแบบ microservice

นี่คือวิธีที่มันทำงาน:
 * operation ทั้งหมดถูกเรียงลำดับเป็นลำดับ (sequence) โดย operation ทั้งหมดเป็นอิสระต่อกันในฐานข้อมูลของตัวเอง
 * operation ถูกดำเนินการตั้งแต่ตัวแรกไปจนถึงตัวสุดท้าย
 * เมื่อ operation หนึ่งล้มเหลว กระบวนการทั้งหมดจะเริ่ม roll back ย้อนไปจนถึงจุดเริ่มต้น ด้วย operation ชดเชย (compensating operation)

<div style="margin-left:3rem">
    <img src="./images/saga.png" alt="saga" width="500" />
</div>

เราจะประสานงาน (coordinate) workflow นี้อย่างไร? มีสองแนวทางที่เราสามารถทำได้:
 * Choreography - บริการทั้งหมดที่เกี่ยวข้องกับ saga สมัครสมาชิก (subscribe) event ที่เกี่ยวข้อง และทำหน้าที่ของตัวเองใน saga
 * Orchestration - coordinator ตัวเดียวสั่งการบริการทั้งหมดให้ทำงานของตัวเองตามลำดับที่ถูกต้อง

ความท้าทายของการใช้ choreography คือ business logic ถูกแบ่งกระจายไปยังหลายบริการ ซึ่งสื่อสารกันแบบอะซิงโครนัส
แนวทาง orchestration จัดการความซับซ้อนได้ดี ดังนั้นจึงมักเป็นแนวทางที่นิยมใช้ในระบบกระเป๋าเงินดิจิทัล

นี่คือการเปรียบเทียบระหว่าง TC/C และ Saga:

|                                           | TC/C            | Saga                     |
|-------------------------------------------|-----------------|--------------------------|
| การดำเนินการชดเชย (Compensating action)                       | ใน Cancel phase | ใน rollback phase        |
| การประสานงานจากศูนย์กลาง (Central coordination)                      | มี             | มี (โหมด orchestration) |
| ลำดับการดำเนินการ (Operation execution order)                 | ใดก็ได้             | เชิงเส้น (linear)                   |
| ความเป็นไปได้ในการดำเนินการแบบขนาน (Parallel execution possibility)            | มี             | ไม่มี (ดำเนินการเชิงเส้น)    |
| อาจเห็นสถานะที่ไม่สอดคล้องกันบางส่วน (Could see the partial inconsistent status) | มี              | มี                       |
| อยู่ใน Application หรือ database logic             | Application     | Application              |

ความแตกต่างหลักคือ TC/C สามารถทำงานแบบขนานได้ ดังนั้นการตัดสินใจของเราจึงขึ้นอยู่กับความต้องการด้าน latency - หากเราต้องการ latency ต่ำ เราควรเลือกใช้แนวทาง TC/C

ไม่ว่าเราจะเลือกแนวทางใด เรายังคงต้องรองรับการตรวจสอบ (auditing) และการ replay ประวัติ เพื่อกู้คืนจากสถานะที่ล้มเหลว

### **Event sourcing**
ในชีวิตจริง แอปพลิเคชันกระเป๋าเงินดิจิทัลอาจถูกตรวจสอบ (audit) และเราต้องตอบคำถามบางอย่าง:
 * เรารู้หรือไม่ว่ายอดคงเหลือของบัญชีเป็นเท่าไร ณ เวลาใดเวลาหนึ่ง?
 * เรารู้ได้อย่างไรว่ายอดคงเหลือในอดีตและปัจจุบันนั้นถูกต้อง?
 * เราจะพิสูจน์ได้อย่างไรว่าตรรกะของระบบยังคงถูกต้องหลังจากมีการเปลี่ยนแปลงโค้ด?

Event sourcing เป็นเทคนิคที่ช่วยเราตอบคำถามเหล่านี้ได้

มันประกอบด้วยแนวคิดสี่อย่าง:
 * command - การกระทำที่ตั้งใจจากโลกจริง เช่น โอนเงิน $1 จากบัญชี A ไปยัง B ต้องมีลำดับแบบ global เนื่องจากจะถูกใส่ไว้ใน FIFO queue
   * command ต่างจาก event ตรงที่สามารถล้มเหลวได้ และมีความสุ่ม (randomness) บ้าง เนื่องจาก IO หรือสถานะที่ไม่ถูกต้อง
   * command สามารถสร้าง event ได้ตั้งแต่ศูนย์ตัวขึ้นไป
   * การสร้าง event อาจมีความสุ่ม เช่น external IO เข้ามาเกี่ยวข้อง จะกล่าวถึงเรื่องนี้อีกครั้งในภายหลัง
 * event - ข้อเท็จจริงในประวัติศาสตร์เกี่ยวกับเหตุการณ์ที่เกิดขึ้นในระบบ เช่น "โอนเงิน $1 จาก A ไปยัง B"
   * ต่างจาก command ตรงที่ event เป็นข้อเท็จจริงที่เกิดขึ้นแล้วภายในระบบของเรา
   * เช่นเดียวกับ command event ต้องถูกเรียงลำดับ ดังนั้นจึงถูกใส่ไว้ใน FIFO queue
 * state - สิ่งที่เปลี่ยนแปลงไปอันเป็นผลจาก event เช่น key-value store ระหว่างบัญชีกับยอดคงเหลือของบัญชีนั้น
 * state machine - ขับเคลื่อนกระบวนการ event sourcing โดยหลักแล้วจะตรวจสอบ (validate) command และนำ event มาปรับใช้เพื่ออัปเดตสถานะของระบบ
   * state machine ควรเป็นแบบ deterministic ดังนั้นจึงไม่ควรอ่าน external IO หรือพึ่งพาความสุ่ม

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

นี่คือมุมมองเชิงพลวัต (dynamic view) ของ event sourcing:

<div style="margin-left:3rem">
    <img src="./images/dynamic-event-sourcing.png" alt="dynamic-event-sourcing" width="500" />
</div>

สำหรับ wallet service ของเรา command คือ request การโอนยอดคงเหลือ เราสามารถใส่ command เหล่านี้ไว้ใน FIFO queue เช่น Kafka:

<div style="margin-left:3rem">
    <img src="./images/command-queue.png" alt="command-queue" width="500" />
</div>

นี่คือภาพรวมทั้งหมด:

<div style="margin-left:3rem">
    <img src="./images/wallet-service-state-macghine.png" alt="wallet-service-state-machine" width="500" />
</div>

 * state machine อ่าน command จาก command queue
 * สถานะยอดคงเหลือ (balance state) ถูกอ่านจากฐานข้อมูล
 * command ถูกตรวจสอบ (validate) หากถูกต้อง จะสร้าง event ขึ้นสองรายการสำหรับแต่ละบัญชี
 * event ถัดไปจะถูกอ่านและนำมาปรับใช้ โดยการอัปเดตยอดคงเหลือ (state) ในฐานข้อมูล

ข้อได้เปรียบหลักของการใช้ event sourcing คือความสามารถในการทำซ้ำได้ (reproducibility) ในการออกแบบนี้ operation การอัปเดตสถานะทั้งหมดจะถูกบันทึกเป็นประวัติที่ไม่เปลี่ยนแปลง (immutable) ของการเปลี่ยนแปลงยอดคงเหลือทั้งหมด

ยอดคงเหลือในอดีตสามารถถูกสร้างขึ้นใหม่ได้เสมอ โดยการ replay event ตั้งแต่ต้น
เนื่องจากรายการ event นั้นไม่เปลี่ยนแปลงและ state machine เป็นแบบ deterministic เราจึงมั่นใจได้ว่าจะสามารถ replay สถานะระหว่างกลางใด ๆ ได้สำเร็จ

<div style="margin-left:3rem">
    <img src="./images/historical-states.png" alt="historical-states" width="500" />
</div>

คำถามที่เกี่ยวข้องกับการตรวจสอบ (audit) ทั้งหมดที่ถามไว้ในตอนต้นของหัวข้อนี้ สามารถตอบได้โดยอาศัย event sourcing:
 * เรารู้หรือไม่ว่ายอดคงเหลือของบัญชีเป็นเท่าไร ณ เวลาใดเวลาหนึ่ง? - สามารถ replay event ตั้งแต่ต้นจนถึงจุดเวลาที่เราสนใจได้
 * เรารู้ได้อย่างไรว่ายอดคงเหลือในอดีตและปัจจุบันนั้นถูกต้อง? - ความถูกต้องสามารถตรวจสอบได้โดยการคำนวณ event ใหม่ทั้งหมดตั้งแต่ต้น
 * เราจะพิสูจน์ได้อย่างไรว่าตรรกะของระบบยังคงถูกต้องหลังจากมีการเปลี่ยนแปลงโค้ด? - เราสามารถรันโค้ดคนละเวอร์ชันกับชุด event เดียวกัน แล้วตรวจสอบว่าผลลัพธ์เหมือนกันหรือไม่

การตอบคำถามของลูกค้าเกี่ยวกับยอดคงเหลือของตัวเอง สามารถจัดการได้โดยใช้สถาปัตยกรรม CQRS - สามารถมี read-only state machine หลายตัวที่รับผิดชอบในการ query สถานะในอดีต โดยอิงจากรายการ event ที่ไม่เปลี่ยนแปลง:

<div style="margin-left:3rem">
    <img src="./images/cqrs-architecture.png" alt="cqrs-architecture" width="500" />
</div>

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)
ในหัวข้อนี้ เราจะสำรวจการปรับปรุงประสิทธิภาพบางส่วน เนื่องจากเรายังคงต้องขยายระบบให้รองรับ 1 ล้าน TPS

### **Event sourcing ประสิทธิภาพสูง (High-performance event sourcing)**
การปรับปรุงประสิทธิภาพแรกที่เราจะสำรวจคือการบันทึก command และ event ลงใน local disk store แทนที่จะใช้ external store เช่น Kafka

วิธีนี้หลีกเลี่ยง network latency และเนื่องจากเราทำแค่การ append เท่านั้น operation นั้นโดยทั่วไปจะเร็วสำหรับ HDD

การปรับปรุงถัดไปคือการแคช command และ event ล่าสุดไว้ใน memory เพื่อประหยัดเวลาในการโหลดข้อมูลกลับมาจากดิสก์

ในระดับล่าง เราสามารถบรรลุการปรับปรุงข้างต้นได้ โดยใช้ประโยชน์จากคำสั่งที่เรียกว่า mmap ซึ่งจัดเก็บข้อมูลใน local disk พร้อมทั้งแคชไว้ใน memory:

<div style="margin-left:3rem">
    <img src="./images/mmap-optimization.png" alt="mmap-optimization" width="500" />
</div>

การปรับปรุงถัดไปที่เราสามารถทำได้คือการจัดเก็บ state ไว้ใน local file system โดยใช้ SQLite - ฐานข้อมูลเชิงสัมพันธ์แบบ file-based ในเครื่อง RocksDB ก็เป็นอีกตัวเลือกที่ดีเช่นกัน

สำหรับวัตถุประสงค์ของเรา เราจะเลือก RocksDB เนื่องจากมันใช้ log-structured merge-tree (LSM) ซึ่งถูก optimize สำหรับ operation การเขียน
ประสิทธิภาพในการอ่านถูก optimize ผ่านการแคช

<div style="margin-left:3rem">
    <img src="./images/rocks-db-approach.png" alt="rocks-db-approach" width="500" />
</div>

เพื่อปรับปรุงความสามารถในการทำซ้ำ (reproducibility) เราสามารถบันทึก snapshot ลงดิสก์เป็นระยะ ๆ เพื่อไม่ต้องสร้าง state ขึ้นใหม่จากจุดเริ่มต้นทุกครั้ง เราสามารถจัดเก็บ snapshot เป็นไฟล์ binary ขนาดใหญ่ใน distributed file storage เช่น HDFS:

<div style="margin-left:3rem">
    <img src="./images/snapshot-approach.png" alt="snapshot-approach" width="500" />
</div>

### **Event sourcing ที่เชื่อถือได้และมีประสิทธิภาพสูง (Reliable high-performance event sourcing)**
การปรับปรุงทั้งหมดที่ทำมานั้นดีมาก แต่ทำให้บริการของเรากลายเป็น stateful เราจำเป็นต้องนำการทำ replication รูปแบบใดรูปแบบหนึ่งมาใช้ เพื่อความน่าเชื่อถือ (reliability)

ก่อนที่เราจะทำเช่นนั้น เราควรวิเคราะห์ก่อนว่าข้อมูลประเภทใดในระบบของเราต้องการความน่าเชื่อถือสูง:
 * state และ snapshot สามารถถูกสร้างขึ้นใหม่ได้เสมอโดยการสร้างขึ้นจากรายการ event ดังนั้นเราจึงต้องรับประกันความน่าเชื่อถือของรายการ event เท่านั้น
 * บางคนอาจคิดว่าเราสามารถสร้างรายการ event ขึ้นใหม่จากรายการ command ได้เสมอ แต่นั่นไม่จริง เนื่องจาก command ไม่ใช่แบบ deterministic
 * สรุปคือเราจำเป็นต้องรับประกันความน่าเชื่อถือสูงสำหรับรายการ event เท่านั้น

เพื่อให้ได้ความน่าเชื่อถือสูงสำหรับ event เราจำเป็นต้องทำสำเนารายการนี้ไปยังหลาย node เราจำเป็นต้องรับประกันว่า:
 * ไม่มีข้อมูลสูญหาย
 * ลำดับสัมพัทธ์ (relative order) ของข้อมูลภายในไฟล์ log ยังคงเหมือนกันในทุก replica

เพื่อบรรลุสิ่งนี้ เราสามารถใช้ consensus algorithm เช่น Raft

ใน Raft จะมี leader ที่ทำงานอยู่ (active) และมี follower ที่ทำงานแบบ passive หาก leader ตาย follower ตัวหนึ่งจะเข้ามารับหน้าที่แทน
ตราบใดที่มากกว่าครึ่งหนึ่งของ node ยังคงทำงานอยู่ ระบบจะยังคงทำงานต่อไปได้

<div style="margin-left:3rem">
    <img src="./images/raft-replication.png" alt="raft-replication" width="500" />
</div>

ด้วยแนวทางนี้ node ทั้งหมดจะอัปเดต state โดยอิงจากรายการ event Raft รับประกันว่า leader และ follower มีรายการ event เดียวกัน

### **Event sourcing แบบกระจาย (Distributed event sourcing)**
มาถึงตอนนี้ เราออกแบบระบบที่มีประสิทธิภาพสูงในระดับ node เดียวและมีความน่าเชื่อถือได้สำเร็จแล้ว

ข้อจำกัดบางประการที่เราต้องจัดการต่อ:
 * ความจุของ raft group เดียวมีจำกัด ในบางจุดเราจำเป็นต้อง shard ข้อมูลและ implement distributed transaction
 * ในสถาปัตยกรรม CQRS การไหลของ request/response นั้นช้า ไคลเอนต์จะต้อง poll ระบบเป็นระยะเพื่อรู้ว่ากระเป๋าเงินของตัวเองถูกอัปเดตเมื่อไร

การ polling ไม่ใช่แบบ real-time ดังนั้นอาจใช้เวลาสักพักก่อนที่ผู้ใช้จะรู้ว่ายอดคงเหลือของตัวเองถูกอัปเดต นอกจากนี้ยังอาจทำให้ query service รับภาระหนักเกินไป หากความถี่ในการ polling สูงเกินไป:

<div style="margin-left:3rem">
    <img src="./images/polling-approach.png" alt="polling-approach" width="500" />
</div>

เพื่อลดภาระของระบบ เราสามารถนำ reverse proxy มาใช้ ซึ่งส่ง command แทนผู้ใช้และ poll หา response แทนผู้ใช้:

<div style="margin-left:3rem">
    <img src="./images/reverse-proxy.png" alt="reverse-proxy" width="500" />
</div>

วิธีนี้ช่วยลดภาระของระบบ เนื่องจากเราสามารถดึงข้อมูลของผู้ใช้หลายคนได้ด้วย request เดียว แต่มันยังคงไม่ได้แก้ปัญหาความต้องการด้านการได้รับผลลัพธ์แบบ real-time

การเปลี่ยนแปลงสุดท้ายที่เราสามารถทำได้คือ ให้ read-only state machine ส่ง (push) response กลับไปยัง reverse proxy ทันทีที่มันพร้อม วิธีนี้ทำให้ผู้ใช้รู้สึกว่าการอัปเดตเกิดขึ้นแบบ real-time:

<div style="margin-left:3rem">
    <img src="./images/push-state-machines.png" alt="push-state-machines" width="500" />
</div>

สุดท้าย เพื่อขยาย (scale) ระบบให้มากยิ่งขึ้น เราสามารถ shard ระบบออกเป็นหลาย raft group โดยเรา implement distributed transaction บนพื้นฐานของมันโดยใช้ orchestrator ผ่าน TC/C หรือ Saga:

<div style="margin-left:3rem">
    <img src="./images/sharded-raft-groups.png" alt="sharded-raft-groups" width="500" />
</div>

นี่คือตัวอย่างวงจรชีวิต (lifecycle) ของ request การโอนยอดคงเหลือในระบบสุดท้ายของเรา:
 * ผู้ใช้ A ส่ง distributed transaction ไปยัง Saga coordinator พร้อมสอง operation - `A-1` และ `C+1`
 * Saga coordinator สร้าง record ในตาราง phase status เพื่อติดตามสถานะของ transaction
 * coordinator กำหนด partition ที่ต้องส่ง command ไป
 * raft leader ของ Partition 1 ได้รับคำสั่ง `A-1` ตรวจสอบความถูกต้อง แปลงเป็น event แล้วทำสำเนาไปยัง node อื่น ๆ ใน raft group
 * ผลลัพธ์ของ event ถูกซิงค์ไปยัง read state machine ซึ่งส่ง response กลับไปยัง coordinator
 * coordinator สร้าง record ระบุว่า operation สำเร็จ และดำเนินการต่อไปยัง operation ถัดไป - `C+1`
 * operation ถัดไปถูกดำเนินการในลักษณะเดียวกับตัวแรก - มีการกำหนด partition, ส่งคำสั่ง, ดำเนินการ, read state machine ส่ง response กลับ
 * coordinator สร้าง record ระบุว่า operation ที่ 2 สำเร็จเช่นกัน และสุดท้ายแจ้งผลลัพธ์ให้ไคลเอนต์ทราบ

---

## ขั้นตอนที่ 4: สรุป (Wrap Up)
นี่คือวิวัฒนาการของการออกแบบของเรา:
 * เราเริ่มต้นจากโซลูชันที่ใช้ in-memory Redis ปัญหาของแนวทางนี้คือมันไม่ใช่ durable storage
 * เราจึงเปลี่ยนมาใช้ฐานข้อมูลเชิงสัมพันธ์ ซึ่งเราดำเนินการ distributed transaction บนพื้นฐานนั้นด้วย 2PC, TC/C หรือ distributed saga
 * ถัดมา เรานำ event sourcing มาใช้ เพื่อทำให้ operation ทั้งหมดสามารถตรวจสอบได้ (auditable)
 * เราเริ่มต้นด้วยการจัดเก็บข้อมูลใน external storage โดยใช้ external database และ queue แต่นั่นไม่มีประสิทธิภาพ
 * เราจึงดำเนินการจัดเก็บข้อมูลใน local file storage โดยใช้ประโยชน์จากประสิทธิภาพของ operation แบบ append-only เรายังใช้การแคชเพื่อปรับปรุง read path
 * แนวทางก่อนหน้านี้ แม้จะมีประสิทธิภาพ แต่ไม่ทนทาน (durable) ดังนั้นเราจึงนำ Raft consensus พร้อมการทำ replication มาใช้ เพื่อหลีกเลี่ยง single point of failure
 * เรายังนำ CQRS มาใช้ร่วมกับ reverse proxy เพื่อจัดการวงจรชีวิตของ transaction แทนผู้ใช้ของเรา
 * สุดท้าย เรา partition ข้อมูลของเราไปยังหลาย raft group ซึ่งถูก orchestrate โดยใช้กลไก distributed transaction - TC/C หรือ distributed saga
