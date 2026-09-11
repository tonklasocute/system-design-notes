# บทที่ 17: เพื่อนที่อยู่ใกล้เคียง (Nearby Friends)

## บทนำ

บทนี้มุ่งเน้นไปที่การออกแบบ backend ที่สามารถขยายตัวได้ สำหรับแอปพลิเคชันที่ช่วยให้ผู้ใช้สามารถแชร์ตำแหน่งที่อยู่ของตนเอง และค้นหาเพื่อนที่อยู่ **ใกล้เคียง (nearby)**

ความแตกต่างหลักกับบทเรื่อง proximity คือ ในโจทย์นี้ **ตำแหน่งที่อยู่มีการเปลี่ยนแปลงตลอดเวลา (locations constantly change)** ในขณะที่บทนั้นตำแหน่งที่อยู่ของธุรกิจแทบจะไม่เปลี่ยนแปลง

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)

คำถามบางส่วนที่ใช้ในการนำสัมภาษณ์:
 * C: ระยะทางทางภูมิศาสตร์เท่าใดจึงจะถือว่า "ใกล้เคียง"
 * I: 5 ไมล์ ตัวเลขนี้ควรปรับตั้งค่าได้ (configurable)
 * C: ระยะทางคำนวณเป็นเส้นตรง (straight-line distance) หรือคำนึงถึงสิ่งกีดขวาง เช่น แม่น้ำที่กั้นระหว่างเพื่อนด้วย
 * I: ใช่ นั่นเป็นสมมติฐานที่สมเหตุสมผล
 * C: แอปนี้มีผู้ใช้กี่คน
 * I: 1 พันล้านคน และ 10% ของผู้ใช้เหล่านั้นใช้ฟีเจอร์เพื่อนที่อยู่ใกล้เคียง
 * C: เราจำเป็นต้องเก็บประวัติตำแหน่งที่อยู่ (location history) หรือไม่
 * I: ใช่ มันมีคุณค่า เช่น สำหรับ machine learning
 * C: เราสามารถสมมติได้หรือไม่ว่าเพื่อนที่ไม่มีความเคลื่อนไหว (inactive) จะหายไปจากฟีเจอร์นี้ภายใน 10 นาที
 * I: ได้
 * C: เราต้องกังวลเรื่อง GDPR หรือเรื่องอื่น ๆ ในทำนองนี้หรือไม่
 * I: ไม่ต้อง เพื่อความง่าย

### **ความต้องการเชิงฟังก์ชัน (Functional requirements)**

 * ผู้ใช้ควรสามารถเห็นเพื่อนที่อยู่ใกล้เคียงบนแอปมือถือของตนได้ เพื่อนแต่ละคนจะมีระยะทางและ timestamp กำกับไว้ ซึ่งบ่งบอกว่าตำแหน่งนั้นถูกอัปเดตเมื่อใด
 * รายชื่อเพื่อนที่อยู่ใกล้เคียงควรถูกอัปเดตทุก ๆ ไม่กี่วินาที

### **ความต้องการที่ไม่ใช่เชิงฟังก์ชัน (Non-functional requirements)**

- **Low latency**: สำคัญมากที่จะได้รับการอัปเดตตำแหน่งโดยไม่มีความล่าช้ามากเกินไป
- **Reliability**: การสูญเสียข้อมูลบางจุดเป็นครั้งคราวเป็นสิ่งที่ยอมรับได้ แต่ระบบโดยรวมต้องพร้อมใช้งาน (available) เสมอ
- **Eventual consistency**: ที่เก็บข้อมูลตำแหน่ง (location data store) ไม่จำเป็นต้องมี strong consistency ความล่าช้าไม่กี่วินาทีในการรับข้อมูลตำแหน่งที่ replica ต่าง ๆ เป็นสิ่งที่ยอมรับได้

### **การประมาณคร่าว ๆ (Back-of-the-envelope)**

การประมาณการบางส่วนเพื่อกำหนดขนาด (scale) ที่เป็นไปได้:
 * เพื่อนที่อยู่ใกล้เคียง คือเพื่อนที่อยู่ในรัศมี 5 ไมล์
 * ช่วงเวลาการรีเฟรชตำแหน่ง (location refresh interval) คือ 30 วินาที เนื่องจากความเร็วในการเดินของมนุษย์นั้นช้า จึงไม่จำเป็นต้องอัปเดตตำแหน่งบ่อยเกินไป
 * โดยเฉลี่ยแล้ว มีผู้ใช้ 100 ล้านคนใช้ฟีเจอร์นี้ทุกวัน โดยมี concurrent users 10% หรือคิดเป็น 10 ล้านคน
 * โดยเฉลี่ยแล้ว ผู้ใช้หนึ่งคนมีเพื่อน 400 คน และเพื่อนทุกคนใช้ฟีเจอร์เพื่อนที่อยู่ใกล้เคียงนี้
 * แอปแสดงเพื่อนที่อยู่ใกล้เคียง 20 คนต่อหนึ่งหน้า
 * **Location Update QPS** = 10 ล้าน / 30 == ~334k การอัปเดตต่อวินาที

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

ก่อนจะเจาะลึกเรื่องการออกแบบ API และ data model เราจะศึกษาโปรโตคอลการสื่อสารที่จะใช้ก่อน เนื่องจากมันไม่ได้เป็นที่แพร่หลายทั่วไปเหมือนโมเดลการสื่อสารแบบ request-response แบบดั้งเดิม

### **การออกแบบระดับสูง (High-level design)**

ในระดับสูง เราต้องการสร้างการส่งข้อความ (message passing) ที่มีประสิทธิภาพระหว่าง peer ด้วยกัน สิ่งนี้ทำได้ผ่านโปรโตคอลแบบ peer-to-peer แต่วิธีนี้ไม่เหมาะกับการใช้งานจริงสำหรับแอปมือถือที่มีการเชื่อมต่อไม่เสถียร (flaky connection) และมีข้อจำกัดด้านการใช้พลังงานที่เข้มงวด

แนวทางที่ใช้งานได้จริงมากกว่าคือ การใช้ backend ที่ใช้งานร่วมกัน (shared backend) เป็นกลไก fan-out ไปยังเพื่อนที่ต้องการติดต่อ:

<div style="margin-left:3rem">
    <img src="./images/fan-out-backend.png" alt="fan-out-backend" width="500" />
</div>

Backend ทำหน้าที่อะไรบ้าง?
 * รับการอัปเดตตำแหน่งจากผู้ใช้ที่ active ทั้งหมด
 * สำหรับการอัปเดตตำแหน่งแต่ละครั้ง หาผู้ใช้ที่ active ทั้งหมดที่ควรได้รับข้อมูลนั้น แล้วส่งต่อไปให้
 * ไม่ส่งต่อข้อมูลตำแหน่ง หากระยะห่างระหว่างเพื่อนเกินกว่าค่า threshold ที่ตั้งไว้

ฟังดูเรียบง่าย แต่ความท้าทายคือการออกแบบระบบให้รองรับ scale ที่เรากำลังทำงานอยู่นี้

เราจะเริ่มต้นด้วยการออกแบบที่เรียบง่ายกว่าก่อน แล้วค่อยพูดถึงแนวทางที่ก้าวหน้ากว่าใน deep dive:

<div style="margin-left:3rem">
    <img src="./images/simple-high-level-design.png" alt="simple-high-level-design" width="500" />
</div>

- **Load balancer**: กระจายทราฟฟิกไปยัง rest API server และ bidirectional web socket server
- **Rest API servers**: จัดการงานเสริม (auxiliary tasks) เช่น การจัดการเพื่อน การอัปเดตโปรไฟล์ เป็นต้น
- **Websocket servers**: server แบบ stateful ที่ทำหน้าที่ส่งต่อ request การอัปเดตตำแหน่งไปยัง client ที่เกี่ยวข้อง นอกจากนี้ยังจัดการการ seed ข้อมูลตำแหน่งของเพื่อนที่อยู่ใกล้เคียงให้กับ mobile client ตอนเริ่มต้น (initialization) ด้วย (จะกล่าวถึงในรายละเอียดในภายหลัง)
- **Redis location cache**: ใช้เก็บข้อมูลตำแหน่งล่าสุดของผู้ใช้ที่ active แต่ละคน มีการตั้งค่า TTL ไว้ในแต่ละ entry ของ cache เมื่อ TTL หมดอายุ แสดงว่าผู้ใช้นั้นไม่ active อีกต่อไป และข้อมูลของเขาจะถูกลบออกจาก cache
- **User database**: เก็บข้อมูลผู้ใช้และความสัมพันธ์ระหว่างเพื่อน (friendship data) สามารถใช้ฐานข้อมูลเชิงสัมพันธ์ (relational) หรือ NoSQL ก็ได้เพื่อจุดประสงค์นี้
- **Location history database**: เก็บประวัติข้อมูลตำแหน่งของผู้ใช้ ไม่จำเป็นต้องใช้โดยตรงในฟีเจอร์เพื่อนที่อยู่ใกล้เคียง แต่ใช้เพื่อติดตามข้อมูลย้อนหลังสำหรับวัตถุประสงค์ในการวิเคราะห์ (analytical purposes)
- **Redis pubsub**: ใช้เป็น message bus ที่มีน้ำหนักเบา (lightweight) ซึ่งเปิด topic แยกสำหรับแต่ละ channel ของผู้ใช้เพื่อรองรับการอัปเดตตำแหน่ง

<div style="margin-left:3rem">
    <img src="./images/redis-pubsub-usage.png" alt="redis-pubsub-usage" width="500" />
</div>

ในตัวอย่างข้างต้น websocket server จะ subscribe เข้ากับ channel ของผู้ใช้ที่เชื่อมต่ออยู่กับตน และส่งต่อการอัปเดตตำแหน่งไปยังผู้ใช้ที่เหมาะสมทุกครั้งที่ได้รับข้อมูล

### **การอัปเดตตำแหน่งเป็นระยะ (Periodic location update)**

ต่อไปนี้คือวิธีการทำงานของ flow การอัปเดตตำแหน่งเป็นระยะ:

<div style="margin-left:3rem">
    <img src="./images/periodic-location-update.png" alt="periodic-location-update" width="500" />
</div>

 * Mobile client ส่งการอัปเดตตำแหน่งไปยัง load balancer
 * Load balancer ส่งต่อการอัปเดตตำแหน่งไปยัง persistent connection ของ websocket server สำหรับ client นั้น
 * Websocket server บันทึกข้อมูลตำแหน่งลงใน location history database
 * ข้อมูลตำแหน่งจะถูกอัปเดตใน location cache นอกจากนี้ websocket server ยังบันทึกข้อมูลตำแหน่งไว้ใน memory เพื่อใช้ในการคำนวณระยะทาง (distance calculations) สำหรับผู้ใช้คนนั้นในครั้งถัดไป
 * Websocket server publish ข้อมูลตำแหน่งไปยัง channel ของผู้ใช้นั้นผ่าน redis pub sub
 * Redis pubsub กระจายการอัปเดตตำแหน่ง (broadcast) ไปยังผู้ subscribe ทั้งหมดของ channel ของผู้ใช้นั้น กล่าวคือ server ที่รับผิดชอบเพื่อนของผู้ใช้คนนั้น
 * Websocket server ที่ subscribe อยู่จะได้รับการอัปเดตตำแหน่ง คำนวณว่าควรส่งข้อมูลนี้ไปยังผู้ใช้คนใดบ้าง แล้วส่งข้อมูลนั้นไป

นี่คือ flow เดียวกันแต่ในเวอร์ชันที่ละเอียดมากขึ้น:

<div style="margin-left:3rem">
    <img src="./images/detailed-periodic-location-update.png" alt="detailed-periodic-location-update" width="500" />
</div>

โดยเฉลี่ยแล้ว จะมีการอัปเดตตำแหน่ง 40 ครั้งที่ต้องส่งต่อ เนื่องจากผู้ใช้หนึ่งคนมีเพื่อนโดยเฉลี่ย 400 คน และ 10% ของเพื่อนเหล่านั้น online อยู่ในช่วงเวลาเดียวกัน

### **การออกแบบ API (API Design)**

Websocket Routine ที่เราต้องรองรับ:
 * การอัปเดตตำแหน่งเป็นระยะ (periodic location update) - ผู้ใช้ส่งข้อมูลตำแหน่งไปยัง websocket server
 * client รับการอัปเดตตำแหน่ง (client receives location update) - server ส่งข้อมูลตำแหน่งและ timestamp ของเพื่อน
 * การเริ่มต้นของ websocket client (websocket client initialization) - client ส่งตำแหน่งของผู้ใช้ไป server ส่งข้อมูลตำแหน่งของเพื่อนที่อยู่ใกล้เคียงกลับมา
 * Subscribe เพื่อนใหม่ (Subscribe to a new friend) - websocket server ส่ง friend ID ที่ mobile client ควร track เช่น เมื่อเพื่อนขึ้น online เป็นครั้งแรก
 * Unsubscribe เพื่อน (Unsubscribe a friend) - websocket server ส่ง friend ID มาให้ mobile client ทำการ unsubscribe เนื่องจากเช่น เพื่อนคนนั้น offline ไปแล้ว

HTTP API - รูปแบบ request/response แบบดั้งเดิม สำหรับความรับผิดชอบที่เป็นงานเสริม (auxiliary responsibilities)

### **Data model**

 * Location cache จะเก็บ mapping ระหว่าง `user_id` กับ `lat,long,timestamp` Redis เป็นตัวเลือกที่ดีมากสำหรับ cache นี้ เนื่องจากเราสนใจแค่ตำแหน่งปัจจุบันเท่านั้น และมันรองรับ TTL eviction ซึ่งเราต้องการสำหรับกรณีการใช้งานนี้
 * Location history table เก็บข้อมูลชุดเดียวกัน แต่อยู่ในรูปแบบตารางเชิงสัมพันธ์ (relational table) ที่มีสี่คอลัมน์ตามที่กล่าวข้างต้น Cassandra สามารถใช้สำหรับข้อมูลนี้ได้ เนื่องจากมันถูก optimize สำหรับงานที่มี write-heavy load

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

มาพูดคุยกันว่าเราจะขยาย (scale) การออกแบบระดับสูงอย่างไร เพื่อให้มันทำงานได้ที่ scale ที่เรากำลังตั้งเป้าไว้

### **แต่ละคอมโพเนนต์สามารถขยายได้ดีเพียงใด (How well does each component scale?)**

- **API servers**: สามารถขยายได้ง่ายด้วย autoscaling groups และการทำสำเนา server instance
- **Websocket servers**: เราสามารถขยาย ws server ออกไปได้ง่าย แต่จำเป็นต้องแน่ใจว่าเราปิด connection ที่มีอยู่เดิมอย่างนุ่มนวล (gracefully) เมื่อ tear down server ตัวหนึ่ง เช่น เราสามารถทำเครื่องหมาย server ว่ากำลัง "draining" ใน load balancer และหยุดส่ง connection ไปที่มัน ก่อนที่จะถูกนำออกจาก server pool ในที่สุด
- **Client initialization**: เมื่อ client เชื่อมต่อกับ server เป็นครั้งแรก มันจะดึงข้อมูลเพื่อนของผู้ใช้ subscribe เข้ากับ channel ของเพื่อนเหล่านั้นบน redis pubsub ดึงตำแหน่งของเพื่อนจาก cache และสุดท้ายส่งต่อให้ client
- **User database**: เราสามารถ shard ฐานข้อมูลนี้โดยใช้ user_id ก็อาจสมเหตุสมผลที่จะเปิด user/friends data ผ่าน service และ API เฉพาะ ซึ่งดูแลโดยทีมที่รับผิดชอบโดยเฉพาะ
- **Location cache**: เราสามารถ shard cache นี้ได้ง่าย ๆ โดยการเปิด redis node หลาย ๆ ตัว นอกจากนี้ TTL ยังช่วยจำกัดปริมาณ memory สูงสุดที่จะถูกใช้ในเวลาใดเวลาหนึ่ง แต่เราก็ยังต้องจัดการกับ write load ที่มีปริมาณมากอยู่ดี
- **Redis pub/sub server**: เราใช้ประโยชน์จากข้อเท็จจริงที่ว่า จะไม่มีการใช้ memory เลยหาก channel ถูกสร้างขึ้นแต่ยังไม่ถูกใช้งาน ดังนั้นเราสามารถ pre-allocate channel ล่วงหน้าให้กับผู้ใช้ทุกคนที่ใช้ฟีเจอร์เพื่อนที่อยู่ใกล้เคียง เพื่อหลีกเลี่ยงการต้องจัดการเช่น การเปิด channel ใหม่เมื่อผู้ใช้ online และการแจ้งเตือน websocket server ที่ active อยู่

### **เจาะลึกการขยายคอมโพเนนต์ redis pub/sub (Scaling deep-dive on redis pub/sub component)**

เราจะต้องใช้ memory ประมาณ 200GB เพื่อดูแล pub/sub channel ทั้งหมด ซึ่งสามารถทำได้โดยใช้ redis server 2 ตัว ตัวละ 100GB

เนื่องจากเราต้อง push การอัปเดตตำแหน่งประมาณ 14 ล้านครั้งต่อวินาที เราจะต้องมี redis server อย่างน้อย 140 ตัวเพื่อรองรับปริมาณโหลดนั้น โดยสมมติว่า server หนึ่งตัวสามารถรองรับการ push ได้ ~100k ครั้งต่อวินาที

ดังนั้นเราจึงต้องมี redis server cluster แบบกระจาย (distributed) เพื่อรองรับภาระ CPU ที่หนักหน่วงนี้

เพื่อรองรับ distributed redis cluster เราจำเป็นต้องใช้คอมโพเนนต์ service discovery เช่น zookeeper หรือ etcd เพื่อติดตามว่า server ตัวใดยัง alive อยู่บ้าง

ข้อมูลที่เราต้อง encode ไว้ในคอมโพเนนต์ service discovery มีดังนี้:

<div style="margin-left:3rem">
    <img src="./images/channel-distribution-data.png" alt="channel-distribution-data" width="500" />
</div>

Web socket server ใช้ข้อมูลที่ถูก encode นี้ ซึ่งดึงมาจาก zookeeper เพื่อกำหนดว่า channel ใดตัวหนึ่งอยู่ที่ไหน เพื่อความมีประสิทธิภาพ ข้อมูล hash ring สามารถ cache ไว้ใน memory บน websocket server แต่ละตัวได้

ในแง่ของการขยายหรือลดขนาด server cluster เราสามารถตั้ง job รายวันเพื่อขยาย cluster ตามที่จำเป็นโดยอ้างอิงจากข้อมูลทราฟฟิกในอดีต (historical traffic data) เรายังสามารถ overprovision cluster เพื่อรองรับ traffic ที่พุ่งสูงขึ้นเป็นครั้งคราวได้อีกด้วย

Redis cluster สามารถถือว่าเป็น stateful storage server ได้ เนื่องจากมี state บางส่วนที่ต้องดูแลสำหรับ channel และจำเป็นต้องมีการประสานงาน (coordination) กับผู้ subscribe เพื่อให้พวกเขาส่งมอบ (hand-off) ไปยัง node ที่ถูกจัดสรรใหม่ใน cluster

เราต้องระมัดระวังปัญหาที่อาจเกิดขึ้นระหว่างการดำเนินการขยาย (scaling operations):
 * จะมี resubscription request จำนวนมากจาก web socket server เนื่องจาก channel ถูกย้ายไปมา
 * การอัปเดตตำแหน่งบางส่วนอาจถูกพลาดไปจาก client ระหว่างการดำเนินการนี้ ซึ่งเป็นสิ่งที่ยอมรับได้สำหรับโจทย์นี้ แต่เราก็ควรพยายามลดโอกาสที่มันจะเกิดขึ้นให้น้อยที่สุด ควรพิจารณาทำการดำเนินการเช่นนี้ในช่วงเวลาที่ traffic ต่ำที่สุดของวัน
 * เราสามารถใช้ประโยชน์จาก consistent hashing เพื่อลดปริมาณ channel ที่ต้องถูกย้ายเมื่อมีการเพิ่ม/ลด server

<div style="margin-left:3rem">
    <img src="./images/consistent-hashing.png" alt="consistent-hashing" width="500" />
</div>

### **การเพิ่ม/ลบเพื่อน (Adding/removing friends)**

เมื่อใดก็ตามที่มีการเพิ่ม/ลบเพื่อน websocket server ที่รับผิดชอบผู้ใช้ที่ได้รับผลกระทบจำเป็นต้อง subscribe/unsubscribe จาก channel ของเพื่อนคนนั้น

เนื่องจากฟีเจอร์ "เพื่อนที่อยู่ใกล้เคียง" เป็นส่วนหนึ่งของแอปที่ใหญ่กว่า เราสามารถสมมติได้ว่ามี callback ฝั่ง mobile client ที่ถูกลงทะเบียนไว้ทุกครั้งที่มี event ใด ๆ เกิดขึ้น และ client จะส่งข้อความไปยัง websocket server เพื่อทำการดำเนินการที่เหมาะสม

### **ผู้ใช้ที่มีเพื่อนจำนวนมาก (Users with many friends)**

เราสามารถกำหนดเพดาน (cap) จำนวนเพื่อนสูงสุดที่หนึ่งคนสามารถมีได้ เช่น facebook มีเพดานสูงสุดที่ 5000 คน

Websocket server ที่จัดการผู้ใช้ประเภท "whale" (ผู้ใช้ที่มีเพื่อนจำนวนมาก) อาจมีโหลดสูงกว่าปกติในฝั่งของมัน แต่ตราบใดที่เรามี web socket server เพียงพอ ก็ไม่น่าจะเป็นปัญหา

### **คนแปลกหน้าที่อยู่ใกล้เคียง (Nearby random person)**

จะเกิดอะไรขึ้นถ้าผู้สัมภาษณ์ต้องการอัปเดตการออกแบบให้รวมฟีเจอร์ที่เราสามารถเห็นคนแปลกหน้าปรากฏขึ้นบนแผนที่เพื่อนที่อยู่ใกล้เคียงของเราเป็นครั้งคราว

วิธีหนึ่งในการจัดการกับสิ่งนี้คือ การกำหนด pool ของ pubsub channel โดยอิงจาก geohash:

<div style="margin-left:3rem">
    <img src="./images/geohash-pubsub.png" alt="geohash-pubsub" width="500" />
</div>

ใครก็ตามที่อยู่ภายใน geohash เดียวกันจะ subscribe เข้ากับ channel ที่เหมาะสม เพื่อรับการอัปเดตตำแหน่งของผู้ใช้แปลกหน้า:

<div style="margin-left:3rem">
    <img src="./images/location-updates-geohash.png" alt="location-updates-geohash" width="500" />
</div>

เรายังสามารถ subscribe เข้ากับหลาย geohash เพื่อจัดการกับกรณีที่บางคนอยู่ใกล้แต่อยู่ใน geohash ที่อยู่ติดกัน (bordering):

<div style="margin-left:3rem">
    <img src="./images/geohash-borders.png" alt="geohash-borders" width="500" />
</div>

### **ทางเลือกอื่นแทน Redis pub/sub (Alternative to Redis pub/sub)**

ทางเลือกอื่นในการใช้ Redis สำหรับ pub/sub คือการใช้ประโยชน์จาก Erlang ซึ่งเป็นภาษาโปรแกรมมิ่งทั่วไป (general programming language) ที่ optimize ไว้สำหรับแอปพลิเคชัน distributed computing

ด้วย Erlang เราสามารถ spawn process ขนาดเล็กจำนวนหลายล้านตัวที่สื่อสารกันเองได้ เราสามารถจัดการทั้ง websocket connection และ pub/sub channel ภายใน distributed erlang application เดียวได้

อย่างไรก็ตาม ความท้าทายในการใช้ Erlang คือ มันเป็นภาษาโปรแกรมมิ่งเฉพาะกลุ่ม (niche) และอาจเป็นเรื่องยากที่จะหานักพัฒนา erlang ที่มีความสามารถสูง

---

## ขั้นตอนที่ 4: สรุปส่งท้าย (Wrap Up)

เราได้ออกแบบระบบที่รองรับฟีเจอร์เพื่อนที่อยู่ใกล้เคียงสำเร็จแล้ว

คอมโพเนนต์หลัก:
- **Web socket servers**: การสื่อสารแบบ real-time ระหว่าง client และ server
- **Redis**: การอ่านและเขียนข้อมูลตำแหน่งที่รวดเร็ว + pub/sub channel

เรายังได้สำรวจวิธีการขยาย restful api server, websocket server, ชั้นข้อมูล (data layer), redis pub/sub server และเรายังได้สำรวจทางเลือกอื่นแทนการใช้ Redis Pub/Sub อีกด้วย นอกจากนี้เรายังได้สำรวจฟีเจอร์ "คนแปลกหน้าที่อยู่ใกล้เคียง" (random nearby person) ด้วยเช่นกัน
