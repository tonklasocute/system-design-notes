# บทที่ 4: การออกแบบตัวจำกัดอัตรา (Design a Rate Limiter)

## บทนำ
บทนี้จะสำรวจการออกแบบและการทำงานของ **ตัวจำกัดอัตรา (Rate Limiter)** ซึ่งเป็นคอมโพเนนต์ที่ใช้ควบคุมอัตราการส่ง traffic จากไคลเอนต์หรือบริการต่าง ๆ ตัวจำกัดอัตรามีความสำคัญอย่างมากในการป้องกันการใช้งานในทางที่ผิด (abuse) ลดค่าใช้จ่าย และรักษาเสถียรภาพของทรัพยากรเซิร์ฟเวอร์ ตัวอย่างการใช้งาน ได้แก่ การจำกัดจำนวนโพสต์ การสร้างบัญชี และการรับรางวัล

## ประโยชน์ของการทำ Rate Limiting (Benefits of Rate Limiting)
- **ป้องกันการโจมตีแบบ DoS (Preventing DoS Attacks):** บล็อกการเรียกที่มากเกินไปเพื่อหลีกเลี่ยงการที่ทรัพยากรถูกใช้จนหมด (resource starvation)
- **ลดค่าใช้จ่าย (Cost Reduction):** จำกัด request ที่ไม่จำเป็นเพื่อลดค่าใช้จ่ายของเซิร์ฟเวอร์
- **ป้องกันภาระเกิน (Preventing Overloads):** กรอง request ที่มากเกินไปออกเพื่อรักษาเสถียรภาพของประสิทธิภาพเซิร์ฟเวอร์

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)
### ฟีเจอร์หลัก (Key Features)
- ตัวจำกัดอัตราของ API ฝั่งเซิร์ฟเวอร์
- รองรับ throttle rule ได้หลายแบบ
- รองรับระบบขนาดใหญ่ใน distributed environment
- มีตัวเลือกให้เป็น standalone service หรือฝังไว้ใน application-level code
- แจ้งเตือนผู้ใช้เมื่อถูก throttle

### ความต้องการ (Requirements)
- การ throttle request ที่แม่นยำ
- Latency ต่ำที่สุด
- ใช้หน่วยความจำน้อย
- รองรับการทำงานแบบ distributed
- มีการจัดการ exception ที่ชัดเจน
- มี fault tolerance สูง

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)
### ตำแหน่งที่วาง (Placement Options)
<div style="margin-left:2rem">
    <img src="./images/rate_limiter_architecture.png"  alt="Rate Limiting Middleware Architecture" width="550">
</div>

1. **การ Implement ฝั่งไคลเอนต์ (Client-Side Implementation):** ไม่น่าเชื่อถือ เนื่องจากอาจถูกใช้งานในทางที่ผิดได้ง่าย
2. **การ Implement ฝั่งเซิร์ฟเวอร์ (Server-Side Implementation):** เป็นตัวเลือกที่นิยมมากกว่า เนื่องจากควบคุมได้และน่าเชื่อถือกว่า
3. **Middleware (API Gateway):** ตัวเลือกที่ยืดหยุ่นสำหรับการทำ rate limiting แบบรวมศูนย์

### แนวทางในการเลือกตำแหน่ง (Guidelines for Placement)
- ประเมิน tech stack ที่มีอยู่และเลือกตัวเลือกที่มีประสิทธิภาพ
- เลือก algorithm ที่เหมาะสมตามความต้องการทางธุรกิจ
- ใช้ API gateway หากระบบใช้สถาปัตยกรรมแบบ microservices
- เลือกใช้โซลูชันเชิงพาณิชย์ (commercial solutions) หากทรัพยากรมีจำกัด

## ขั้นตอนที่ 3: อัลกอริทึมสำหรับ Rate Limiting (Rate Limiting Algorithms)
### 1. Token Bucket
<div style="margin-left:2rem">
  <img src="./images/token-bucket.png"  alt="Token Bucket Algorithm" width="550">
</div>

- **คำอธิบาย:** token จะถูกเติมเข้าไปใน bucket ด้วยอัตราคงที่ แต่ละ request จะใช้ (consume) token หนึ่งตัว
- **พารามิเตอร์:** ขนาดของ bucket และอัตราการเติม (refill rate)
- **ข้อดี:** ทำได้ง่าย ประหยัดหน่วยความจำ รองรับ traffic ที่พุ่งสูงเป็นช่วง (burst)
- **ข้อเสีย:** ต้องปรับแต่งพารามิเตอร์อย่างระมัดระวัง

### 2. Leaking Bucket
<div style="margin-left:2rem">
  <img src="./images/leaking-bucket.png"  alt="Leaking Bucket Algorithm" width="550">
</div>

- **คำอธิบาย:** ประมวลผล request ด้วยอัตราคงที่ โดยใช้ FIFO queue
- **ข้อดี:** ประหยัดหน่วยความจำ อัตราการไหลออก (outflow rate) คงที่
- **ข้อเสีย:** traffic ที่พุ่งสูงเป็นช่วงอาจทำให้ request ล่าสุดถูกหน่วงเวลา

  ตัวอย่าง: https://github.com/uber-go/ratelimit

### 3. Fixed Window Counter
<div style="margin-left:2rem">
  <img src="./images/fixed-window-counter.png"  alt="Fixed Window Counter" width="550">
</div>

- **คำอธิบาย:** แบ่งเวลาออกเป็นช่วง (interval) คงที่ และใช้ counter เพื่อจำกัดจำนวน request
- **ข้อดี:** เรียบง่าย มีประสิทธิภาพสำหรับ use case บางประเภท
- **ข้อเสีย:** traffic ที่พุ่งสูงตรงขอบของ window อาจทำให้จำนวน request เกิน quota ที่กำหนดได้

- traffic ที่พุ่งสูงกะทันหันตรงขอบของ time window
อาจทำให้มี request ผ่านเข้ามามากกว่า quota ที่อนุญาตไว้

  <img src="./images/fixed-window-issue.png"  alt="Fixed Window Issue" width="550">

### 4. Sliding Window Log
<div style="margin-left:2rem">
  <img src="./images/sliding-window-log.png"  alt="Sliding Window Log" width="550">
</div>

- **คำอธิบาย:** ติดตาม timestamp เพื่อให้เกิด rolling time window
- **ข้อดี:** จำกัดอัตราได้อย่างแม่นยำ
- **ข้อเสีย:** ใช้หน่วยความจำสูง

### 5. Sliding Window Counter
<div style="margin-left:2rem">
  <img src="./images/sliding-window-counter.png"  alt="Fixed Window Counter" width="550">
</div>

- **คำอธิบาย:** ผสมผสานวิธี fixed window และ sliding log เข้าด้วยกัน เพื่อลดความรุนแรงของ spike
- **ข้อดี:** ประหยัดหน่วยความจำ รองรับ traffic ที่พุ่งสูงเป็นช่วง
- **ข้อเสีย:** เป็นการประมาณค่า (approximation) จึงอาจไม่เข้มงวดสมบูรณ์แบบ

## สถาปัตยกรรมระดับสูง (High-Level Architecture)
<div style="margin-left:2rem">
  <img src="./images/architecture.png" style="margin-left: 40px; margin-top: 40px; margin-bottom: 20px;" alt="Architecture" width="550">
</div>

- **การจัดเก็บข้อมูล (Data Storage):** ใช้แคชในหน่วยความจำ (in-memory caching เช่น Redis) สำหรับ operation ของ counter ที่รวดเร็ว
- **ขั้นตอน:**
  1. ไคลเอนต์ส่ง request ไปยัง middleware
  2. Middleware ตรวจสอบ counter ใน Redis
  3. Request จะถูกประมวลผลหรือปฏิเสธตามขีดจำกัดที่ตั้งไว้

## ประเด็นขั้นสูง (Advanced Considerations)
### Distributed Environments
- **ความท้าทาย:** race condition และปัญหาการ synchronization
- **แนวทางแก้ไข:** ใช้ lock, Lua script หรือ sorted set ใน Redis รวมถึงใช้ centralized data store เพื่อทำ synchronization

### การปรับแต่งประสิทธิภาพ (Performance Optimizations)
- ตั้งค่าหลาย data center เพื่อลด latency
- ใช้โมเดล eventual consistency สำหรับ synchronization

### การมอนิเตอร์ (Monitoring)
- วิเคราะห์ข้อมูล (analytics) อย่างสม่ำเสมอ เพื่อให้แน่ใจว่า algorithm ยังมีประสิทธิภาพ และปรับกฎเกณฑ์ตามความจำเป็น
