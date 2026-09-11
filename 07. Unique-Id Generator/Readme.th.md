# บทที่ 7: การออกแบบตัวสร้าง Unique ID ในระบบ Distributed (Design a Unique ID Generator in Distributed Systems)

## บทนำ
บทนี้จะกล่าวถึงความท้าทายในการออกแบบ **ตัวสร้าง unique ID (Unique ID Generator)** สำหรับระบบ distributed คีย์แบบ auto-increment ตามแบบดั้งเดิมนั้นไม่เหมาะกับสภาพแวดล้อมแบบ distributed เนื่องจากความท้าทายด้านการขยายระบบ (scalability) และการซิงค์กัน (synchronization) บทนี้จะเน้นไปที่การสร้าง ID แบบตัวเลข 64 บิตที่ไม่ซ้ำกัน (unique) และเรียงลำดับได้ (sortable) โดยต้องตอบโจทย์ความต้องการดังนี้:
- ID ต้อง **ไม่ซ้ำกัน (unique)** และ **เรียงลำดับตามวันที่ (ordered by date)**
- ID ต้องอยู่ภายใน **64 บิต**
- ระบบควรสร้าง ID ได้ **มากกว่า 10,000 ID ต่อวินาที**

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)
### ความต้องการพื้นฐาน (Basic Requirements)
- ID ต้องไม่ซ้ำกัน เป็นตัวเลข และต้องอยู่ภายใน 64 บิต
- ID เพิ่มขึ้นตามเวลา แต่ไม่จำเป็นต้องเพิ่มทีละ `+1` เสมอไป
- ID ควรเรียงลำดับได้ตามวันที่
- ระบบต้องรองรับปริมาณงานสูง (10,000 ID/วินาที)

---

## ขั้นตอนที่ 2: ตัวเลือกการออกแบบระดับสูง (High-Level Design Options)
### 1. Multi-Master Replication
- **แนวทาง:** ใช้ `auto_increment` ของฐานข้อมูล โดยเพิ่มค่าทีละขั้น (step increment เช่น `+k` สำหรับเซิร์ฟเวอร์ k ตัว)

    <p align="left">
    <img src="./images/multi-master.png"  alt="Multi Master" width="400">
    </p>

- **ข้อเสีย:**
  - ขยายระบบข้าม data center ได้ยาก
  - ID ไม่ได้เพิ่มขึ้นตามเวลาเสมอไป
  - เกิดปัญหาการขยายระบบเมื่อมีการเพิ่ม/ลบเซิร์ฟเวอร์

### 2. UUID (Universally Unique Identifier)
- **แนวทาง:**
    - สร้างตัวระบุที่ไม่ซ้ำกันขนาด 128 บิต โดยแต่ละเซิร์ฟเวอร์สร้างขึ้นเองอย่างอิสระโดยใช้ UUID
    - UUID สามารถสร้างขึ้นได้อย่างอิสระโดยไม่ต้องมีการประสานงาน (coordination) ระหว่างเซิร์ฟเวอร์

        <p align="left">
        <img src="./images/uuid.png"  alt="UUID generator" width="600">
        </p>

- **ข้อดี:**
  - ไม่ต้องมีการประสานงานระหว่างเซิร์ฟเวอร์
  - ขยายระบบได้ง่ายตามจำนวน web server
- **ข้อเสีย:**
  - เกินขนาด 64 บิตที่ต้องการ
  - ID ไม่สามารถเรียงลำดับตามเวลาได้ และอาจไม่ใช่ตัวเลข (non-numeric)


### 3. Ticket Server
- **แนวทาง:** ใช้ฐานข้อมูลแบบรวมศูนย์ (centralized) เพื่อเพิ่มค่าและกำหนด ID

    <p align="left">
    <img src="./images/ticket-server.png"  alt="UUID generator" width="500">
    </p>

- **ข้อดี:**
  - ทำได้ง่ายสำหรับระบบขนาดเล็ก
  - สร้าง ID ที่เป็นตัวเลข
- **ข้อเสีย:**
  - เป็น single point of failure
  - มีความท้าทายด้านการซิงค์กันเมื่อมีเซิร์ฟเวอร์หลายตัว

### 4. แนวทาง Twitter Snowflake (Twitter Snowflake Approach)
- **แนวทาง:**

    <div style="margin-left:3rem">
      <img src="./images/twitter-snowflake.png"  alt="Snowflake approach" width="500">
    </div>
    <div style="margin-left:3rem">
      <img src="./images/snowflake-id-breakdown.png"  alt="Snowflake ID breakdow" width="500">
    </div>

    - แบ่ง ID ออกเป็นส่วนต่าง ๆ เพื่อรับประกันความไม่ซ้ำกันและการขยายระบบ
    - **Sign Bit (1 บิต):** เป็น `0` เสมอ ซึ่งอาจใช้แยกความแตกต่างระหว่างเลขแบบ signed และ unsigned
    - **Timestamp (41 บิต):** จำนวนมิลลิวินาทีนับตั้งแต่ epoch ที่กำหนดเอง (ค่าเริ่มต้นของ Twitter คือ `1288834974657` ซึ่งตรงกับวันที่ 4 พฤศจิกายน 2010 เวลา 01:42:54 UTC) ทำให้ ID เรียงลำดับตามเวลาได้
    - **Datacenter ID (5 บิต):** ระบุ data center ได้สูงสุด `2^5 = 32` แห่ง
    - **Machine ID (5 บิต):** ระบุเครื่อง (machine) ได้สูงสุด `2^5 = 32` เครื่องภายในแต่ละ data center
    - **Sequence Number (12 บิต):** ติดตาม ID ที่สร้างบนเครื่องหนึ่งภายในมิลลิวินาทีเดียวกัน รองรับได้สูงสุด `2^12 = 4096` ID ต่อมิลลิวินาที ค่า sequence จะถูกรีเซ็ตเป็น `0` ทุกมิลลิวินาที



- **ข้อดี:**
    - **ความสามารถในการขยายระบบ (Scalability):** รองรับ ID มากกว่า 10,000 ID ต่อวินาทีข้ามหลายเซิร์ฟเวอร์
    - **การเรียงลำดับตามเวลา (Time-Order):** รับประกันว่า ID สามารถเรียงลำดับตามเวลาได้
    - **การกระจายอำนาจ (Decentralization):** ไม่มี single point of failure


## ขั้นตอนที่ 4: ประเด็นเพิ่มเติมที่ต้องพิจารณา (Additional Considerations)
### 1. การซิงค์นาฬิกา (Clock Synchronization)
- **ความท้าทาย:** การสร้าง ID สมมติว่านาฬิกาของเซิร์ฟเวอร์ต่าง ๆ ซิงค์กันอยู่แล้ว
- **แนวทางแก้ไข:** ใช้ **Network Time Protocol (NTP)** เพื่อลดความคลาดเคลื่อน (drift) ให้น้อยที่สุด

### 2. การปรับขนาดของแต่ละส่วน (Section Length Tuning)
- ปรับขนาดของแต่ละส่วน (เช่น ลดจำนวนบิตของ sequence เพิ่มจำนวนบิตของ timestamp) ตาม use case

### 3. ความพร้อมใช้งานสูง (High Availability)
- ตัวสร้าง ID เป็นระบบที่สำคัญยิ่งยวด (mission-critical) และต้องมี fault tolerance
- ควรพิจารณาความซ้ำซ้อน (redundancy) และกลไก failover
