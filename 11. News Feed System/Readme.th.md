# บทที่ 11: ออกแบบระบบ News Feed (Design a News Feed System)

## บทนำ
**ระบบ news feed** แสดงรายการโพสต์ (สถานะ, รูปภาพ, วิดีโอ, และลิงก์) จากคนที่ผู้ใช้เชื่อมต่อด้วย โดยมีการอัปเดตอยู่ตลอดเวลา ตัวอย่างเช่น news feed ของ Facebook, ฟีดของ Instagram, และ timeline ของ Twitter บทนี้จะสำรวจการออกแบบระบบ news feed ที่สามารถขยายตัวได้ (scalable)

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)

### ข้อกำหนด (Requirements)
1. **แพลตฟอร์ม:** ระบบรองรับทั้งเว็บแอปและแอปมือถือ
2. **ฟีเจอร์:**
   - ผู้ใช้สามารถเผยแพร่ (publish) โพสต์ได้
   - ผู้ใช้สามารถดูโพสต์จากเพื่อน ๆ ใน news feed ของตนเองได้
3. **การเรียงลำดับ (Sorting):** ฟีดถูกเรียงลำดับตามเวลาแบบย้อนหลัง (**reverse chronological order**) เพื่อความง่าย
4. **สเกล (Scale):**
   - ผู้ใช้สามารถมีเพื่อนได้สูงสุด 5,000 คน
   - ผู้ใช้งานที่ใช้งานต่อวัน (DAU) จำนวน 10 ล้านคน
   - ฟีดอาจประกอบด้วยข้อความ, รูปภาพ, และวิดีโอ

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)

### ภาพรวม (Overview)
การออกแบบประกอบด้วยลำดับการทำงานหลัก 2 ส่วน:
1. **Feed Publishing:** ผู้ใช้เผยแพร่โพสต์ ซึ่งจะถูกเขียนลงฐานข้อมูลและเผยแพร่ (propagate) ไปยังฟีดของเพื่อน ๆ
2. **News Feed Building:** ผู้ใช้ดึง (retrieve) news feed ของตนเองโดยการรวบรวมโพสต์จากเพื่อน ๆ ตามลำดับเวลาแบบย้อนหลัง

---

### News Feed APIs
1. **Feed Publishing API:**
   - **Endpoint:** `POST /v1/me/feed`
   - **พารามิเตอร์:** `content` (เนื้อหาข้อความของโพสต์) และ `auth_token` (การยืนยันตัวตน)

2. **News Feed Retrieval API:**
   - **Endpoint:** `GET /v1/me/feed`
   - **พารามิเตอร์:** `auth_token` (การยืนยันตัวตน)

---

### Feed Publishing

   <div style="margin-left:3rem">
      <img src="./images/feed-publishing.png" alt="Feed Publishing" width="400">
   </div>

1. **ปฏิสัมพันธ์ของผู้ใช้ (User Interaction):** ผู้ใช้เผยแพร่โพสต์ผ่าน feed publishing API
2. **Load Balancer:** กระจายทราฟฟิกไปยังเว็บเซิร์ฟเวอร์
3. **เว็บเซิร์ฟเวอร์ (Web Servers):** ยืนยันตัวตนของ request และส่งต่อไปยังบริการต่าง ๆ
4. **Post Service:** จัดเก็บโพสต์ไว้ในฐานข้อมูลและแคช
5. **Fanout Service:** เผยแพร่โพสต์ไปยัง news feed ของเพื่อน ๆ ในแคช
6. **Notification Service:** ส่งการแจ้งเตือนไปยังเพื่อน ๆ

---

### News Feed Building

   <div style="margin-left:3rem">
      <img src="./images/news-feed-building.png" alt="News Feed Building" width="400">
   </div>

1. **ปฏิสัมพันธ์ของผู้ใช้ (User Interaction):** ผู้ใช้ร้องขอ news feed ของตนเองผ่าน retrieval API
2. **Load Balancer:** กระจายทราฟฟิกไปยังเว็บเซิร์ฟเวอร์
3. **เว็บเซิร์ฟเวอร์ (Web Servers):** ส่งต่อ request ไปยัง news feed service
4. **News Feed Service:** ดึง post ID จาก news feed cache และดึงรายละเอียดของโพสต์ทั้งหมดจากฐานข้อมูลหรือแคช


---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### เจาะลึก Feed Publishing (Feed Publishing Deep Dive)
1. **เว็บเซิร์ฟเวอร์ (Web Servers):**
   - ยืนยันตัวตนของผู้ใช้ด้วย `auth_token`
   - บังคับใช้การจำกัดอัตรา (rate limit) เพื่อป้องกันสแปม

2. **Fanout Service:**
   - **Fanout on Write:** push โพสต์ไปยังฟีดของเพื่อน ๆ ทันทีที่มีการเขียน (write time)
     - **ข้อดี:** อัปเดตแบบ real-time, ดึงฟีดได้อย่างรวดเร็ว
     - **ข้อเสีย:** ใช้ทรัพยากรมากสำหรับผู้ใช้ที่มีเพื่อนจำนวนมาก
   - **Fanout on Read:** ดึง (pull) โพสต์ในตอนที่มีการอ่าน (read time)
     - **ข้อดี:** มีประสิทธิภาพสำหรับผู้ใช้ที่ไม่ค่อย active
     - **ข้อเสีย:** ดึงฟีดได้ช้ากว่า
   - **แนวทางแบบผสม (Hybrid Approach):** ใช้โมเดลแบบ push สำหรับผู้ใช้ส่วนใหญ่ และใช้โมเดลแบบ pull สำหรับผู้ใช้ที่มีการเชื่อมต่อจำนวนมาก (เช่น คนดัง)

        <img src="./images/feed-publishing-deep-dive.png" alt="Feed Publishing Deep Dive" width="500">

    **Fanout service** ทำงานดังนี้:

    1. **ดึง Friend ID (Fetch Friend IDs):** ดึงรายชื่อเพื่อนจาก graph database
    2. **กรองเพื่อนจากแคช (Filter Friends from Cache):** เข้าถึงการตั้งค่าของผู้ใช้ในแคชเพื่อคัดเพื่อนบางคนออก (เช่น เพื่อนที่ถูกปิดเสียง (muted) หรือมีการตั้งค่าการแชร์แบบเฉพาะเจาะจง)
    3. **ส่งไปยัง Message Queue (Send to Message Queue):** ส่งรายชื่อเพื่อนที่กรองแล้วพร้อมกับ post ID ของโพสต์ใหม่ไปยัง message queue เพื่อประมวลผลต่อ
    4. **Fanout Workers:** worker ดึงข้อมูลจาก message queue และอัปเดต news feed cache โดยแคชจะเก็บข้อมูลในรูปแบบ mapping `<post_id, user_id>` แทนที่จะเก็บ user object และ post object แบบเต็ม เพื่อประหยัดหน่วยความจำ
    5. **จัดเก็บใน News Feed Cache (Store in News Feed Cache):** เพิ่ม post ID ใหม่ต่อท้าย news feed cache ของเพื่อน ๆ โดยมีการกำหนดจำนวนสูงสุดที่ปรับแต่งได้ (configurable limit) เพื่อให้เก็บเฉพาะโพสต์ล่าสุดเท่านั้น เนื่องจากผู้ใช้ส่วนใหญ่สนใจแต่เนื้อหาล่าสุด ทำให้ควบคุมปริมาณการใช้หน่วยความจำของแคชได้

        <img src="./images/fanout-service.png" alt="Fanout Service" width="500">

## เจาะลึกการดึง News Feed (News Feed Retrieval Deep Dive)

### สถาปัตยกรรมของแคช (Cache Architecture)
แคชถูกแบ่งออกเป็น 5 ชั้น:
1. **News Feed Cache:** จัดเก็บ post ID เพื่อการดึงข้อมูลที่รวดเร็ว
2. **Content Cache:** จัดเก็บรายละเอียดของโพสต์ (โพสต์ที่ได้รับความนิยมจะอยู่ใน hot cache)
3. **Social Graph Cache:** จัดเก็บข้อมูลความสัมพันธ์ของผู้ใช้
4. **Action Cache:** ติดตาม action ของผู้ใช้ (การกดไลก์, การตอบกลับ, การแชร์)
5. **Counter Cache:** เก็บจำนวนนับของการกดไลก์, การตอบกลับ, ผู้ติดตาม ฯลฯ

    <img src="./images/cache-architecture.png" alt="Cache Architecture" width="500">
---

## การปรับปรุงประสิทธิภาพที่สำคัญ (Key Optimizations)

### การขยายตัว (Scaling)
1. **การขยายฐานข้อมูล (Database Scaling):**
   - การขยายแนวนอน (horizontal scaling) และการทำ sharding
   - ใช้ read replica สำหรับ query ที่มีทราฟฟิกสูง
2. **ชั้นเว็บแบบไม่เก็บสถานะ (Stateless Web Tier):** รักษาให้เว็บเซิร์ฟเวอร์เป็นแบบ stateless เพื่อให้สามารถทำ horizontal scaling ได้

### การแคช (Caching)
1. จัดเก็บข้อมูลที่ถูกเข้าถึงบ่อยไว้ในหน่วยความจำ
2. ใช้แคชหลายชั้นเพื่อลด latency และภาระของฐานข้อมูล

### ความน่าเชื่อถือ (Reliability)
1. **Consistent Hashing:** กระจาย request ไปยังเซิร์ฟเวอร์ต่าง ๆ อย่างสม่ำเสมอ
2. **Message Queue:** แยกส่วนคอมโพเนนต์ของระบบออกจากกัน และบัฟเฟอร์ทราฟฟิก

### การเฝ้าติดตาม (Monitoring)
1. ติดตาม metric สำคัญ เช่น QPS (queries per second) และ latency
2. เฝ้าติดตามอัตราการ hit ของแคช (cache hit rate) และปรับการตั้งค่าตามความเหมาะสม
