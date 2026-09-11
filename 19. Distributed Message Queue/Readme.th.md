# บทที่ 19: คิวข้อความแบบกระจาย (Distributed Message Queue)

## บทนำ

ในบทนี้ เราจะออกแบบ **คิวข้อความแบบกระจาย (distributed message queue)**

ประโยชน์ของ message queue มีดังนี้:
- **การลดการพึ่งพากัน (Decoupling):** ขจัดการผูกติดกันอย่างแน่นหนา (tight coupling) ระหว่างคอมโพเนนต์ ทำให้แต่ละส่วนสามารถอัปเดตแยกจากกันได้
- **เพิ่มความสามารถในการขยายระบบ (Improved scalability):** producer และ consumer สามารถขยาย (scale) แยกจากกันได้ตามปริมาณทราฟฟิก
- **เพิ่มความพร้อมใช้งาน (Increased availability):** หากระบบส่วนหนึ่งล่ม ส่วนอื่น ๆ ยังคงสามารถทำงานร่วมกับคิวต่อไปได้
- **ประสิทธิภาพที่ดีขึ้น (Better performance):** producer สามารถผลิตข้อความได้โดยไม่ต้องรอการยืนยันจาก consumer

การใช้งาน message queue ที่ได้รับความนิยม ได้แก่ Kafka, RabbitMQ, RocketMQ, Apache Pulsar, ActiveMQ, ZeroMQ

พูดกันอย่างเคร่งครัดแล้ว Kafka และ Pulsar ไม่ใช่ message queue แต่เป็น event streaming platform อย่างไรก็ตาม ฟีเจอร์ของทั้งสองประเภทเริ่มมาบรรจบกัน จนทำให้เส้นแบ่งระหว่าง message queue และ event streaming platform เลือนลางลง

ในบทนี้ เราจะสร้าง message queue ที่รองรับฟีเจอร์ขั้นสูงมากขึ้น เช่น การเก็บข้อมูลระยะยาว (long data retention), การบริโภคข้อความซ้ำได้ (repeated message consumption) และอื่น ๆ

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ

Message queue ควรรองรับฟีเจอร์พื้นฐานเพียงไม่กี่อย่าง คือ producer ผลิตข้อความ และ consumer บริโภคข้อความเหล่านั้น อย่างไรก็ตาม ยังมีข้อพิจารณาอื่น ๆ ที่แตกต่างกันไป เช่น ประสิทธิภาพ, การส่งมอบข้อความ (message delivery), การเก็บข้อมูล (data retention) เป็นต้น

ตัวอย่างชุดคำถามระหว่างผู้สมัคร (Candidate) และผู้สัมภาษณ์ (Interviewer):
 * C: รูปแบบและขนาดเฉลี่ยของข้อความเป็นอย่างไร? เป็นข้อความล้วน (text only) หรือไม่?
 * I: ข้อความเป็นข้อความล้วน และโดยทั่วไปมีขนาดไม่กี่ KB
 * C: ข้อความสามารถถูกบริโภคซ้ำได้หรือไม่?
 * I: ได้ ข้อความสามารถถูกบริโภคซ้ำโดย consumer ที่ต่างกันได้ นี่เป็นข้อกำหนดเพิ่มเติมที่ message queue แบบดั้งเดิมไม่รองรับ
 * C: ข้อความถูกบริโภคตามลำดับเดียวกับที่ถูกผลิตหรือไม่?
 * I: ใช่ ต้องรักษาการรับประกันลำดับ (order guarantee) ไว้ นี่เป็นข้อกำหนดเพิ่มเติมที่ message queue แบบดั้งเดิมไม่รองรับ
 * C: ข้อกำหนดเรื่องการเก็บรักษาข้อมูล (data retention) เป็นอย่างไร?
 * I: ข้อความต้องถูกเก็บรักษาไว้เป็นเวลาสองสัปดาห์ นี่เป็นข้อกำหนดเพิ่มเติม
 * C: เราต้องรองรับ producer และ consumer จำนวนเท่าใด?
 * I: ยิ่งมากยิ่งดี
 * C: เราต้องการรองรับ data delivery semantic แบบใด? At-most-once, at-least-once, หรือ exactly-once?
 * I: เราต้องการรองรับ at-least-once อย่างแน่นอน และหากเป็นไปได้ อยากรองรับได้ทั้งหมดและปรับตั้งค่าได้
 * C: throughput เป้าหมายสำหรับ end-to-end latency คือเท่าใด?
 * I: ควรรองรับ throughput สูงสำหรับกรณีการใช้งานอย่าง log aggregation และรองรับ throughput ต่ำสำหรับกรณีการใช้งานแบบดั้งเดิมทั่วไป

### **ข้อกำหนดเชิงฟังก์ชัน (Functional requirements)**

 * Producer ส่งข้อความไปยัง message queue
 * Consumer บริโภคข้อความจากคิว
 * ข้อความสามารถถูกบริโภคได้ครั้งเดียวหรือซ้ำได้
 * ข้อมูลย้อนหลัง (historical data) สามารถถูกตัดทอน (truncate) ได้
 * ขนาดข้อความอยู่ในระดับ KB
 * ต้องรักษาลำดับของข้อความ
 * data delivery semantic สามารถปรับตั้งค่าได้ - at-most-once/at-least-once/exactly-once

### **ข้อกำหนดที่ไม่ใช่เชิงฟังก์ชัน (Non-functional requirements)**

- **Throughput สูงหรือ latency ต่ำ:** ปรับตั้งค่าได้ตามกรณีการใช้งาน
- **ขยายระบบได้ (Scalable):** ระบบควรเป็นแบบกระจาย (distributed) และรองรับปริมาณข้อความที่พุ่งสูงขึ้นอย่างกะทันหันได้
- **คงทนและถาวร (Persistent and durable):** ข้อมูลควรถูกบันทึกลงดิสก์ และทำสำเนา (replicate) ไว้ระหว่างโหนดต่าง ๆ

Message queue แบบดั้งเดิมมักไม่รองรับการเก็บรักษาข้อมูล (data retention) และไม่การันตีลำดับของข้อความ ซึ่งช่วยให้การออกแบบง่ายขึ้นมาก และเราจะพูดถึงประเด็นนี้กัน

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ

คอมโพเนนต์หลักของ message queue:

<div style="margin-left:3rem">
    <img src="./images/message-queue-components.png" alt="message-queue-components" width="500" />
</div>

 * Producer ส่งข้อความไปยังคิว
 * Consumer สมัครสมาชิก (subscribe) กับคิวและบริโภคข้อความที่ตนสมัครไว้
 * Message queue เป็นบริการตรงกลางที่ทำหน้าที่แยก producer ออกจาก consumer ทำให้แต่ละฝั่งสามารถขยายระบบได้อย่างอิสระ
 * ทั้ง producer และ consumer เป็นไคลเอนต์ (client) ในขณะที่ message queue เป็นเซิร์ฟเวอร์

### **รูปแบบการส่งข้อความ (Messaging models)**

รูปแบบการส่งข้อความแบบแรกคือแบบ point-to-point ซึ่งพบได้ทั่วไปใน message queue แบบดั้งเดิม:

<div style="margin-left:3rem">
    <img src="./images/point-to-point-model.png" alt="point-to-point-model" width="500" />
</div>

 * ข้อความหนึ่งถูกส่งไปยังคิว และถูกบริโภคโดย consumer เพียงหนึ่งตัวเท่านั้น
 * สามารถมี consumer หลายตัวได้ แต่ข้อความหนึ่งจะถูกบริโภคเพียงครั้งเดียว
 * เมื่อข้อความถูกยืนยัน (acknowledge) ว่าถูกบริโภคแล้ว ข้อความนั้นจะถูกลบออกจากคิว
 * ในรูปแบบ point-to-point จะไม่มีการเก็บรักษาข้อมูล (data retention) แต่ในการออกแบบของเรามีการเก็บรักษาข้อมูลนี้

ในทางกลับกัน รูปแบบ publish-subscribe เป็นที่นิยมมากกว่าสำหรับ event streaming platform:

<div style="margin-left:3rem">
    <img src="./images/publish-subscribe-model.png" alt="publish-subscribe-model" width="500" />
</div>

 * ในรูปแบบนี้ ข้อความจะถูกผูกไว้กับหัวข้อ (topic)
 * Consumer จะสมัครสมาชิกกับ topic หนึ่ง ๆ และรับข้อความทั้งหมดที่ถูกส่งไปยัง topic นั้น

### **Topic, Partition และ Broker**

จะเกิดอะไรขึ้นหากปริมาณข้อมูลของ topic หนึ่งมีมากเกินไป? วิธีหนึ่งในการขยายระบบคือการแบ่ง topic ออกเป็น partition ต่าง ๆ (หรือที่เรียกว่า sharding):

<div style="margin-left:3rem">
    <img src="./images/partitions.png" alt="partitions" width="500" />
</div>

 * ข้อความที่ถูกส่งไปยัง topic จะถูกกระจายอย่างเท่าเทียมกันไปยัง partition ต่าง ๆ
 * เซิร์ฟเวอร์ที่ทำหน้าที่จัดเก็บ partition เรียกว่า broker
 * แต่ละ topic ทำงานเสมือนคิวโดยใช้หลัก FIFO ในการประมวลผลข้อความ ลำดับของข้อความจะถูกรักษาไว้ภายใน partition เดียวกัน
 * ตำแหน่งของข้อความภายใน partition เรียกว่า **offset**
 * ข้อความที่ถูกผลิตแต่ละชิ้นจะถูกส่งไปยัง partition ที่กำหนดไว้เฉพาะ โดย partition key จะระบุว่าข้อความควรไปอยู่ที่ partition ใด
   * ตัวอย่างเช่น `user_id` สามารถใช้เป็น partition key เพื่อรับประกันลำดับของข้อความสำหรับผู้ใช้คนเดียวกัน
 * Consumer แต่ละตัวจะสมัครสมาชิกกับ partition หนึ่งตัวหรือมากกว่า เมื่อมี consumer หลายตัวสำหรับข้อความชุดเดียวกัน พวกมันจะรวมกันเป็น consumer group

### **Consumer Group**

Consumer group คือกลุ่มของ consumer ที่ทำงานร่วมกันเพื่อบริโภคข้อความจาก topic หนึ่ง ๆ:

<div style="margin-left:3rem">
    <img src="./images/consumer-groups.png" alt="consumer-groups" width="500" />
</div>

 * ข้อความจะถูกทำสำเนาต่อ consumer group (ไม่ใช่ต่อ consumer แต่ละตัว)
 * แต่ละ consumer group จะดูแลรักษา offset ของตัวเอง
 * การอ่านข้อความแบบขนาน (parallel) โดย consumer group ช่วยเพิ่ม throughput แต่จะกระทบต่อการรับประกันลำดับ
 * ปัญหานี้สามารถบรรเทาได้โดยอนุญาตให้มี consumer เพียงหนึ่งตัวจากกลุ่มที่สมัครสมาชิกกับ partition หนึ่ง ๆ
 * นั่นหมายความว่า เราไม่สามารถมี consumer ในกลุ่มมากกว่าจำนวน partition ที่มีอยู่ได้

### **สถาปัตยกรรมระดับสูง (High-level architecture)**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- **Client:** producer และ consumer. Producer ผลักข้อความไปยัง topic ที่กำหนดไว้ ส่วน consumer group จะสมัครสมาชิกเพื่อรับข้อความจาก topic นั้น
- **Broker:** เก็บ partition หลายตัว โดยแต่ละ partition จะเก็บส่วนหนึ่งของข้อความสำหรับ topic นั้น ๆ
- **Data storage:** จัดเก็บข้อความไว้ใน partition
- **State storage:** เก็บสถานะของ consumer
- **Metadata storage:** เก็บค่าคอนฟิกและคุณสมบัติของ topic
- **Coordination service:** รับผิดชอบเรื่อง service discovery (broker ตัวใดยังทำงานอยู่) และการเลือกตั้งผู้นำ (leader election - broker ตัวใดเป็น leader ที่รับผิดชอบการจัดสรร partition)

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ

เพื่อให้ได้ throughput สูงและรักษาข้อกำหนดด้าน data retention ที่สูง เราได้ตัดสินใจเลือกการออกแบบที่สำคัญดังนี้:
 * เราเลือกโครงสร้างข้อมูลที่จัดเก็บบนดิสก์ (on-disk data structure) ซึ่งใช้ประโยชน์จากคุณสมบัติของ HDD สมัยใหม่ และกลยุทธ์การแคชดิสก์ (disk caching) ของ OS สมัยใหม่
 * โครงสร้างข้อมูลของข้อความเป็นแบบ immutable (แก้ไขไม่ได้) เพื่อหลีกเลี่ยงการทำสำเนาข้อมูลเพิ่มเติม ซึ่งเราต้องการหลีกเลี่ยงในระบบที่มีปริมาณ/ทราฟฟิกสูง
 * เราออกแบบการเขียนข้อมูลโดยยึดหลักการทำแบบเป็นชุด (batching) เนื่องจากการทำ I/O ขนาดเล็กเป็นศัตรูของ throughput สูง

### **Data storage**

เพื่อหา data store ที่ดีที่สุดสำหรับข้อความ เราต้องพิจารณาคุณสมบัติของข้อความก่อน:
 * เน้นการเขียนหนักและอ่านหนัก (write-heavy, read-heavy)
 * ไม่มี operation แบบ update/delete ใน message queue แบบดั้งเดิมจะมี operation "delete" เนื่องจากข้อความไม่ถูกเก็บรักษาไว้
 * รูปแบบการเข้าถึง (access pattern) ส่วนใหญ่เป็นแบบลำดับ (sequential)

ตัวเลือกที่เรามี:
- **ฐานข้อมูล (Database):** ไม่เหมาะสมนัก เนื่องจากฐานข้อมูลทั่วไปไม่รองรับทั้งระบบที่เขียนหนักและอ่านหนักได้ดี
- **Write-ahead log (WAL):** ไฟล์ข้อความล้วนที่รองรับเพียงการ append (ต่อท้าย) เท่านั้น และเป็นมิตรกับ HDD มาก
  * เราแบ่ง partition ออกเป็น segment เพื่อหลีกเลี่ยงการดูแลไฟล์ขนาดใหญ่มากเกินไป
  * segment เก่าจะเป็นแบบอ่านอย่างเดียว (read-only) มีเพียง segment ล่าสุดเท่านั้นที่รับการเขียน

<div style="margin-left:3rem">
    <img src="./images/wal-example.png" alt="wal-example" width="500" />
</div>

ไฟล์ WAL มีประสิทธิภาพสูงมากเมื่อใช้งานร่วมกับ HDD แบบดั้งเดิม

มีความเข้าใจผิดว่าการเข้าถึง HDD นั้นช้า แต่ในความเป็นจริงขึ้นอยู่กับรูปแบบการเข้าถึงเป็นอย่างมาก
เมื่อรูปแบบการเข้าถึงเป็นแบบลำดับ (sequential) ดังเช่นกรณีของเรา HDD สามารถทำความเร็วในการเขียน/อ่านได้หลาย MB/s ซึ่งเพียงพอต่อความต้องการของเรา
เรายังใช้ประโยชน์จากข้อเท็จจริงที่ว่า OS จะแคชข้อมูลดิสก์ไว้ในหน่วยความจำอย่างจริงจังอีกด้วย

### **โครงสร้างข้อมูลของข้อความ (Message data structure)**

สิ่งสำคัญคือ schema ของข้อความต้องสอดคล้องกันระหว่าง producer, คิว และ consumer เพื่อหลีกเลี่ยงการทำสำเนาข้อมูลเพิ่มเติม ซึ่งช่วยให้การประมวลผลมีประสิทธิภาพมากขึ้น

ตัวอย่างโครงสร้างข้อความ:

<div style="margin-left:3rem">
    <img src="./images/message-structure.png" alt="message-structure" width="500" />
</div>

key ของข้อความจะระบุว่าข้อความนั้นเป็นของ partition ใด ตัวอย่างการแมป (mapping) คือ `hash(key) % numPartitions`
เพื่อเพิ่มความยืดหยุ่น producer สามารถ override key เริ่มต้นได้ เพื่อควบคุมว่าข้อความจะถูกกระจายไปยัง partition ใด

value ของข้อความคือ payload ของข้อความนั้น อาจเป็นข้อความล้วน (plaintext) หรือ binary block ที่ถูกบีบอัดก็ได้

**หมายเหตุ:** key ของข้อความ แตกต่างจาก KV store แบบดั้งเดิม ตรงที่ไม่จำเป็นต้องไม่ซ้ำกัน (unique) สามารถมี key ที่ซ้ำกันได้ และแม้กระทั่งไม่มี key เลยก็ยอมรับได้

ฟิลด์อื่น ๆ ของข้อความ:
- **Topic:** topic ที่ข้อความนี้เป็นสมาชิกอยู่
- **Partition:** ID ของ partition ที่ข้อความนี้เป็นสมาชิกอยู่
- **Offset:** ตำแหน่งของข้อความภายใน partition สามารถระบุตำแหน่งข้อความได้ผ่าน `topic`, `partition`, `offset`
- **Timestamp:** เวลาที่ข้อความถูกจัดเก็บ
- **Size:** ขนาดของข้อความนี้
- **CRC:** checksum เพื่อยืนยันความถูกต้องของข้อความ

ฟีเจอร์เพิ่มเติม เช่น การกรอง (filtering) สามารถรองรับได้โดยการเพิ่มฟิลด์เข้าไปอีก

### **การทำงานเป็นชุด (Batching)**

การทำ batching มีความสำคัญอย่างยิ่งต่อประสิทธิภาพของระบบเรา เราใช้แนวคิดนี้ทั้งที่ producer, consumer และ message queue

เหตุที่สำคัญ:
 * ช่วยให้ระบบปฏิบัติการสามารถจัดกลุ่มข้อความเข้าด้วยกัน ซึ่งช่วยลดต้นทุนของการทำ network round trip ที่มีราคาแพง
 * ข้อความถูกเขียนลง WAL เป็นกลุ่ม ๆ ตามลำดับ ซึ่งนำไปสู่การเขียนแบบ sequential จำนวนมากและการแคชดิสก์ที่มีประสิทธิภาพ

มีข้อแลกเปลี่ยน (trade-off) ระหว่าง latency และ throughput:
 * การทำ batching มากขึ้นนำไปสู่ throughput สูงขึ้นแต่ latency สูงขึ้นด้วย
 * การทำ batching น้อยลงนำไปสู่ throughput ต่ำลงแต่ latency ต่ำลงด้วย

หากเราจำเป็นต้องรองรับ latency ที่ต่ำลงเนื่องจากระบบถูก deploy ในลักษณะ message queue แบบดั้งเดิม ระบบสามารถปรับจูนให้ใช้ batch size ที่เล็กลงได้

หากปรับจูนเพื่อเน้น throughput เราอาจต้องการ partition ต่อ topic มากขึ้น เพื่อชดเชย throughput ของการเขียนดิสก์แบบ sequential ที่ช้าลง

### **Producer Flow**

หาก producer ต้องการส่งข้อความไปยัง partition หนึ่ง ๆ producer ควรเชื่อมต่อกับ broker ตัวใด?

ทางเลือกหนึ่งคือการเพิ่มชั้น routing layer เพื่อ route ข้อความไปยัง broker ที่ถูกต้อง หากเปิดใช้งานการทำ replication broker ที่ถูกต้องจะเป็น leader replica:

<div style="margin-left:3rem">
    <img src="./images/routing-layer.png" alt="routing-layer" width="500" />
</div>

 * Routing layer อ่านแผนการทำ replication (replication plan) จาก metadata store และแคชไว้ในเครื่อง
 * Producer ส่งข้อความไปยัง routing layer
 * ข้อความถูกส่งต่อไปยัง broker 1 ซึ่งเป็น leader ของ partition นั้น
 * Follower replica ดึงข้อความใหม่จาก leader เมื่อได้รับการยืนยันเพียงพอแล้ว leader จะ commit ข้อมูลและตอบกลับไปยัง producer

เหตุผลของการมี replica คือเพื่อให้ระบบทนต่อความล้มเหลว (fault tolerance) ได้

แนวทางนี้ใช้งานได้ แต่มีข้อเสียบางประการ:
 * เกิด network hop เพิ่มเติมเนื่องจากมีคอมโพเนนต์เพิ่มขึ้นมา
 * การออกแบบนี้ไม่เอื้อให้เกิดการทำ batching ข้อความได้

เพื่อบรรเทาปัญหาเหล่านี้ เราสามารถฝัง routing layer เข้าไปใน producer ได้:

<div style="margin-left:3rem">
    <img src="./images/routing-layer-producer.png" alt="routing-layer-producer" width="500" />
</div>

 * network hop ที่น้อยลงทำให้ latency ต่ำลง
 * Producer สามารถควบคุมได้ว่าข้อความจะถูก route ไปยัง partition ใด
 * บัฟเฟอร์ (buffer) ช่วยให้เราสามารถทำ batch ข้อความในหน่วยความจำ และส่ง batch ขนาดใหญ่ออกไปในคำขอเดียว ซึ่งช่วยเพิ่ม throughput

การเลือกขนาด batch เป็น trade-off แบบคลาสสิกระหว่าง throughput กับ latency

<div style="margin-left:3rem">
    <img src="./images/batch-size-throughput-vs-latency.png" alt="batch-size-throughput-vs-latency" width="500" />
</div>

 * batch size ที่ใหญ่ขึ้นนำไปสู่เวลารอที่นานขึ้นก่อนที่ batch จะถูก commit
 * batch size ที่เล็กลงทำให้คำขอถูกส่งเร็วขึ้นและมี latency ต่ำลง แต่ throughput ต่ำลงด้วย

### **Consumer Flow**

Consumer ระบุ offset ของตนใน partition และรับข้อความเป็นชุด (chunk) โดยเริ่มต้นจาก offset นั้น:

<div style="margin-left:3rem">
    <img src="./images/consumer-example.png" alt="consumer-example" width="500" />
</div>

ข้อพิจารณาที่สำคัญข้อหนึ่งเมื่อออกแบบ consumer คือควรใช้รูปแบบ push หรือ pull:
- **Push model:** ทำให้ latency ต่ำลง เนื่องจาก broker จะผลักข้อความไปให้ consumer ทันทีที่ได้รับข้อความ
  * อย่างไรก็ตาม หากอัตราการบริโภคช้ากว่าอัตราการผลิต consumer อาจรับข้อมูลไม่ไหว (overwhelmed)
  * เป็นเรื่องท้าทายในการจัดการกับ consumer ที่มีกำลังประมวลผลต่างกัน เนื่องจาก broker เป็นผู้ควบคุมอัตราการบริโภค
- **Pull model:** ทำให้ consumer เป็นผู้ควบคุมอัตราการบริโภคเอง
  * หากอัตราการบริโภคช้า consumer จะไม่รับข้อมูลไม่ไหว และเราสามารถขยาย (scale) เพื่อให้ตามทัน
  * pull model เหมาะสมกับการประมวลผลแบบ batch มากกว่า เพราะใน push model broker ไม่สามารถรู้ได้ว่า consumer รับข้อความได้กี่ข้อความ
  * ในทางกลับกัน ด้วย pull model consumer สามารถดึง (fetch) ข้อความจำนวนมากได้อย่างจริงจัง
  * ข้อเสียคือ latency ที่สูงขึ้นและมีการเรียก network เพิ่มเติมเมื่อไม่มีข้อความใหม่ ปัญหาหลังนี้สามารถบรรเทาได้ด้วยการทำ long polling

ดังนั้น message queue ส่วนใหญ่ (รวมถึงของเราด้วย) จึงเลือกใช้ pull model

<div style="margin-left:3rem">
    <img src="./images/consumer-flow.png" alt="consumer-flow" width="500" />
</div>

 * consumer ใหม่สมัครสมาชิกกับ topic A และเข้าร่วม group 1
 * broker ที่ถูกต้องจะถูกค้นหาโดยการทำ hash ชื่อของ group วิธีนี้ทำให้ consumer ทั้งหมดใน group เดียวกันเชื่อมต่อกับ broker ตัวเดียวกัน
 * โปรดสังเกตว่า consumer group coordinator นี้แตกต่างจาก coordination service (ZooKeeper)
 * Coordinator ยืนยันว่า consumer ได้เข้าร่วม group แล้ว และจัดสรร partition 2 ให้กับ consumer ตัวนั้น
 * มีกลยุทธ์การจัดสรร partition ที่แตกต่างกันไป - round-robin, range เป็นต้น
 * Consumer ดึงข้อความล่าสุดจาก offset สุดท้าย state storage เป็นผู้เก็บ offset ของ consumer
 * Consumer ประมวลผลข้อความและ commit offset ไปยัง broker ลำดับของ operation เหล่านี้มีผลต่อ message delivery semantic

### **การ Rebalance ของ Consumer (Consumer rebalancing)**

Consumer rebalancing รับผิดชอบในการตัดสินใจว่า consumer ตัวใดรับผิดชอบ partition ใด

กระบวนการนี้จะเกิดขึ้นเมื่อ consumer เข้าร่วม/ออกจากกลุ่ม หรือเมื่อมีการเพิ่ม/ลบ partition

broker ซึ่งทำหน้าที่เป็น coordinator มีบทบาทสำคัญอย่างมากในการควบคุมดูแล (orchestrate) กระบวนการ rebalancing

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalancing.png" alt="consumer-rebalancing" width="500" />
</div>

 * consumer ทั้งหมดในกลุ่มเดียวกันเชื่อมต่อกับ coordinator ตัวเดียวกัน โดย coordinator จะถูกค้นหาจากการ hash ชื่อ group
 * เมื่อรายชื่อ consumer เปลี่ยนแปลง coordinator จะเลือก leader ใหม่ของกลุ่ม
 * leader ของกลุ่มจะคำนวณแผนการจัดสรร partition (partition dispatch plan) ใหม่ และรายงานกลับไปยัง coordinator ซึ่งจะกระจายข้อมูลนี้ไปยัง consumer ตัวอื่น ๆ

เมื่อ coordinator หยุดรับ heartbeat จาก consumer ในกลุ่ม กระบวนการ rebalancing จะถูกกระตุ้นให้เริ่มทำงาน:

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalance-example.png" alt="consumer-rebalance-example" width="500" />
</div>

มาสำรวจกันว่าจะเกิดอะไรขึ้นเมื่อ consumer เข้าร่วมกลุ่ม:

<div style="margin-left:3rem">
    <img src="./images/consumer-join-group-usecase.png" alt="consumer-join-group-usecase" width="500" />
</div>

 * ในตอนแรก มีเพียง consumer A อยู่ในกลุ่ม และบริโภคข้อความจากทุก partition
 * consumer B ส่งคำขอเพื่อเข้าร่วมกลุ่ม
 * coordinator แจ้งสมาชิกทุกคนในกลุ่มว่าถึงเวลาต้อง rebalance แบบ passive - เป็นการตอบกลับต่อ heartbeat
 * เมื่อ consumer ทั้งหมดกลับเข้าร่วมกลุ่มแล้ว coordinator จะเลือก leader และแจ้งผลการเลือกตั้งให้ทุกคนทราบ
 * leader สร้างแผนการจัดสรร partition และส่งไปยัง coordinator ส่วนตัวอื่น ๆ จะรอแผนดังกล่าว
 * consumer เริ่มบริโภคข้อความจาก partition ที่ได้รับมอบหมายใหม่

ต่อไปนี้คือสิ่งที่เกิดขึ้นเมื่อ consumer ออกจากกลุ่ม:

<div style="margin-left:3rem">
    <img src="./images/consumer-leaves-group-usecase.png" alt="consumer-leaves-group-usecase" width="500" />
</div>

 * consumer A และ B อยู่ในกลุ่มเดียวกัน
 * consumer B ขอออกจากกลุ่ม
 * เมื่อ coordinator ได้รับ heartbeat ของ A จะแจ้งให้ทราบว่าถึงเวลาต้อง rebalance
 * ขั้นตอนที่เหลือเหมือนเดิม

กระบวนการจะคล้ายกันเมื่อ consumer ไม่ส่ง heartbeat เป็นเวลานาน:

<div style="margin-left:3rem">
    <img src="./images/consumer-no-heartbeat-usecase.png" alt="consumer-no-heartbeat-usecase" width="500" />
</div>

### **State Storage**

State storage เก็บการแมประหว่าง partition กับ consumer รวมถึง offset ล่าสุดที่ถูกบริโภคของแต่ละ partition

<div style="margin-left:3rem">
    <img src="./images/state-storage.png" alt="state-storage" width="500" />
</div>

offset ของ Group 1 อยู่ที่ 6 หมายความว่าข้อความก่อนหน้าทั้งหมดถูกบริโภคแล้ว หาก consumer ตัวใดตัวหนึ่งล่ม consumer ตัวใหม่จะดำเนินการต่อจากข้อความนั้นเป็นต้นไป

รูปแบบการเข้าถึงข้อมูล (data access pattern) สำหรับสถานะของ consumer:
 * มี read/write operation บ่อยครั้ง แต่ปริมาณต่ำ
 * ข้อมูลถูกอัปเดตบ่อย แต่แทบไม่ถูกลบ
 * เป็นการอ่าน/เขียนแบบสุ่ม (random)
 * ความสอดคล้องของข้อมูล (consistency) มีความสำคัญ

จากข้อกำหนดเหล่านี้ KV storage ที่รวดเร็วอย่าง Zookeeper จึงเหมาะสมที่สุด

### **Metadata Storage**

Metadata storage เก็บค่าคอนฟิกและคุณสมบัติของ topic - จำนวน partition, ระยะเวลาการเก็บรักษาข้อมูล (retention period), การกระจาย replica

Metadata มักไม่เปลี่ยนแปลงบ่อยและมีปริมาณน้อย แต่มีข้อกำหนดด้านความสอดคล้อง (consistency) สูง
Zookeeper จึงเป็นตัวเลือกที่ดีสำหรับที่เก็บข้อมูลนี้

### **ZooKeeper**

Zookeeper มีความสำคัญอย่างยิ่งต่อการสร้าง distributed message queue

มันเป็น key-value store แบบลำดับชั้น (hierarchical) ที่มักถูกใช้สำหรับการตั้งค่าแบบกระจาย (distributed configuration), บริการซิงโครไนซ์ (synchronization service) และ naming registry (กล่าวคือ service discovery)

<div style="margin-left:3rem">
    <img src="./images/zookeeper.png" alt="zookeeper" width="500" />
</div>

ด้วยการเปลี่ยนแปลงนี้ broker จะต้องดูแลรักษาเพียงข้อมูลของข้อความเท่านั้น ส่วน metadata และ state storage จะอยู่ใน Zookeeper

Zookeeper ยังช่วยในการเลือกตั้ง leader (leader election) ของ broker replica อีกด้วย

### **การทำสำเนา (Replication)**

ในระบบกระจาย ปัญหาด้านฮาร์ดแวร์เป็นสิ่งที่หลีกเลี่ยงไม่ได้ เราสามารถรับมือกับปัญหานี้ผ่านการทำ replication เพื่อให้ได้ความพร้อมใช้งานสูง (high availability)

<div style="margin-left:3rem">
    <img src="./images/replication-example.png" alt="replication-example" width="500" />
</div>

 * แต่ละ partition จะถูกทำสำเนาไปยัง broker หลายตัว แต่จะมี leader replica เพียงตัวเดียว
 * Producer ส่งข้อความไปยัง leader replica
 * Follower ดึงข้อความที่ถูกทำสำเนาจาก leader
 * เมื่อ replica เพียงพอถูกซิงค์แล้ว leader จะส่งการยืนยัน (acknowledgment) กลับไปยัง producer
 * การกระจาย replica สำหรับแต่ละ partition เรียกว่า replica distribution plan
 * leader ของ partition หนึ่ง ๆ เป็นผู้สร้าง replica distribution plan และบันทึกไว้ใน Zookeeper

### **In-sync Replica (ISR)**

ปัญหาหนึ่งที่เราต้องรับมือคือการรักษาข้อความให้ตรงกัน (in-sync) ระหว่าง leader และ follower สำหรับ partition หนึ่ง ๆ

In-sync replica (ISR) คือ replica ของ partition ที่ยังคงซิงค์อยู่กับ leader

`replica.lag.max.messages` กำหนดว่า replica สามารถล้าหลัง (lag) leader ได้กี่ข้อความจึงจะยังถือว่าอยู่ใน in-sync

<div style="margin-left:3rem">
    <img src="./images/in-sync-replicas-example.png" alt="in-sync-replicas-example" width="500" />
</div>

 * offset ที่ถูก commit อยู่ที่ 13
 * มีข้อความใหม่สองข้อความถูกเขียนไปยัง leader แต่ยังไม่ถูก commit
 * ข้อความหนึ่งจะถูก commit เมื่อ replica ทั้งหมดใน ISR ซิงค์ข้อความนั้นเรียบร้อยแล้ว
 * Replica 2 และ 3 ตามทัน leader เต็มที่แล้ว จึงอยู่ใน ISR
 * Replica 4 ล้าหลัง จึงถูกนำออกจาก ISR ชั่วคราว

ISR สะท้อนให้เห็น trade-off ระหว่างประสิทธิภาพและความคงทน (durability)
 * เพื่อไม่ให้ producer สูญเสียข้อความ replica ทั้งหมดควรซิงค์กันก่อนที่จะส่ง acknowledgment
 * แต่ replica ที่ช้าจะทำให้ partition ทั้งหมดไม่พร้อมใช้งาน (unavailable)

การจัดการ acknowledgment สามารถปรับตั้งค่าได้

`ACK=all` หมายความว่า replica ทั้งหมดใน ISR ต้องซิงค์ข้อความนั้นก่อน การส่งข้อความจะช้า แต่ความคงทนของข้อความสูงที่สุด

<div style="margin-left:3rem">
    <img src="./images/ack-all.png" alt="ack-all" width="500" />
</div>

`ACK=1` หมายความว่า producer จะได้รับ acknowledgment เมื่อ leader ได้รับข้อความแล้ว การส่งข้อความเร็ว แต่ความคงทนของข้อความต่ำ

<div style="margin-left:3rem">
    <img src="./images/ack-1.png" alt="ack-1" width="500" />
</div>

`ACK=0` หมายความว่า producer ส่งข้อความโดยไม่รอ acknowledgment ใด ๆ จาก leader การส่งข้อความเร็วที่สุด แต่ความคงทนของข้อความต่ำที่สุด

<div style="margin-left:3rem">
    <img src="./images/ack-0.png" alt="ack-0" width="500" />
</div>

ในฝั่ง consumer เราสามารถเชื่อมต่อ consumer ทั้งหมดเข้ากับ leader ของ partition และให้พวกมันอ่านข้อความจาก leader ได้:
 * นี่เป็นการออกแบบที่เรียบง่ายที่สุดและดำเนินการง่ายที่สุด
 * ข้อความใน partition จะถูกส่งไปยัง consumer เพียงตัวเดียวในกลุ่ม ซึ่งจำกัดจำนวนการเชื่อมต่อไปยัง leader replica
 * จำนวนการเชื่อมต่อไปยัง leader replica มักไม่สูงนัก ตราบใดที่ topic นั้นไม่ได้รับความนิยมสูงมาก (ไม่ "hot")
 * เราสามารถขยาย topic ที่ hot ได้โดยการเพิ่มจำนวน partition และ consumer
 * ในบางสถานการณ์ อาจเหมาะสมกว่าที่จะให้ consumer อ่านจาก ISR ตัวหนึ่งเป็นหลัก เช่น หากอยู่คนละ data center

รายการ ISR ถูกดูแลรักษาโดย leader ซึ่งจะติดตามความล้าหลัง (lag) ระหว่างตัวมันเองกับ replica แต่ละตัว

### **ความสามารถในการขยายระบบ (Scalability)**

มาประเมินกันว่าเราสามารถขยายแต่ละส่วนของระบบได้อย่างไร

#### Producer

Producer มีขนาดเล็กกว่า consumer มาก ความสามารถในการขยายของมันสามารถทำได้ง่ายโดยการเพิ่ม/ลบ instance ของ producer

#### Consumer

Consumer group แยกจากกันโดยสิ้นเชิง สามารถเพิ่ม/ลบ consumer group ได้ตามต้องการอย่างง่ายดาย

การ rebalancing ช่วยจัดการกรณีที่ consumer ถูกเพิ่ม/ลบออกจากกลุ่มได้อย่างราบรื่น

Consumer group และการ rebalancing ช่วยให้เราบรรลุทั้งความสามารถในการขยายระบบและการทนต่อความล้มเหลว

#### Broker

Broker จัดการกับความล้มเหลวอย่างไร?

<div style="margin-left:3rem">
    <img src="./images/broker-failure-recovery.png" alt="broker-failure-recovery" width="500" />
</div>

 * เมื่อ broker หนึ่งล่ม ยังคงมี replica เพียงพอที่จะป้องกันไม่ให้ข้อมูลของ partition สูญหาย
 * leader ตัวใหม่จะถูกเลือกตั้ง และ broker coordinator จะกระจาย partition ที่เคยอยู่ที่ broker ที่ล่มไปยัง replica ที่เหลืออยู่
 * replica ที่เหลืออยู่จะรับ partition ใหม่และทำหน้าที่เป็น follower จนกว่าจะตามทัน leader และกลายเป็น ISR

ข้อพิจารณาเพิ่มเติมเพื่อให้ broker ทนต่อความล้มเหลวได้:
 * จำนวน ISR ขั้นต่ำเป็นตัวสร้างสมดุลระหว่าง latency และความปลอดภัย สามารถปรับจูนได้ตามความต้องการ
 * หาก replica ทั้งหมดของ partition หนึ่งอยู่บนโหนดเดียวกัน จะเป็นการสิ้นเปลืองทรัพยากร replica ควรกระจายอยู่บน broker ที่แตกต่างกัน
 * หาก replica ทั้งหมดของ partition ล่มพร้อมกัน ข้อมูลจะสูญหายไปตลอดกาล การกระจาย replica ไปยังหลาย data center ช่วยได้ แต่จะเพิ่ม latency มาก ทางเลือกหนึ่งคือใช้ [data mirroring](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=27846330) เป็นวิธีแก้ปัญหาทางอ้อม

เราจัดการกับการกระจาย replica ใหม่อย่างไรเมื่อมีการเพิ่ม broker ตัวใหม่?

<div style="margin-left:3rem">
    <img src="./images/broker-replica-redistribution.png" alt="broker-replica-redistribution" width="500" />
</div>

 * เราสามารถอนุญาตให้มี replica มากกว่าที่กำหนดไว้ชั่วคราว จนกว่า broker ใหม่จะตามทัน
 * เมื่อตามทันแล้ว เราสามารถลบ replica ของ partition ที่ไม่จำเป็นอีกต่อไปได้

#### Partition

เมื่อใดก็ตามที่มีการเพิ่ม partition ใหม่ producer จะได้รับการแจ้งเตือน และ consumer rebalancing จะถูกกระตุ้น

ในแง่ของการจัดเก็บข้อมูล เราสามารถจัดเก็บเฉพาะข้อความใหม่ลงใน partition ใหม่ได้ แทนที่จะพยายามคัดลอกข้อความเก่าทั้งหมด:

<div style="margin-left:3rem">
    <img src="./images/partition-exmaple.png" alt="partition-example" width="500" />
</div>

การลดจำนวน partition มีความซับซ้อนกว่า:

<div style="margin-left:3rem">
    <img src="./images/partition-decrease.png" alt="partition-decrease" width="500" />
</div>

 * เมื่อ partition หนึ่งถูกปลดระวาง (decommission) ข้อความใหม่จะถูกรับโดย partition ที่เหลืออยู่เท่านั้น
 * partition ที่ถูกปลดระวางจะไม่ถูกลบออกในทันที เนื่องจากยังสามารถบริโภคข้อความจากมันได้อยู่
 * เมื่อระยะเวลาการเก็บรักษาข้อมูล (retention period) ที่กำหนดไว้ล่วงหน้าผ่านไปแล้ว ข้อมูลจะถูกตัดทอน (truncate) และพื้นที่จัดเก็บจะถูกปลดปล่อย
 * ในช่วงเปลี่ยนผ่าน producer จะส่งข้อความไปยัง partition ที่ยัง active เท่านั้น แต่ consumer จะอ่านจากทุก partition
 * เมื่อ retention period หมดอายุลง consumer จะถูก rebalance

### **Data Delivery Semantics**

มาพูดคุยกันถึง delivery semantic ที่แตกต่างกัน

#### At-most Once

ด้วยการรับประกันนี้ ข้อความจะถูกส่งมอบไม่เกินหนึ่งครั้ง และอาจไม่ถูกส่งมอบเลยก็ได้

<div style="margin-left:3rem">
    <img src="./images/at-most-once.png" alt="at-most-once" width="500" />
</div>

 * Producer ส่งข้อความแบบ asynchronous ไปยัง topic หากการส่งข้อความล้มเหลว จะไม่มีการลองใหม่ (retry)
 * Consumer ดึงข้อความและ commit offset ทันที หาก consumer ล่มก่อนที่จะประมวลผลข้อความ ข้อความนั้นจะไม่ถูกประมวลผลเลย

#### At-least Once

ข้อความหนึ่งอาจถูกส่งมากกว่าหนึ่งครั้ง และไม่ควรมีข้อความใดถูกทิ้งไว้โดยไม่ถูกประมวลผล

<div style="margin-left:3rem">
    <img src="./images/at-least-once.png" alt="at-least-once" width="500" />
</div>

 * Producer ส่งข้อความด้วย `ack=1` หรือ `ack=all` หากมีปัญหาใด ๆ จะทำการลองใหม่ (retry) ต่อไปเรื่อย ๆ
 * Consumer ดึงข้อความและ commit offset ก็ต่อเมื่อประมวลผลข้อความนั้นเสร็จสิ้นแล้ว
 * มีความเป็นไปได้ที่ข้อความจะถูกส่งมอบมากกว่าหนึ่งครั้ง เช่น หาก consumer ล่มหลังจากประมวลผลข้อความแล้วแต่ก่อน commit offset
 * ด้วยเหตุนี้ วิธีนี้จึงเหมาะกับกรณีการใช้งานที่การเกิดข้อมูลซ้ำ (duplication) เป็นสิ่งที่ยอมรับได้ หรือสามารถทำการลบข้อมูลซ้ำ (deduplication) ได้

#### Exactly Once

มีต้นทุนสูงมากในการนำไปใช้งานจริงกับระบบ แม้ว่าจะเป็นการรับประกันที่เป็นมิตรต่อผู้ใช้มากที่สุด:

<div style="margin-left:3rem">
    <img src="./images/exactly-once.png" alt="exactly-once" width="500" />
</div>

### **ฟีเจอร์ขั้นสูง (Advanced features)**

มาพูดคุยกันถึงฟีเจอร์ขั้นสูงบางอย่างที่อาจถูกพูดถึงในการสัมภาษณ์

#### การกรองข้อความ (Message filtering)

Consumer บางตัวอาจต้องการบริโภคเฉพาะข้อความประเภทหนึ่งภายใน partition เท่านั้น

สิ่งนี้สามารถทำได้โดยการสร้าง topic แยกกันสำหรับข้อความแต่ละกลุ่มย่อย แต่วิธีนี้อาจมีต้นทุนสูงหากระบบมีกรณีการใช้งานที่แตกต่างกันมากเกินไป
 * เป็นการสิ้นเปลืองทรัพยากรที่จะจัดเก็บข้อความเดียวกันไว้ใน topic ที่ต่างกัน
 * Producer จะถูกผูกติดอย่างแน่นหนา (tightly coupled) กับ consumer มากขึ้น เนื่องจากต้องเปลี่ยนแปลงตามความต้องการของ consumer แต่ละตัว

เราสามารถแก้ปัญหานี้ได้โดยใช้การกรองข้อความ (message filtering)
 * แนวทางง่าย ๆ คือทำการกรองที่ฝั่ง consumer แต่จะทำให้เกิดทราฟฟิกของ consumer ที่ไม่จำเป็น
 * อีกทางเลือกหนึ่งคือ ข้อความสามารถมีแท็ก (tag) แนบอยู่ และ consumer สามารถระบุว่าตนสมัครสมาชิกแท็กใดบ้าง
 * การกรองยังสามารถทำผ่าน payload ของข้อความได้ แต่จะมีความท้าทายและไม่ปลอดภัยหากข้อความถูกเข้ารหัส/serialize ไว้
 * สำหรับสูตรทางคณิตศาสตร์ที่ซับซ้อนมากขึ้น broker อาจนำ grammar parser หรือ script executor มาใช้ แต่จะทำให้ message queue มีความหนักเกินไป

<div style="margin-left:3rem">
    <img src="./images/message-filtering.png" alt="message-filtering" width="500" />
</div>

#### ข้อความหน่วงเวลาและข้อความตามกำหนดเวลา (Delayed messages & scheduled messages)

สำหรับบางกรณีการใช้งาน เราอาจต้องการหน่วงเวลาหรือกำหนดเวลาการส่งมอบข้อความ
ตัวอย่างเช่น เราอาจส่งคำขอตรวจสอบการชำระเงินที่จะทำงานอีก 30 นาทีข้างหน้า ซึ่งจะกระตุ้นให้ consumer ตรวจสอบว่าการชำระเงินสำเร็จหรือไม่

สิ่งนี้สามารถทำได้โดยการส่งข้อความไปยังที่เก็บข้อมูลชั่วคราว (temporary storage) ใน broker และย้ายข้อความไปยัง partition ในเวลาที่เหมาะสม:

<div style="margin-left:3rem">
    <img src="./images/delayed-message-implementation.png" alt="delayed-message-implementation" width="500" />
</div>

 * ที่เก็บข้อมูลชั่วคราวสามารถเป็น topic ข้อความพิเศษหนึ่งตัวหรือมากกว่า
 * ฟังก์ชันจับเวลาสามารถทำได้โดยใช้ delay queue เฉพาะทาง หรือ [hierarchical time wheel](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf)

---

## ขั้นตอนที่ 4: สรุปจบ

ประเด็นเพิ่มเติมที่สามารถพูดคุยได้:
- **โปรโตคอลการสื่อสาร (Protocol of communication):** ข้อพิจารณาที่สำคัญ - รองรับทุกกรณีการใช้งานและปริมาณข้อมูลสูง รวมถึงตรวจสอบความถูกต้องของข้อความ โปรโตคอลที่นิยม ได้แก่ AMQP และ Kafka protocol
- **การบริโภคซ้ำ (Retry consumption):** หากเราไม่สามารถประมวลผลข้อความได้ทันที เราสามารถส่งไปยัง topic สำหรับการลองใหม่ (retry) โดยเฉพาะ เพื่อลองประมวลผลอีกครั้งในภายหลัง
- **การเก็บถาวรข้อมูลย้อนหลัง (Historical data archive):** ข้อความเก่าสามารถถูกสำรองไว้ในที่จัดเก็บข้อมูลขนาดใหญ่ เช่น HDFS หรือ object storage (เช่น S3)
