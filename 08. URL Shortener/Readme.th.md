# บทที่ 8: การออกแบบ URL Shortener (Design a URL Shortener)

## บทนำ
บทนี้จะกล่าวถึงการออกแบบบริการย่อ URL (URL shortening service) อย่าง TinyURL เป้าหมายหลักของระบบได้แก่ **การย่อ URL**, **การ redirect** และ **ความสามารถในการขยายระบบสูง (high scalability)** เพื่อรองรับปริมาณ traffic ขนาดใหญ่

### ความต้องการ (Requirements)
- shortened URL ต้อง **ไม่ซ้ำกัน (unique)** และ **สั้นที่สุดเท่าที่จะเป็นไปได้**
- รองรับ **การสร้าง URL 100 ล้านรายการต่อวัน** โดยรองรับการใช้งานได้นาน 10 ปี
- รองรับ **read operation ที่มีประสิทธิภาพ** ด้วยอัตราส่วน read-to-write ที่ 10:1
- จัดเก็บข้อมูล 365,000 ล้านระเบียน (records) ซึ่งต้องการพื้นที่จัดเก็บประมาณ **365 TB** ตลอด 10 ปี

---

## ขั้นตอนที่ 1: การออกแบบระดับสูง (High-Level Design)

### API Endpoint
1. **การย่อ URL (URL Shortening):**
   - Endpoint: `POST api/v1/data/shorten`
   - พารามิเตอร์: `{longUrl: longURLString}`
   - ส่งกลับ: `shortURL`

2. **การ Redirect URL (URL Redirecting):**
   - Endpoint: `GET api/v1/shortUrl`
   - ส่งกลับ: `longURL` เพื่อใช้ในการ redirect

    <p align="center">
    <img src="./images/url-redirection.png" alt="URL Redirection" width="600">
    </p>

### การ Redirect URL (URL Redirection)
- **301 Redirect:** 301 redirect แสดงว่า URL ที่ร้องขอถูกย้ายไปยัง long URL แบบ "ถาวร" เบราว์เซอร์จะแคชการตอบกลับนี้ไว้ และ
request ครั้งต่อ ๆ ไปสำหรับ URL เดียวกันจะไม่ถูกส่งไปยังบริการย่อ URL อีก
- **302 Redirect:** เป็นแบบชั่วคราว เหมาะสำหรับงานด้าน analytics เช่น การติดตามการคลิก (click)

### การย่อ URL (URL Shortening)
<p align="center">
    <img src="./images/url-shortening.png" alt="URL Shortening" width="400">
</p>

- ใช้ **hash function** เพื่อสร้าง short URL โดย map long URL ไปยังเวอร์ชันที่ย่อแล้วซึ่งไม่ซ้ำกัน
- hash function ต้องตอบโจทย์ความต้องการดังนี้:
    - longURL แต่ละตัวต้องถูก hash ไปเป็น hashValue เพียงค่าเดียว
    - hashValue แต่ละค่าต้องสามารถ map กลับไปยัง longURL ได้

---

## ขั้นตอนที่ 2: เจาะลึกการออกแบบ (Deep Dive into Design)

### Data Model
จัดเก็บ mapping ระหว่าง `<shortURL, longURL>` ไว้ในฐานข้อมูลเชิงสัมพันธ์ (relational database) เพื่อ optimize การใช้หน่วยความจำ schema ของตารางประกอบด้วย:
- `id` (primary key)
- `shortURL`
- `longURL`

    <img src="./images/table-schema.png" alt="Table Schema" width="300">

### Hash Function
#### 1. การแปลงเป็น Base 62 (Base 62 Conversion):
- เข้ารหัสตัวเลขโดยใช้ตัวอักษร `[0-9, a-z, A-Z]` ซึ่งมีทั้งหมด **62 ตัวอักษรที่เป็นไปได้**
- การแปลงฐาน (base conversion) เป็นอีกแนวทางหนึ่งที่นิยมใช้กับบริการย่อ URL
- สามารถกำหนด unique id ให้กับ short URL แล้วนำ ID นั้นไปแปลงเป็น base 62 เพื่อให้ได้ short URL
- hash ที่มีความยาว 7 ตัวอักษรรองรับ URL ที่ไม่ซ้ำกันได้ถึง **3.5 ล้านล้าน (trillion)** รายการ ซึ่งเพียงพอสำหรับ URL จำนวน 365,000 ล้านรายการ

**ตัวอย่าง:**
แปลง ID `2009215674938` เป็น Base 62:
- `2009215674938` → `zn9edcu`

#### 2. Hash + การแก้ปัญหา Collision (Hash + Collision Resolution):
- ใช้ hash function เช่น CRC32, MD5 หรือ SHA-1

    <img src="./images/hash-function.png" alt="Hash Function" width="500">

- แนวทางหนึ่งคือการเก็บตัวอักษร 7 ตัวแรกของ hash value อย่างไรก็ตาม วิธีนี้อาจทำให้เกิด hash collision
- ในการแก้ collision ให้ต่อท้าย (append) สตริงที่กำหนดไว้ล่วงหน้าแบบวนซ้ำ (recursively) จนกว่าจะไม่มี collision อีก แต่วิธีนี้อาจมีค่าใช้จ่ายสูง
- แก้ collision ด้วย **Bloom Filter** เพื่อการค้นหาที่มีประสิทธิภาพ

    <p align="center">
    <img src="./images/url-lookup.png" alt="URL Lookup" width="500">
    </p>

### การเปรียบเทียบ (Comparison)

-  **Hash + Collision Resolution:**
    - ความยาวของ short URL คงที่
    - ไม่ต้องใช้ตัวสร้าง unique ID
    - อาจเกิด collision และต้องมีการแก้ไข
    - ไม่สามารถหา short URL ตัวถัดไปที่ว่างอยู่ได้ เนื่องจากไม่ได้ขึ้นอยู่กับ ID

- **Base 62 Conversion**
    - ความยาวไม่คงที่ และเพิ่มขึ้นตาม ID
    - ต้องใช้ตัวสร้าง unique ID
    - ไม่มี collision เกิดขึ้น
    - หา short URL ตัวถัดไปได้ง่าย หาก ID เพิ่มขึ้นทีละ 1 (อาจเป็นข้อกังวลด้านความปลอดภัย)


---

### ลำดับการทำงานของการย่อ URL (URL Shortening Flow)

<p align="center">
    <img src="./images/url-shortening-flow.png" alt="URL Shortening" width="500">
</p>

1. ตรวจสอบว่า `longURL` มีอยู่ในฐานข้อมูลแล้วหรือไม่
2. หากพบ ให้ส่งคืน `shortURL` ที่มีอยู่แล้ว
3. หากไม่พบ:
   - สร้าง unique ID โดยใช้ **ตัวสร้าง ID แบบ distributed (distributed ID generator)**
   - แปลง ID เป็น `shortURL` โดยใช้ Base 62
   - จัดเก็บ mapping ของ `<id, shortURL, longURL>` ลงในฐานข้อมูล



---

### ลำดับการทำงานของการ Redirect URL (URL Redirecting Flow)
<p align="center">
    <img src="./images/url-redirecting-flow.png" alt="URL Shortening" width="600">
</p>

1. ผู้ใช้คลิกที่ `shortURL`
2. Query mapping ของ `<shortURL, longURL>`:
   - ตรวจสอบ **cache** ก่อนเพื่อการเข้าถึงที่รวดเร็วกว่า
   - หากไม่มีใน cache ให้ query จากฐานข้อมูล
3. Redirect ผู้ใช้ไปยัง `longURL`


---

## ประเด็นเพิ่มเติม (Additional Considerations)
### Rate Limiter
- ป้องกันการใช้งานในทางที่ผิด (abuse) โดยการตั้งขีดจำกัดจำนวน request ต่อ IP

### ความสามารถในการขยายระบบ (Scalability)
1. **Web Tier:** เป็นแบบ stateless ขยายระบบได้โดยการเพิ่ม/ลบ web server
2. **Database Tier:** ใช้ replication และ sharding

### Analytics
- เก็บข้อมูล เช่น อัตราการคลิก (click rate), แหล่งที่มา (source) และ timestamp เพื่อใช้ในเชิงธุรกิจ

### ความพร้อมใช้งานสูงและความน่าเชื่อถือ (High Availability and Reliability)
- รับประกันบริการที่สอดคล้องและน่าเชื่อถือ โดยใช้ database replication และการออกแบบที่ทนต่อความล้มเหลว (fault-tolerant)
