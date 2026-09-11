# บทที่ 12: ออกแบบระบบแชท (Design a Chat System)

## บทนำ
**ระบบแชท (chat system)** รองรับการส่งข้อความแบบ real-time ระหว่างผู้ใช้ บทนี้เน้นการออกแบบแอปแชทที่ประกอบด้วย:
- **การแชทแบบตัวต่อตัว (One-on-One Chat)**
- **การแชทแบบกลุ่ม (Group Chat) (สูงสุด 100 คน)**
- **ตัวบ่งชี้สถานะออนไลน์ (Online Presence Indicators)**
- **รองรับหลายอุปกรณ์ (Multiple Device Support)**
- **Push Notification**

ระบบนี้ตั้งเป้าไว้ที่ผู้ใช้งานที่ใช้งานต่อวัน (**DAU**) จำนวน **50 ล้านคน** และจัดเก็บประวัติการแชทไว้อย่างถาวร

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)

### ข้อกำหนด (Requirements)
1. **ฟีเจอร์:**
   - การแชทแบบตัวต่อตัวและแบบกลุ่ม (สูงสุด 100 คน)
   - ข้อความแบบตัวอักษร (สูงสุด 100,000 ตัวอักษร)
   - ตัวบ่งชี้สถานะออนไลน์/ออฟไลน์
   - รองรับหลายอุปกรณ์
   - Push notification
2. **สเกล:** ออกแบบสำหรับ DAU จำนวน 50 ล้านคน
3. **การจัดเก็บ (Storage):** ประวัติการแชทถูกจัดเก็บอย่างถาวร

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)

### โปรโตคอลการสื่อสาร (Communication Protocols)
1. **ฝั่งผู้ส่ง (Sender Side):** ใช้ HTTP ในการส่งข้อความ โดยใช้ประโยชน์จาก persistent connection เพื่อความมีประสิทธิภาพ

      <div style="margin-left:2rem">
      <img src="./images/basic-design.png" alt="Basic Design" width="500">
      <div>

2. **ฝั่งผู้รับ (Receiver Side):**
   - **Polling:**
      - ไคลเอนต์จะถามเซิร์ฟเวอร์เป็นระยะ ๆ ว่ามีข้อความใหม่หรือไม่
      - ไม่มีประสิทธิภาพ เนื่องจากมีการส่ง request ซ้ำซ้อนบ่อยเกินไป

         <img src="./images/polling.png" alt="Polling" width="400">

   - **Long Polling:**
      - คงการเชื่อมต่อไว้จนกว่าจะมีข้อความมาถึง
      - ไม่มีประสิทธิภาพสำหรับผู้ใช้ที่ไม่ได้ใช้งาน (inactive)

         <img src="./images/long-polling.png" alt="Long Polling" width="400">

   - **WebSocket:**
      - เป็นการเชื่อมต่อแบบสองทิศทาง (bi-directional) และคงอยู่ตลอด (persistent) สำหรับการสื่อสารแบบ real-time ถูกเลือกใช้ทั้งสำหรับการส่งและรับข้อความ
      - ใช้โปรโตคอล WebSocket (ws) สำหรับการส่งและรับข้อความ

         <img src="./images/websocket.png" alt="Websocket"  width="400" >

---

### คอมโพเนนต์ (Components)

<div style="margin-left:5rem">
   <img src="./images/high-level-stateless-arch.png" alt="High Level Architecture" height="350">
   <img src="./images/high-level-statefull-arch.png" alt="High Level Architecture" height="350" width="550">
</div>

1. **บริการแบบ Stateless (Stateless Services):**
   - จัดการเรื่องการสมัครสมาชิก, การล็อกอิน, และการจัดการโปรไฟล์ผู้ใช้
   - เชื่อมต่อกับ service discovery เพื่อแนะนำ chat server ที่เหมาะสมที่สุด
2. **บริการแบบ Stateful (Stateful Services):**
   - Chat server รักษาการเชื่อมต่อแบบ WebSocket ที่คงอยู่ตลอด (persistent)
   - รับผิดชอบการนำส่งและซิงค์ข้อความ
3. **การเชื่อมต่อกับบริการภายนอก (Third-Party Integration):**
   - บริการ push notification แจ้งเตือนผู้ใช้เมื่อมีข้อความใหม่
   - ดูรายละเอียดการ implement การแจ้งเตือนได้ในบท Notification System


---
### การออกแบบ (Design)

ไคลเอนต์จะคงการเชื่อมต่อ WebSocket ไว้กับ chat server อย่างต่อเนื่องเพื่อรองรับการส่งข้อความแบบ real-time

<div style="margin-left:3rem">
      <img src="./images/high-level-design.png" alt="High Level Design" width="450">
</div>

- Chat server ช่วยจัดการการส่ง/รับข้อความ
- Presence server จัดการสถานะออนไลน์/ออฟไลน์
- API server จัดการทุกอย่างรวมถึงการล็อกอินของผู้ใช้, การสมัครสมาชิก, การเปลี่ยนโปรไฟล์ ฯลฯ
- Notification server ส่ง push notification
- สุดท้าย key-value store ถูกใช้เพื่อจัดเก็บประวัติการแชท เหตุผลที่เลือกใช้ key-value store สำหรับฐานข้อมูลประวัติการแชท มีดังนี้:
   - รองรับการทำ horizontal scaling ได้ง่าย
   - key-value store มี latency ต่ำมากในการเข้าถึงข้อมูล
   - ฐานข้อมูลเชิงสัมพันธ์ (relational database) จัดการกับข้อมูลส่วนหางยาว (long tail) ได้ไม่ดีนัก เมื่อ index มีขนาดใหญ่ขึ้น การเข้าถึงข้อมูลแบบสุ่ม (random access) จะมีค่าใช้จ่ายสูง
   - key-value store ถูกนำไปใช้โดยแอปพลิเคชันแชทที่มีความน่าเชื่อถือและได้รับการพิสูจน์แล้ว ตัวอย่างเช่น ทั้ง Facebook Messenger และ Discord


ต่อไปนี้คือ data model สำหรับการแชทแบบตัวต่อตัวและการแชทแบบกลุ่ม
   - primary key คือ message ID ซึ่งช่วยกำหนดลำดับของข้อความ
   - สำหรับการแชทแบบกลุ่ม primary key แบบผสม (composite) คือ (channel_id, message_id)
      - ID สามารถสร้างได้โดยใช้ global 64-bit sequence number generator เช่น Snowflake
      - แนวทางที่ดีกว่าคือการใช้ local sequence number generator คำว่า local หมายถึง ID มีความไม่ซ้ำกันเฉพาะภายในกลุ่มนั้น ๆ เท่านั้น
      - เหตุผลที่ local ID ใช้งานได้ผลดี คือ การรักษาลำดับของข้อความภายในช่องแชทแบบตัวต่อตัวหรือช่องแชทแบบกลุ่มเพียงช่องเดียวก็เพียงพอแล้ว

      <img src="./images/one-to-one-chat.png" alt="One to one chat design" width="300">
      <img src="./images/group-chat.png" alt="Group chat design" width="300">


## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### Service Discovery

<div style="margin-left:3rem">
   <img src="./images/zookeeper.png" alt="Zookeeper" width="400">
</div>

- บทบาทหลักของ service discovery คือการแนะนำ chat server ที่ดีที่สุดให้กับไคลเอนต์ โดยพิจารณาจากเกณฑ์ต่าง ๆ เช่น ตำแหน่งทางภูมิศาสตร์ และความจุของเซิร์ฟเวอร์
- ใช้ **Apache Zookeeper** เพื่อจัดสรร chat server ตามเกณฑ์ต่าง ๆ เช่น ตำแหน่งทางภูมิศาสตร์และความจุของเซิร์ฟเวอร์
- ช่วยให้การกระจายโหลดมีประสิทธิภาพและลด latency ให้น้อยที่สุด


### ลำดับการทำงานของการส่งข้อความ (Messaging Flows)
#### การแชทแบบตัวต่อตัว (One-on-One Chat)


1. ผู้ใช้ A ส่งข้อความไปยัง Chat Server 1
2. Chat Server 1 กำหนด message ID ที่ไม่ซ้ำกันและจัดเก็บข้อความไว้ใน key-value store
3. หากผู้ใช้ B ออนไลน์อยู่ ข้อความจะถูกส่งต่อไปยัง Chat Server 2 โดยรักษาการเชื่อมต่อ WebSocket ที่คงอยู่ตลอดไว้
4. หากผู้ใช้ B ออฟไลน์อยู่ ระบบจะส่ง push notification



#### การแชทแบบกลุ่ม (Group Chat)

<div style="margin-left:3rem">
   <img src="./images/group-chat-flow.png" alt="Group Chat Flow" width="400">
</div>

- ข้อความจะถูกคัดลอกไปยัง inbox ของผู้รับแต่ละคนในกลุ่ม
- ทำให้การซิงค์ข้อมูลง่ายขึ้น แต่จะมีค่าใช้จ่ายสูงขึ้นสำหรับกลุ่มขนาดใหญ่
- ในฝั่งผู้รับ ผู้รับหนึ่งคนสามารถได้รับข้อความจากผู้ส่งหลายคน ผู้รับแต่ละคนจะมี inbox (message sync queue) ซึ่งเก็บข้อความจากผู้ส่งที่แตกต่างกัน

---

#### การซิงค์ข้อความ (Message Synchronization)

ผู้ใช้หลายคนมีอุปกรณ์หลายเครื่อง เราจำเป็นต้องซิงค์ข้อความข้ามอุปกรณ์เหล่านั้น อุปกรณ์แต่ละเครื่องจะเก็บตัวแปรที่เรียกว่า cur_max_message_id ซึ่งใช้ติดตาม message ID ล่าสุดบนอุปกรณ์นั้น ข้อความที่ตรงกับเงื่อนไขสองข้อต่อไปนี้จะถือว่าเป็นข้อความใหม่:

<div style="margin-left:3rem">
   <img src="./images/message-synchronization.png" alt="Message Synchronization"  width="400">
</div>

- recipient ID ตรงกับ ID ของผู้ใช้ที่กำลังล็อกอินอยู่ในขณะนั้น
- message ID ใน key-value store มีค่ามากกว่า cur_max_message_id

---

### สถานะออนไลน์ (Online Presence)
1. **กลไก Heartbeat (Heartbeat Mechanism):**
   <div style="margin-left:3rem">
      <img src="./images/heartbeat-mechanism.png" alt="Heartbeat Mechanism" width="400">
   </div>

   - ไคลเอนต์จะส่ง heartbeat เป็นระยะ ๆ ไปยัง presence server เพื่อบ่งบอกว่ากำลังออนไลน์อยู่
   - หากไม่ได้รับ heartbeat ภายในเกณฑ์ที่กำหนด (เช่น x = 30) ผู้ใช้จะถูกทำเครื่องหมายว่าออฟไลน์



2. **โมเดล Fanout (Fanout Model):**

   <div style="margin-left:3rem">
      <img src="./images/fanout-presence.png" alt="Fanout Presence" width="400">
   </div>

   - การอัปเดตสถานะออนไลน์จะถูก push ไปยังเพื่อน ๆ โดยใช้โมเดล publish-subscribe ซึ่งแต่ละคู่เพื่อนจะมี channel ของตัวเอง
   - เมื่อสถานะออนไลน์ของผู้ใช้ A เปลี่ยนไป จะทำการ publish event ไปยังสาม channel คือ channel A-B, A-C, และ A-D
   - สาม channel นี้ถูก subscribe โดยผู้ใช้ B, C, และ D ตามลำดับ ซึ่งจะได้รับการอัปเดตสถานะออนไลน์
   - การออกแบบนี้มีประสิทธิภาพสำหรับกลุ่มผู้ใช้ขนาดเล็ก


---

## ประเด็นเพิ่มเติมที่ต้องพิจารณา (Additional Considerations)
### ความสามารถในการขยายตัว (Scalability)
- **การขยายแนวนอน (Horizontal Scaling):** เพิ่มเซิร์ฟเวอร์เมื่อจำนวนผู้ใช้เพิ่มขึ้น
- **Load Balancing:** กระจายทราฟฟิกไปยังเซิร์ฟเวอร์ต่าง ๆ อย่างสม่ำเสมอ
- **การแคช (Caching):** ลดภาระของฐานข้อมูลและปรับปรุง latency ให้ดีขึ้น

### การจัดการข้อผิดพลาด (Error Handling)
- **กลไก Retry (Retry Mechanisms):** จัดการกับความล้มเหลวในการนำส่งข้อความด้วยการ retry และการต่อคิว (queuing)
- **ความล้มเหลวของเซิร์ฟเวอร์ (Server Failures):** ใช้ service discovery เพื่อจัดสรรเซิร์ฟเวอร์ใหม่ในกรณีที่เกิดความล้มเหลว

### แนวทางการต่อยอดในอนาคต (Future Extensions)
1. **การรองรับสื่อ (Media Support):** เพิ่มการจัดการรูปภาพและวิดีโอ รวมถึงการบีบอัด (compression) และการจัดเก็บบนคลาวด์
2. **การเข้ารหัสแบบ End-to-End (End-to-End Encryption):** รักษาความเป็นส่วนตัวของข้อความ
3. **การแคชฝั่งไคลเอนต์ (Client-Side Caching):** ลดปริมาณการส่งข้อมูลเพื่อประสิทธิภาพที่ดีขึ้น
4. **การปรับปรุงเวลาในการโหลด (Improved Load Times):** ใช้เครือข่ายแคชที่กระจายอยู่ตามพื้นที่ทางภูมิศาสตร์
