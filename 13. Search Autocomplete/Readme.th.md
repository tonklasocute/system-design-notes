# บทที่ 13: ออกแบบระบบ Search Autocomplete (Design a Search Autocomplete System)

## บทนำ
Autocomplete หรือที่รู้จักกันในชื่อ typeahead หรือ incremental search ทำหน้าที่แสดงคำแนะนำแบบ real-time ให้กับผู้ใช้ในขณะที่กำลังพิมพ์ในช่องค้นหา ระบบต้องนำส่งคำแนะนำ top-k ที่เกี่ยวข้องและได้รับความนิยมได้อย่างมีประสิทธิภาพ โดยอ้างอิงจากข้อมูล query ในอดีต

### ฟีเจอร์สำคัญ (Key Features)
- แนะนำผลลัพธ์แบบ autocomplete ได้สูงสุด **5 รายการ**
- อ้างอิงจาก **ความนิยมของ query (query popularity)** (ความถี่)
- รองรับเฉพาะ **ตัวอักษรภาษาอังกฤษพิมพ์เล็ก (lowercase)**
- เวลาตอบสนองที่รวดเร็ว (<100 ms) และสามารถขยายตัวได้ (scalable)

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหา (Understanding the Problem)

### ข้อกำหนด (Requirements)
1. **คำแนะนำแบบ Real-Time (Real-Time Suggestions):** แสดงผลลัพธ์ที่เกี่ยวข้องขณะที่ผู้ใช้กำลังพิมพ์
2. **ผลลัพธ์ Top-k (Top-k Results):** ส่งคืนผลลัพธ์สูงสุด 5 รายการ เรียงตามความนิยม
3. **ความสามารถในการขยายตัว (Scalability):** รองรับ DAU จำนวน **10 ล้านคน** โดยมี peak QPS ที่ **48,000**
4. **ความพร้อมใช้งานสูง (High Availability):** รองรับความล้มเหลวได้โดยไม่ทำให้ระบบหยุดทำงาน
5. **การเติบโตของข้อมูล (Data Growth):** รองรับการเติบโตของพื้นที่จัดเก็บข้อมูล query ใหม่ที่ **0.4 GB ต่อวัน**

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)
ในระดับสูง ระบบถูกแบ่งออกเป็นสองบริการ:
1. **Data Gathering Service:**
    - รวบรวม query ของผู้ใช้และรวมข้อมูล (aggregate) เพื่อวิเคราะห์ความถี่แบบ real-time
    - การประมวลผลแบบ real-time ไม่เหมาะกับชุดข้อมูลขนาดใหญ่ อย่างไรก็ตามถือเป็นจุดเริ่มต้นที่ดี


2. **Query Service:** ให้บริการคำแนะนำ top-k ตามข้อมูลที่ผู้ใช้ป้อนเข้ามา

---

### Data Gathering Service
<div style="margin-left:3rem">
    <img src="./images/data-gathering.png" alt="Data Gathering" width="600">
</div>

- รวมข้อมูล query จาก analytics log และอัปเดตตาราง frequency
- ประมวลผลข้อมูลย้อนหลังทุกสัปดาห์เพื่อสร้าง **trie** (prefix tree)




### Query Service
<div style="margin-left:3rem">
    <img src="./images/frequency-table.png" alt="Frequency Table" width="400">
    <img src="./images/basic-search-suggestions.png" alt="Search Suggestions" width="360">
</div>

- ใช้ตาราง frequency จาก data gathering service
- ประมวลผล input ของผู้ใช้และดึงคำแนะนำ top-k จากตาราง frequency โดยใช้ trie
- ปรับให้เหมาะสมสำหรับการค้นหาที่รวดเร็ว โดยใช้แคชและโครงสร้างข้อมูลที่มีประสิทธิภาพ
- ตัวอย่างเช่น เมื่อผู้ใช้พิมพ์ "tw" ในช่องค้นหา query ที่ถูกค้นหามากที่สุด 5 อันดับแรกต่อไปนี้จะถูกแสดงขึ้นมา


---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### โครงสร้างข้อมูล Trie (Trie Data Structure)
**Trie** เป็นโครงสร้างข้อมูลลักษณะคล้ายต้นไม้ (tree-like) ที่ใช้จัดเก็บและดึง query string ได้อย่างมีประสิทธิภาพ

#### ฟีเจอร์สำคัญ (Key Features)
1. **การจัดเก็บแบบกระชับ (Compact Storage):** แทน prefix ในรูปแบบลำดับชั้น (hierarchically) เพื่อลดความซ้ำซ้อนให้น้อยที่สุด
2. **ข้อมูลความถี่ (Frequency Information):** จัดเก็บความนิยมของ query ไว้ที่แต่ละ node

4. **ขั้นตอนในการดึง query ที่ถูกค้นหามากที่สุด top k รายการ**
   <div style="margin-left:3rem">
      <img src="./images/trie-structure.png" alt="Trie Structure" width="500">
   </div>

    - ค้นหา node ของ prefix
    - ไล่สำรวจ (traverse) subtree จาก node ของ prefix เพื่อดึง children ที่ถูกต้องทั้งหมด
    - เรียงลำดับ children และดึง top k


3. **การปรับปรุงประสิทธิภาพ (Optimizations):**
   - แคชคำที่ถูกค้นหามากที่สุด top-k ไว้ที่แต่ละ node เพื่อเร่งความเร็วในการดึงข้อมูล และหลีกเลี่ยงการไล่สำรวจ trie ทั้งหมด

        <img src="./images/cached-trie.png" alt="Cached Trie" width="600">

   - จำกัดความยาวของ prefix เพื่อลดขอบเขตการค้นหา เนื่องจากผู้ใช้ไม่ค่อยพิมพ์ query ที่ยาวมาก (เช่น จำกัดไว้ที่ 50 ตัวอักษร)

#### การดำเนินการกับ Trie (Trie Operations)
1. **Create:**
    - สร้างขึ้นทุกสัปดาห์โดยใช้ข้อมูล query ที่รวมไว้แล้ว
    - แหล่งข้อมูลมาจาก Analytics Log/DB
2. **Update:** แทบไม่มีการอัปเดตแบบ real-time การอัปเดตรายสัปดาห์จะแทนที่ข้อมูลเก่า
3. **Delete:**
      <div style="margin-left:3rem">
         <img src="./images/delete-kv.png" alt="Delete KV" width="500">
      </div>

    - ตัวกรอง (filter) จะลบคำแนะนำที่ไม่ต้องการหรือเป็นอันตราย (เช่น hate speech)
    - การมีชั้น filter ทำให้เรามีความยืดหยุ่นในการลบผลลัพธ์ตามกฎการกรองที่แตกต่างกันได้
    - คำแนะนำที่ไม่ต้องการจะถูกลบออกจากฐานข้อมูลจริง ๆ แบบ asynchronous


---

### ลำดับการประมวลผล Query (Query Processing Flow)
1. **การค้นหา Prefix (Prefix Search):**
   - ระบุ node ของ prefix ที่ตรงกับ input ของผู้ใช้
   - ไล่สำรวจ subtree เพื่อรวบรวมคำแนะนำที่ถูกต้อง
2. **การเรียงลำดับ Top-k (Top-k Sorting):**
   - แคชคำแนะนำ top-k ไว้ที่แต่ละ node เพื่อลดภาระในการเรียงลำดับให้น้อยที่สุด
3. **การสร้างผลลัพธ์ (Response Construction):**
   - สร้างผลลัพธ์โดยใช้ข้อมูลที่แคชไว้ เพื่อให้ได้เวลาตอบสนองที่รวดเร็ว

---

### การปรับปรุงประสิทธิภาพ (Optimizations)
1. **แคชที่แต่ละ Node (Cache at Each Node):**
   - จัดเก็บ query top-k เพื่อหลีกเลี่ยงการไล่สำรวจซ้ำซ้อน
2. **จำกัดความยาวของ Prefix (Limit Prefix Length):**
   - จำกัดความยาวของ prefix ให้อยู่ในค่าที่น้อย (เช่น 50 ตัวอักษร) เพื่อการค้นหาที่รวดเร็วขึ้น
3. **AJAX Requests:**
   - ใช้ request แบบ asynchronous ที่มีน้ำหนักเบาสำหรับการตอบสนองแบบ real-time
4. **การแคชในเบราว์เซอร์ (Browser Caching):**
   - บันทึกผลลัพธ์ autocomplete ไว้ในแคชของเบราว์เซอร์สำหรับคำที่ถูกค้นหาบ่อย

---

### Pipeline การรวบรวมข้อมูล (Data Gathering Pipeline)
ในการออกแบบระดับสูง ทุกครั้งที่ผู้ใช้พิมพ์ query การค้นหา ข้อมูลจะถูกอัปเดตแบบ real-time แนวทางนี้ไม่เหมาะกับการใช้งานจริง
- ผู้ใช้อาจป้อน query นับพันล้านครั้งต่อวัน การอัปเดต trie ทุกครั้งที่มี query จึงไม่สามารถทำได้จริง
- คำแนะนำอันดับต้น ๆ อาจไม่เปลี่ยนแปลงมากนักเมื่อ trie ถูกสร้างขึ้นแล้ว


#### การออกแบบที่ปรับปรุงแล้ว (Updated Design)

<div style="margin-left:3rem">
   <img src="./images/data-gathering-flow.png" alt="Updated Data Gathering Flow" width="600">
</div>

1. **Analytics Logs:**
   - จัดเก็บข้อมูล query ดิบไว้เป็น log เพื่อรวมข้อมูลรายสัปดาห์
   - Log เป็นแบบ append-only และไม่มีการทำ index
2. **Aggregators:**
   - ประมวลผล log ให้เป็นตาราง frequency ที่เหมาะสำหรับการสร้าง trie
   - สำหรับแอปพลิเคชันแบบ real-time เช่น Twitter ให้รวมข้อมูลในช่วงเวลาที่สั้นกว่า
   - สำหรับกรณีอื่น ๆ การรวมข้อมูลที่ความถี่ต่ำกว่า เช่น สัปดาห์ละครั้ง ก็เพียงพอแล้ว
3. **Workers:**
   - เซิร์ฟเวอร์แบบ asynchronous จะสร้าง trie ขึ้นใหม่และจัดเก็บไว้ในที่จัดเก็บข้อมูลถาวร (persistent storage)
4. **ทางเลือกในการจัดเก็บข้อมูล (Storage Options):**
    - **Trie Cache:** Trie Cache คือระบบแคชแบบกระจาย (distributed cache) ที่เก็บ trie ไว้ในหน่วยความจำเพื่อการอ่านที่รวดเร็ว
    - **Trie DB**
        1. **Document Store (เช่น MongoDB):** เนื่องจาก trie ใหม่ถูกสร้างขึ้นทุกสัปดาห์ เราสามารถทำการ snapshot เป็นระยะ, serialize ข้อมูล, และจัดเก็บข้อมูลที่ serialize แล้วไว้ในฐานข้อมูลอย่าง MongoDB
        2. **Key-Value Store:**
            - Map prefix เข้ากับข้อมูลของ node เพื่อการเข้าถึงที่รวดเร็ว
            - ทุก prefix ใน trie จะถูก map เข้ากับ key หนึ่งใน hash table
            - ข้อมูลในแต่ละ trie node จะถูก map เข้ากับ value หนึ่งใน hash table

                <img src="./images/trie-db.png" alt="Trie DB" width="600">
---

### ความสามารถในการขยายตัว (Scalability)
1. **Sharding:**
   - กระจาย trie node ไปยังเซิร์ฟเวอร์ต่าง ๆ ตามช่วงของ prefix (เช่น `a-m`, `n-z`)
   - ทำ sharding เพิ่มเติมภายใน prefix เพื่อปรับสมดุลการกระจายข้อมูลที่ไม่สม่ำเสมอ (เช่น `aa-ag`, `ah-an`)
2. **Load Balancing:**
   <div style="margin-left:3rem">
      <img src="./images/sharding.png" alt="Sharding" width="400">
   </div>

   - ใช้ shard map manager เพื่อนำ request ไปยังเซิร์ฟเวอร์ที่เหมาะสม


---

## ขั้นตอนที่ 4: ฟีเจอร์ขั้นสูง (Advanced Features)

### การรองรับหลายภาษา (Multi-Language Support)
1. **ตัวอักษร Unicode (Unicode Characters):** ใช้ Unicode เพื่อรองรับภาษาอื่นนอกเหนือจากภาษาอังกฤษ
2. **Trie เฉพาะประเทศ (Country-Specific Tries):** สร้าง trie แยกต่างหากสำหรับแต่ละประเทศหรือภูมิภาค

### Query ที่กำลังเป็นกระแส (Trending Queries)
- จัดการกับ event แบบ real-time ด้วยการอัปเดต trie node แบบไดนามิก หรือให้น้ำหนัก (weight) กับ query ล่าสุดมากขึ้น
