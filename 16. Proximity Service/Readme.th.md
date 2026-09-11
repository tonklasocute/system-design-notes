# บทที่ 16: บริการค้นหาสถานที่ใกล้เคียง (Proximity Service)

## บทนำ
**บริการค้นหาสถานที่ใกล้เคียง (Proximity Service)** ถูกออกแบบมาเพื่อค้นหาสถานที่ที่อยู่ใกล้เคียง เช่น ร้านอาหาร โรงแรม ปั๊มน้ำมัน และธุรกิจอื่น ๆ ฟังก์ชันนี้ถูกใช้งานในแอปพลิเคชันอย่าง **Google Maps** และ **Yelp** เพื่อช่วยให้ผู้ใช้ค้นพบสถานที่ที่อยู่ภายในรัศมีที่กำหนด


## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขต (Understanding the Problem and Establishing Scope)

### **ความต้องการเชิงฟังก์ชัน (Functional Requirements)**
1. **ค้นหาธุรกิจ (Search for businesses)** โดยอิงจากตำแหน่งของผู้ใช้ (latitude, longitude) และรัศมีการค้นหา
2. **อนุญาตให้เจ้าของธุรกิจ (business owners)** เพิ่ม แก้ไข หรือลบข้อมูลธุรกิจ (ไม่จำเป็นต้อง real-time)
3. **แสดงข้อมูลรายละเอียดของธุรกิจ** เมื่อมีการร้องขอ

### **ความต้องการเชิงไม่ใช่ฟังก์ชัน (Non-Functional Requirements)**
- **latency ต่ำ (Low latency)**: ผู้ใช้ควรได้รับการตอบกลับอย่างรวดเร็ว
- **ความเป็นส่วนตัวของข้อมูล (Data privacy)**: ต้องปฏิบัติตามกฎระเบียบ GDPR และ CCPA
- **ความพร้อมใช้งานสูง (High availability)**: รองรับปริมาณ traffic ที่พุ่งสูงในช่วงเวลาเร่งด่วน

### **การประมาณการแบบคร่าว ๆ (Back-of-the-Envelope Estimation)**
- ผู้ใช้งานที่ active ต่อวัน (daily active users) **100 ล้านคน**
- **ธุรกิจในระบบ 200 ล้านราย**
- **การคำนวณ Search QPS**:
  - ผู้ใช้ทำการค้นหา **5 ครั้งต่อวัน**
  - **Search QPS** = (100M × 5) / 86,400 ≈ **5,000 QPS**

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)

### **การออกแบบ API (API Design)**
#### **ค้นหาธุรกิจใกล้เคียง (Search Nearby Businesses)**
GET /v1/search/nearby

- **พารามิเตอร์ของ request**:
  - `latitude`: ตำแหน่ง latitude ของผู้ใช้
  - `longitude`: ตำแหน่ง longitude ของผู้ใช้
  - `radius`: รัศมีการค้นหา (ค่าเริ่มต้น: 5000 เมตร)

#### **API เกี่ยวกับธุรกิจ (Business APIs)**
| API Endpoint                     | คำอธิบาย                                      |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}`         | ดึงข้อมูลรายละเอียดของธุรกิจ                    |
| `POST /v1/businesses`             | เพิ่มธุรกิจใหม่                              |
| `PUT /v1/businesses/{id}`         | แก้ไขรายละเอียดธุรกิจ                         |
| `DELETE /v1/businesses/{id}`      | ลบธุรกิจออกจากระบบ               |


### **โมเดลข้อมูล (Data Model)**
- เนื่องจากปริมาณการอ่าน (read volume) สูงมาก เพราะมีสองฟีเจอร์ที่ถูกใช้งานบ่อยมาก ฐานข้อมูลเชิงสัมพันธ์ (relational database) อย่าง MySQL จึงเป็นตัวเลือกที่เหมาะสม
  - ค้นหาธุรกิจใกล้เคียง
  - ดูข้อมูลรายละเอียดของธุรกิจ

### **โครงสร้างข้อมูล (Data Schema)**
- ตารางฐานข้อมูลหลักคือตารางธุรกิจ (business table) และตาราง geospatial index
- ตารางธุรกิจประกอบด้วยข้อมูลรายละเอียดเกี่ยวกับธุรกิจ

### **สถาปัตยกรรมระบบระดับสูง (High-Level System Architecture)**
ระบบประกอบด้วยสองส่วนหลัก ได้แก่ Location-Based Service (LBS) และ business-related service

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="HLD" width="400" />
</div>

- **Location-Based Service (LBS)**: 
  - ประมวลผล query การค้นหาที่อิงตำแหน่ง (location-based search)
  - เป็นบริการที่มีการอ่านหนัก (read-heavy) และไม่มี request การเขียนข้อมูล
  - QPS สูงโดยเฉพาะในช่วงเวลาเร่งด่วนในพื้นที่ที่มีความหนาแน่นสูง และระบบเป็นแบบ stateless
- **Business Service**: จัดการ request สองประเภท
  - เจ้าของธุรกิจสร้าง แก้ไข หรือลบธุรกิจ
  - ลูกค้าดูข้อมูลรายละเอียดของธุรกิจ
- **Load Balancer**: กระจาย traffic ไปยัง LBS และ Business service
- **Database Cluster**: 
  - ใช้สถาปัตยกรรมแบบ **primary-replica** สำหรับ workload ที่มีการอ่านหนัก
  - อาจเกิดความคลาดเคลื่อนระหว่างข้อมูลที่อ่านโดย LBS กับข้อมูลที่เขียนโดย primary database
  - ความไม่สอดคล้องกันนี้ไม่ใช่ปัญหา เพราะข้อมูลธุรกิจไม่ได้ถูกอัปเดตแบบ real-time


---

## ขั้นตอนที่ 3: อัลกอริทึมสำหรับดึงข้อมูลธุรกิจใกล้เคียง (Algorithms for Fetching Nearby Businesses)

### **ตัวเลือกที่ 1: การค้นหาแบบสองมิติ (Two-Dimensional Search) (วิธีพื้นฐาน)**

<div style="margin-left:3rem">
    <img src="./images/2d-search.png" alt="2D" width="250" />
</div>

วิธีที่เข้าใจง่ายที่สุดคือการวาดวงกลมด้วยรัศมีที่กำหนดไว้ล่วงหน้า แล้วค้นหาธุรกิจทั้งหมดที่อยู่ภายในวงกลมนั้น

**SQL Query:**
```
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```
**ปัญหา:**
- **ไม่มีประสิทธิภาพ**: ต้อง scan ฐานข้อมูลทั้งหมด
- **ถูกจำกัดด้วย index แบบมิติเดียว** (latitude/longitude)

แนวทางปรับปรุงที่เป็นไปได้คือการสร้าง index บนคอลัมน์ longitude และ latitude ถึงแม้จะดีขึ้นเล็กน้อย แต่ก็ยังคงช้าอยู่มาก

### วิธีที่ดีกว่า
- ปัญหาของแนวทางก่อนหน้าคือ database index สามารถเพิ่มความเร็วในการค้นหาได้เพียงมิติเดียวเท่านั้น
- แนวทางที่เหมาะสมกว่าคือการแทนค่าข้อมูลสองมิติให้เป็นมิติเดียวโดยใช้ geospatial indexing
  - Hash: Even Grid, Geo Hash
  - Tree: Quadtree, Google S2, RTree

  <div style="margin-left:3rem">
    <img src="./images/geospatial-index-types.png" alt="2D" width="500" />
  </div>


### **ตัวเลือกที่ 2: การแบ่งกริดแบบเท่ากัน (Evenly Divided Grid)**

  <div style="margin-left:3rem">
    <img src="./images/even-grid.png" alt="Even Grid" width="400" />
  </div>

- **แบ่งโลกออกเป็นกริดขนาดคงที่ (fixed-size grids)**
- **ปัญหา**: การกระจายตัวของธุรกิจไม่สม่ำเสมอ (ความหนาแน่นสูงในเมือง แต่เบาบางในพื้นที่ชนบท)

### **ตัวเลือกที่ 3: Geohash**
- แบ่งโลกออกเป็นสี่ส่วน (quadrant) ตามแนวเส้นเมริเดียนหลัก (prime meridian) และเส้นศูนย์สูตร (equator) จากนั้นแบ่งแต่ละกริดออกเป็นสี่กริดย่อยอีกครั้ง
- แต่ละกริดสามารถแทนค่าได้โดยการสลับระหว่างบิตของ longitude และ latitude
- ทำซ้ำการแบ่งย่อยนี้ไปเรื่อย ๆ

  <div style="margin-left:3rem">
    <img src="./images/geohash.png" alt="Geohash" width="300" />
    <img src="./images/geohash-1.png" alt="Geohash" width="285" />
  </div>


- **เข้ารหัส latitude และ longitude ให้เป็นสตริงตัวอักษรและตัวเลข (alphanumeric string) เดียว** โดยมีความละเอียด (precision) ทั้งหมด 12 ระดับ
- **โครงสร้างกริดแบบลำดับชั้น (hierarchical grid structure)** ช่วยให้การค้นหาเป็นไปอย่างมีประสิทธิภาพ
- ความละเอียดที่เหมาะสมจะถูกเลือกโดยใช้ความยาว geohash ที่น้อยที่สุดตามตาราง
  <div style="margin-left:3rem">
    <img src="./images/geohash-radius-mapping.png" alt="Geohash Radius" width="400" />
  </div>
- Geohash รับประกันว่ายิ่ง prefix ที่ใช้ร่วมกันระหว่าง geohash สองค่ายาวเท่าไร geohash ทั้งสองก็จะยิ่งอยู่ใกล้กันมากเท่านั้น

- **ความท้าทาย**:
  <div style="margin-left:3rem">
    <img src="./images/boundary-issue.png" alt="Boundary Issue" width="300" />
  </div>

  - **ปัญหาขอบเขต (Boundary issues)** (ธุรกิจที่อยู่ใกล้ขอบกริดอาจถูกตกหล่นจากผลการค้นหา)
    - ตำแหน่งสองแห่งอาจอยู่ใกล้กันมาก แต่ไม่มี prefix ร่วมกันเลย (อาจอยู่คนละฝั่งของเส้นศูนย์สูตร)
    - ตำแหน่งสองแห่งอาจมี prefix ร่วมกันยาว แต่กลับอยู่คนละ geohash
  - วิธีแก้ไข: จำเป็นต้องค้นหากริดข้างเคียงด้วย


### **ตัวเลือกที่ 4: Quadtree**

  Quadtree เป็นโครงสร้างข้อมูลแบบต้นไม้ (tree) ที่แบ่งพื้นที่สองมิติออกเป็นสี่ส่วนซ้ำ ๆ กันแบบ recursive โดยแต่ละ internal node จะมีลูกโหนด (child) ทั้งหมดสี่โหนด ซึ่งแทนพื้นที่ย่อยทั้งสี่ส่วนของพื้นที่นั้น
  - Quadtree เป็นโครงสร้างข้อมูลที่อยู่ในหน่วยความจำ (in-memory) และรันอยู่บนเซิร์ฟเวอร์ LBS แต่ละตัว โดยถูกสร้างขึ้นในช่วงเวลาที่เซิร์ฟเวอร์เริ่มทำงาน (server startup)

  <div style="margin-left:3rem">
    <img src="./images/quadtree.png" alt="Quadtree" width="500" />
  </div>

  - root node จะถูกแบ่งย่อยออกเป็น 4 ส่วนซ้ำ ๆ กันแบบ recursive จนกว่าจะไม่มี node ใดเหลือธุรกิจมากกว่าจำนวน x ที่กำหนด (ในที่นี้คือ 100)

  <div style="margin-left:3rem">
    <img src="./images/building-quadtree.png" alt="Building Quadtree" width="500" />
  </div>

- Quadtree index ใช้หน่วยความจำไม่มากนัก (โดยทั่วไปอยู่ในระดับ GB) และสามารถเก็บไว้ในเซิร์ฟเวอร์เดียวได้อย่างง่ายดาย
- เนื่องจาก time complexity ในการสร้าง tree คือ nlogn จึงอาจใช้เวลาสองสามนาทีในการสร้าง tree
- **มีประสิทธิภาพสำหรับ query แบบ k-nearest search** (เช่น ค้นหาปั๊มน้ำมันที่ใกล้ที่สุด)

  <div style="margin-left:3rem">
    <img src="./images/realworld-quadtree.png" alt="Real World Quadtree" width="400" />
  </div>

#### ข้อพิจารณาด้านการปฏิบัติงาน (Operational considerations)
 - สำหรับธุรกิจประมาณ 200 ล้านราย อาจใช้เวลาสองสามนาทีในการสร้าง quadtree ตอนที่เซิร์ฟเวอร์เริ่มทำงาน
 - ในระหว่างที่กำลังสร้าง quadtree เซิร์ฟเวอร์จะยังไม่สามารถให้บริการ traffic ได้ ดังนั้นการ release เวอร์ชันใหม่ควรทำแบบทยอยไปยังกลุ่มย่อยของเซิร์ฟเวอร์ (incrementally)
 - เมื่อมีการอัปเดตหรือเพิ่มธุรกิจใหม่ วิธีที่ง่ายที่สุดคือการสร้าง quadtree ขึ้นมาใหม่แบบทยอย (incrementally rebuild) (ซึ่งนำไปสู่การ invalidate cache จำนวนมาก)
 - นอกจากนี้ยังสามารถอัปเดต quadtree แบบ on-the-fly ได้ แต่มีความซับซ้อนในการ implement มากกว่า (ต้องใช้กลไก locking)

### **ตัวเลือกที่ 5: Google S2**
Google S2 แมปทรงกลม (sphere) ไปเป็น index มิติเดียวโดยอิงจาก Hilbert curve จุดสองจุดที่อยู่ใกล้กันบน Hilbert curve จะอยู่ใกล้กันในพื้นที่มิติเดียวด้วย


  <div style="margin-left:3rem">
    <img src="./images/hilbert-curve.png" alt="Hilbert curve" width="300" />
    <img src="./images/geofence.png" alt="Geofence" width="355" />
  </div>

- **แบ่งโลกออกเป็น cell ขนาดเล็กโดยใช้ Hilbert curve**
- เหมาะมากสำหรับการทำ geofencing เพราะสามารถครอบคลุมพื้นที่รูปร่างใด ๆ ก็ได้ในหลายระดับความละเอียด
- Geofencing ยังช่วยให้สามารถกำหนดพารามิเตอร์ที่ล้อมรอบพื้นที่ที่สนใจได้
- ข้อได้เปรียบอีกอย่างหนึ่งคือ แทนที่จะมีระดับความละเอียดคงที่ เราสามารถกำหนดค่า min, max level และจำนวน cell สูงสุดใน S2 ได้


## เปรียบเทียบข้อดีข้อเสีย (Tradeoff Comparison)

#### Geohash
- ใช้งานและ implement ได้ง่าย ไม่จำเป็นต้องสร้าง/สร้างใหม่ tree
- รองรับผลลัพธ์ที่มีรัศมีคงที่ (fixed radius)
- การอัปเดต index ทำได้ง่าย
- ไม่สามารถปรับขนาดกริดแบบ dynamic ตามความหนาแน่นของประชากรได้

#### Quadtree
- Implement ยากกว่าเล็กน้อย
- รองรับการดึงข้อมูลธุรกิจแบบ k-nearest
- สามารถปรับขนาดกริดแบบ dynamic ตามความหนาแน่นของประชากรได้
- การอัปเดต index มีความซับซ้อนมากกว่า เนื่องจากอาจต้องสร้าง tree ทั้งหมดขึ้นมาใหม่

---

## ขั้นตอนที่ 4: การขยายฐานข้อมูลและกลยุทธ์การแคช (Scaling the Database and Caching Strategy)

### **การขยายตารางธุรกิจ (Scaling the Business Table)**
- **การทำ sharding โดยใช้ business ID** ช่วยให้การกระจายข้อมูลมีความสม่ำเสมอ
- แต่ละธุรกิจมีแถวข้อมูล (row) แยกกันในตาราง

| Geohash | Business ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **การขยาย Geospatial Index (Scaling the Geospatial Index)**
- อาจไม่เหมาะที่จะทำ sharding กับตาราง geohash ในกรณีนี้ ข้อมูลทั้งหมดสามารถใส่ในเซิร์ฟเวอร์เดียวได้ จึงไม่มีเหตุผลทางเทคนิคที่จะต้องทำ sharding
- แนวทางที่ดีกว่าคือการมี read-replica เพื่อช่วยรองรับภาระการอ่าน (read load)



---

### **กลยุทธ์การแคช (Cache Strategy)**
ตัวเลือกที่ชัดเจนที่สุดสำหรับ cache key คือพิกัดตำแหน่ง (location coordinate) อย่างไรก็ตามมันมีปัญหาอยู่บ้าง:
 - พิกัดตำแหน่งจาก GPS ไม่แม่นยำ
 - ผู้ใช้สามารถเคลื่อนที่ได้ ทำให้พิกัดตำแหน่งเปลี่ยนแปลง
 - key ที่ดีกว่าคือ geohash

| Cache Key  | Cache Value |
|------------|------------|
| `geohash`  | รายการ business ID ในกริดนั้น |
| `business_id` | รายละเอียดธุรกิจ (ชื่อ, ที่อยู่, รีวิว, ฯลฯ) |

---

## ขั้นตอนที่ 5: กลยุทธ์การ Deploy และสถาปัตยกรรมสุดท้าย (Deployment Strategy and Final Architecture)

### **Region และ Availability Zone**
- Deploy LBS และ Business Service **ในหลาย region**

### **การจัดการการอัปเดตแบบ Real-Time (Handling Real-Time Updates)**
- **การอัปเดตข้อมูลธุรกิจถูกประมวลผลแบบ batch รายวัน**

### **สถาปัตยกรรมระบบสุดท้าย (Final System Architecture)**


  <div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="Final Design" width="500" />
  </div>


อัลกอริทึมสุดท้ายมีลักษณะดังนี้:

## ขั้นตอนในการดึงข้อมูลธุรกิจใกล้เคียง (Steps to Retrieve Nearby Businesses)
1. **การร้องขอของผู้ใช้ (User Request):**  
   - ผู้ใช้ค้นหาร้านอาหารภายในระยะ **500 เมตร**  
   - ไคลเอนต์ส่ง **latitude (37.776720), longitude (-122.416730) และ radius (500m)** ไปยัง **load balancer**

2. **การส่งต่อ Request (Request Forwarding):**  
   - **load balancer (LB)** ส่งต่อ request ไปยัง **Location-Based Service (LBS)**

3. **การคำนวณ Geohash (Geohash Calculation):**  
   - LBS กำหนด **ความยาวของ geohash** ที่สอดคล้องกับรัศมีที่ต้องการ  
   - โดยใช้ตารางอ้างอิง **500 เมตร สอดคล้องกับความยาว geohash = 6**

4. **การดึง Geohash ข้างเคียง (Fetching Neighboring Geohashes):**  
   - LBS คำนวณ **geohash ข้างเคียง (neighboring geohashes)** เพื่อรวมพื้นที่ใกล้เคียงเข้ามาด้วย  
   - ผลลัพธ์ที่ได้คือรายการ:  
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```

5. **การดึง Business ID จาก Redis (Fetching Business IDs from Redis):**  
   - สำหรับแต่ละ geohash ในรายการ LBS จะ query ไปยัง **Geohash Redis server** เพื่อดึง **business ID**  
   - ใช้การ query แบบขนาน (parallel) เพื่อลด latency ให้น้อยที่สุด

6. **การดึงข้อมูลและจัดอันดับธุรกิจ (Retrieving & Ranking Businesses):**  
   - LBS ดึง **รายละเอียดธุรกิจแบบเต็ม** จาก **Business Info Redis server**  
   - ธุรกิจจะถูก **เรียงลำดับตามระยะทาง** จากตำแหน่งของผู้ใช้  
   - **ผลลัพธ์ที่จัดอันดับแล้ว** จะถูกส่งกลับไปยังไคลเอนต์

## การปรับแต่งที่สำคัญ (Key Optimizations)
- **การเรียก Redis แบบขนาน (Parallel Redis Calls)**: ลดเวลาตอบสนอง  
- **Geohash Indexing**: รับประกันการทำ spatial query ที่มีประสิทธิภาพ  
- **Caching**: เพิ่มความเร็วในการค้นหาและดึงข้อมูลธุรกิจ  

วิธีการนี้รับประกันการดึงข้อมูลธุรกิจใกล้เคียงตำแหน่งของผู้ใช้ที่มี **latency ต่ำและสามารถขยายตัวได้ (scalable)**

---

### **การเลือกวิธีการทำ Indexing ที่ดีที่สุด (Choosing the Best Indexing Method)**
| วิธีการทำ Indexing | ข้อดี | ข้อเสีย |
|----------------|------|------|
| **Geohash** | Implement ได้ง่าย มีประสิทธิภาพสำหรับการค้นหาตำแหน่งใกล้เคียง | มีปัญหาขอบเขต ขนาดกริดคงที่ |
| **Quadtree** | ปรับตัวตามความหนาแน่นได้แบบ dynamic รองรับ query แบบ k-nearest | ซับซ้อนกว่า ต้องมีการทำ tree rebalancing |
| **Google S2** | ทำ geofencing ขั้นสูงได้ ถูกใช้งานใน Google Maps | Implement ได้ยากกว่า |

---

## แหล่งอ้างอิง (References)
1. [Geohash Algorithm](https://www.movable-type.co.uk/scripts/geohash.html)
2. [Quadtree Indexing](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 Geometry](https://s2geometry.io/)

