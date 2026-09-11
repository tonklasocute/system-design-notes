# บทที่ 10: ออกแบบระบบแจ้งเตือน (Design a Notification System)

## บทนำ
**ระบบแจ้งเตือน (notification system)** เป็นสิ่งสำคัญสำหรับแอปพลิเคชันยุคใหม่ โดยทำหน้าที่ส่งข้อมูลอัปเดตที่ทันเวลา เช่น การแจ้งเตือนเกี่ยวกับผลิตภัณฑ์, กิจกรรม, โปรโมชั่น, และการแจ้งเตือนต่าง ๆ (alerts) การแจ้งเตือนสามารถส่งผ่านช่องทางต่อไปนี้:
1. **Push notification** (มือถือหรือเดสก์ท็อป)
2. **ข้อความ SMS**
3. **อีเมล (Emails)**

บทนี้เน้นการออกแบบระบบที่สามารถขยายตัวได้ (scalable) เพื่อรองรับการส่งการแจ้งเตือนหลายล้านครั้งต่อวัน

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)
### ข้อกำหนด (Requirements)
- **ประเภทการแจ้งเตือน (Notification Types):** Push notification, SMS, และอีเมล
- **การนำส่ง (Delivery):** เป็นระบบ soft real-time ที่มีความล่าช้าน้อยที่สุด
- **แพลตฟอร์ม (Platforms):** iOS, Android, และเดสก์ท็อป
- **ตัวกระตุ้น (Triggers):** การแจ้งเตือนสามารถถูกกระตุ้นจากแอปพลิเคชันฝั่งไคลเอนต์ หรือถูกกำหนดเวลาไว้ล่วงหน้าบนเซิร์ฟเวอร์
- **สเกล (Scale):**
  - **Push Notification:** 10 ล้านครั้ง/วัน
  - **SMS:** 1 ล้านครั้ง/วัน
  - **อีเมล:** 5 ล้านครั้ง/วัน
- **รองรับการยกเลิกรับข้อมูล (Opt-out Support):** ผู้ใช้สามารถปิดการแจ้งเตือนแต่ละประเภทได้

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)

### คอมโพเนนต์ (Components)

1. **ประเภทการแจ้งเตือน (Notification Types):**
   - **Push Notification บน iOS:** ใช้ **Apple Push Notification Service (APNS)**
   - **Push Notification บน Android:** ใช้ **Firebase Cloud Messaging (FCM)**
   - **ข้อความ SMS:** ใช้บริการจากผู้ให้บริการภายนอก (third-party) เช่น Twilio หรือ Nexmo
   - **อีเมล:** ใช้บริการอีเมลเชิงพาณิชย์ เช่น SendGrid หรือ Mailchimp

2. **การรวบรวมข้อมูลติดต่อ (Contact Info Gathering):**
   <div style="margin-left:3rem">
      <img src="./images/contact-info-gathering.png" alt="Contact Info Gathering" width="500">
   </div>

   - เก็บรวบรวม device token, หมายเลขโทรศัพท์, หรือที่อยู่อีเมล ในขั้นตอนการติดตั้งแอปหรือการสมัครสมาชิก
   - จัดเก็บข้อมูลติดต่อไว้ในฐานข้อมูล:
     - **ตาราง Device Token:** สำหรับ push notification
     - **ตาราง User:** สำหรับอีเมลและหมายเลขโทรศัพท์


3. **ลำดับการส่งการแจ้งเตือน (Notification Sending Flow):**

   <div style="margin-left:3rem">
      <img src="./images/high-level-design.png" alt="High Level Design" width="500">
   </div>

   - **Trigger Services:**
      - สร้าง event เพื่อเริ่มต้นการแจ้งเตือน (เช่น การแจ้งเตือนบิลค่าใช้จ่าย, การอัปเดตสถานะการจัดส่ง)
      - บริการนี้อาจเป็น micro-service, cron job, หรือระบบแบบกระจาย (distributed system) ที่กระตุ้น event การส่งการแจ้งเตือน
   - **Notification Server:**
      - จัดเตรียม API ให้บริการต่าง ๆ เรียกใช้เพื่อส่งการแจ้งเตือน
      - ทำการตรวจสอบเบื้องต้น (basic validation) เพื่อยืนยันอีเมลและหมายเลขโทรศัพท์
      - Query ฐานข้อมูลหรือแคชเพื่อดึงข้อมูลที่จำเป็นสำหรับสร้าง (render) การแจ้งเตือน
   - **บริการภายนอก (Third-Party Services):** นำส่งการแจ้งเตือนไปยังผู้ใช้



### ความท้าทายในการออกแบบเบื้องต้น (Challenges in Initial Design)
- **จุดที่อาจเกิดความล้มเหลวทั้งระบบ (Single Point of Failure, SPOF):** เซิร์ฟเวอร์แจ้งเตือนเพียงตัวเดียวสามารถทำให้ทั้งระบบล่มได้
- **ปัญหาด้านการขยายตัว (Scalability Issues):** ยากที่จะขยายฐานข้อมูล, แคช, และคอมโพเนนต์ประมวลผลแบบอิสระจากกัน
- **คอขวดด้านประสิทธิภาพ (Performance Bottlenecks):** ต้องการทรัพยากรจำนวนมากในการส่งการแจ้งเตือน

### การออกแบบที่ปรับปรุงแล้ว (Improved Design)

   <div style="margin-left:3rem">
      <img src="./images/improved-design.png" alt="Improved Design" width="500">
   </div>

- ย้ายฐานข้อมูลและแคชออกจากเซิร์ฟเวอร์แจ้งเตือน
- นำการขยายแนวนอน (**horizontal scaling**) มาใช้ โดยมีเซิร์ฟเวอร์แจ้งเตือนหลายตัว
- ใช้ **message queue** เพื่อแยกส่วนคอมโพเนนต์ต่าง ๆ ของระบบออกจากกัน (decouple)
   - message queue ทำหน้าที่เป็นบัฟเฟอร์เมื่อมีปริมาณการแจ้งเตือนที่ต้องส่งออกจำนวนมาก
- เพิ่ม worker ที่ดึง event การแจ้งเตือนจาก message queue และส่งไปยังบริการภายนอกที่เกี่ยวข้อง



---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### ความน่าเชื่อถือ (Reliability)
1. **ป้องกันการสูญหายของข้อมูล (Prevent Data Loss):**
   <div style="margin-left:3rem">
   <img src="./images/data-loss.png" alt="Data Loss" width="400">
   </div>

   - บันทึกข้อมูลการแจ้งเตือนลงในฐานข้อมูล และทำการ implement กลไก retry
   - มีการเพิ่ม Notification log database เพื่อคงข้อมูลไว้อย่างถาวร (data persistence)


2. **การตัดข้อมูลซ้ำ (Deduplication):**
   - ตรวจสอบ event ID เพื่อหลีกเลี่ยงการส่งการแจ้งเตือนซ้ำ
   - เมื่อ event การแจ้งเตือนเข้ามาครั้งแรก ให้ตรวจสอบว่าเคยพบ event นี้มาก่อนหรือไม่ โดยตรวจสอบจาก event ID
หากเคยพบมาก่อนแล้ว ให้ทิ้งไป มิฉะนั้นให้ส่งการแจ้งเตือนออกไป


### คอมโพเนนต์เพิ่มเติม (Additional Components)
   <div style="margin-left:3rem">
   <img src="./images/events-tracking.png" alt="Events Tracking" width="400">
   </div>

1. **Notification Templates:** เทมเพลตที่จัดรูปแบบไว้ล่วงหน้า เพื่อให้การแจ้งเตือนมีความสม่ำเสมอและมีประสิทธิภาพ
2. **การตั้งค่าการแจ้งเตือน (Notification Settings):**
   - ผู้ใช้สามารถเลือกรับหรือยกเลิกรับการแจ้งเตือนในแต่ละช่องทาง (push, SMS, หรืออีเมล) ได้
   - จัดเก็บไว้ในตารางการตั้งค่าการแจ้งเตือนโดยเฉพาะ
3. **การจำกัดอัตรา (Rate Limiting):** จำกัดความถี่ของการแจ้งเตือนที่ส่งไปยังผู้ใช้
4. **กลไก Retry (Retry Mechanism):** ส่งการแจ้งเตือนซ้ำหากบริการภายนอกล้มเหลว
5. **การเฝ้าติดตาม Queue (Monitoring Queues):** ติดตามการแจ้งเตือนที่อยู่ใน queue เพื่อขยาย worker แบบไดนามิก
6. **การติดตาม Event (Event Tracking):** รวบรวม metric ต่าง ๆ เช่น อัตราการเปิดอ่าน (open rate), อัตราการคลิก (click rate), และการมีส่วนร่วม (engagement)


### ความปลอดภัย (Security)
- ใช้ **AppKey** และ **AppSecret** เพื่อยืนยันตัวตนและรักษาความปลอดภัยของ API สำหรับ push notification

### ลำดับการทำงานของการแจ้งเตือน (Notification Flow)

   <div style="margin-left:3rem">
   <img src="./images/updated-design.png" alt="Updated Design" width="500">
   </div>

1. Trigger service เรียกใช้ API เพื่อส่งการแจ้งเตือน
2. เซิร์ฟเวอร์แจ้งเตือนตรวจสอบความถูกต้องของ request และดึง metadata จากแคชหรือฐานข้อมูล
3. Event การแจ้งเตือนจะถูกส่งไปยัง message queue
4. Worker ประมวลผล event และติดต่อกับบริการภายนอก
5. บริการภายนอกนำส่งการแจ้งเตือนไปยังผู้ใช้


---

## การปรับปรุงประสิทธิภาพที่สำคัญ (Key Optimizations)
1. **การขยายแนวนอน (Horizontal Scaling):** เพิ่มเซิร์ฟเวอร์แจ้งเตือนเพื่อกระจายโหลด
2. **Message Queue:** แยกส่วนการประมวลผลออกจากกันเพื่อรองรับปริมาณงานจำนวนมาก
3. **การแคช (Caching):** ลด latency ด้วยการแคชข้อมูลที่ถูกเข้าถึงบ่อย
4. **การ Crawl แบบกระจาย (Distributed Crawling):** ปรับปรุงการนำส่งข้อความให้เหมาะสมตามพื้นที่ทางภูมิศาสตร์ เพื่อประสิทธิภาพที่ดีขึ้น
