# บทที่ 6: การออกแบบ Key-Value Store (Design a Key-Value Store)

## บทนำ
**Key-Value Store** คือฐานข้อมูลแบบไม่ใช่เชิงสัมพันธ์ (non-relational database) ประเภทหนึ่ง ที่จัดเก็บข้อมูลในรูปแบบคู่ key-value โดยแต่ละ key จะมีลักษณะเฉพาะตัว (unique) และค่าต่าง ๆ จะถูกเข้าถึงผ่าน key เหล่านี้ บทนี้จะอธิบายอย่างละเอียดว่าจะออกแบบ key-value store แบบ distributed ที่รองรับการขยายระบบ (scalable) และมีความพร้อมใช้งานสูง (high-availability) ได้อย่างไร โดยรองรับ operation เช่น:
- `put(key, value)` สำหรับการเพิ่มข้อมูล
- `get(key)` สำหรับการดึงข้อมูล

### คุณลักษณะของการออกแบบ (Characteristics of the Design)
- คู่ key-value ขนาดเล็ก (น้อยกว่า 10 KB)
- รองรับข้อมูลขนาดใหญ่ (big data) พร้อมความพร้อมใช้งานสูงและความสามารถในการขยายระบบ
- การขยายระบบอัตโนมัติ (automatic scaling) และ consistency ที่ปรับแต่งได้ (tunable)
- Latency ต่ำ

---

## Key-Value Store แบบเซิร์ฟเวอร์เดียว (Single Server Key-Value Store)
### การ Implement
- ใช้ **hash table** เพื่อจัดเก็บคู่ key-value ในหน่วยความจำ
- การปรับปรุงประสิทธิภาพ (Optimizations):
  - การบีบอัดข้อมูล (data compression)
  - จัดเก็บข้อมูลที่ถูกเข้าถึงไม่บ่อยนักไว้บนดิสก์

### ข้อจำกัด (Limitation)
หน่วยความจำของเซิร์ฟเวอร์เดียวมีจำกัด จึงจำเป็นต้องใช้ **แนวทางแบบ distributed** เพื่อให้ขยายระบบได้

---

## Key-Value Store แบบ Distributed (Distributed Key-Value Store)
**Distributed Key-Value Store** จะแบ่งข้อมูล (partition) ไปยังเซิร์ฟเวอร์หลายตัว และต้องจัดการกับข้อแลกเปลี่ยน (trade-off) ตามที่ระบุไว้ใน **CAP theorem**

### CAP Theorem
1. **Consistency:** ไคลเอนต์ทุกตัวเห็นข้อมูลชุดเดียวกันในเวลาเดียวกัน
2. **Availability:** ระบบตอบสนองต่อทุก request แม้ว่า node บางตัวจะล่ม
3. **Partition Tolerance:** ระบบยังคงทำงานต่อไปได้แม้เกิด network partition

**ข้อแลกเปลี่ยน:** ตาม CAP theorem มีเพียงสองใน สามคุณสมบัติเท่านั้นที่จะบรรลุพร้อมกันได้

<p align="center">
  <img src="./images/cap.png" alt="CAP" width="400">
</p>

#### ประเภทของระบบ (System Types):
- **ระบบ CP:** เลือก Consistency และ partition tolerance โดยยอมสละ availability (เช่น ระบบธนาคาร)
- **ระบบ AP:** เลือก Availability และ partition tolerance โดยยอมสละ consistency (เช่น eventual consistency)
- **ระบบ CA:** เลือก Consistency และ Availability โดยยอมสละ partition tolerance

    **เนื่องจากความล้มเหลวของเครือข่ายเป็นสิ่งที่หลีกเลี่ยงไม่ได้ ระบบ distributed จึงต้องทนต่อ network partition ดังนั้นระบบแบบ CA จึงไม่สามารถมีอยู่จริงในแอปพลิเคชันโลกจริงได้**

    ในระบบ distributed การเกิด partition เป็นสิ่งที่หลีกเลี่ยงไม่ได้ เมื่อเกิด partition ขึ้น เราต้องเลือกระหว่าง consistency กับ availability ตัวอย่างเช่น หาก node n3 ล่ม
    ข้อมูลใด ๆ ที่ถูกเขียนไปยัง node n1 หรือ n2 จะไม่สามารถถูกส่งต่อ (propagate) ไปยัง n3 ได้ ในทางกลับกัน หากมีการเขียนข้อมูลไปยัง n3 แต่ยังไม่ถูกส่งต่อไปยัง n1 และ n2 node n1 และ n2 จะมีข้อมูลที่ล้าสมัย (stale)

    <p align="center">
    <img src="./images/server-down.png"  alt="Server down" width="400">
    </p>
    
- หากเลือกใช้ระบบ CP เราต้องบล็อก write operation ทั้งหมดไปยัง n1 และ n2 เพื่อหลีกเลี่ยงความไม่สอดคล้องกันของข้อมูล
- หากเลือกใช้ระบบ AP ระบบจะยังคงรับ read operation ต่อไป แม้ว่าอาจส่งคืนข้อมูลที่ล้าสมัยก็ตาม
สำหรับการเขียน n1 และ n2 จะยังคงรับการเขียนต่อไป
และข้อมูลจะถูก sync ไปยัง n3 เมื่อ network partition ได้รับการแก้ไขแล้ว

---

## คอมโพเนนต์ของระบบ (System Components)
### 1. การแบ่งพาร์ทิชันข้อมูล (Data Partitioning)
- **เทคนิค:** ใช้ Consistent Hashing เพื่อกระจายข้อมูลไปยังเซิร์ฟเวอร์หลายตัวอย่างสม่ำเสมอ
- **ข้อดี:**
  - ขยายระบบอัตโนมัติเมื่อมีการเพิ่ม/ลบเซิร์ฟเวอร์
  - รองรับความหลากหลาย (heterogeneity) ผ่าน virtual node โดยจำนวน virtual node ของเซิร์ฟเวอร์หนึ่งจะแปรผันตรงกับความสามารถ (capacity) ของเซิร์ฟเวอร์นั้น

### 2. การทำสำเนาข้อมูล (Data Replication)
- ทำสำเนาข้อมูลไปยังเซิร์ฟเวอร์ `N` ตัว เพื่อความพร้อมใช้งานสูง
- เซิร์ฟเวอร์ N ตัวถูกเลือกโดยการเดินตามเข็มนาฬิกา (clockwise) จากตำแหน่งของเซิร์ฟเวอร์ และเลือกเซิร์ฟเวอร์ N ตัวแรกบน ring เพื่อจัดเก็บสำเนาข้อมูล ควรวาง replica ไว้ใน data center ที่แตกต่างกันเพื่อเพิ่มความน่าเชื่อถือในกรณีที่ใช้ virtual node

    <p align="center">
    <img src="./images/data-replication.png" alt="Data replication" width="300">
    </p>

### 3. Consistency
เนื่องจากข้อมูลถูกทำสำเนาไว้ที่หลาย node จึงต้องมีการ synchronize ระหว่าง replica ต่าง ๆ
- **Quorum Consensus:**
  - `N`: จำนวน replica ทั้งหมด
  - `W`: ขนาดของ write quorum สำหรับการเขียนที่จะถือว่าสำเร็จ การเขียนนั้นต้องได้รับการยืนยัน (acknowledge) จาก replica จำนวน W ตัว
  - `R`: ขนาดของ read quorum สำหรับการอ่านที่จะถือว่าสำเร็จ การอ่านต้องรอการตอบกลับจาก replica อย่างน้อย R ตัว
  - **กฎ:** `W + R > N` จะรับประกัน strong consistency
  - การกำหนดค่า W, R และ N เป็นข้อแลกเปลี่ยนโดยทั่วไประหว่าง latency กับ consistency

    <p align="center">
    <img src="./images/quorum-consensus.png"   alt="Quorum consensus" width="400">
    </p>
    
    - หาก R = 1 และ W = N ระบบจะถูก optimize ให้อ่านได้เร็ว
    - หาก W = 1 และ R = N ระบบจะถูก optimize ให้เขียนได้เร็ว
    - หาก W + R > N จะรับประกัน strong consistency (โดยทั่วไป N = 3, W = R = 2)
    - หาก W + R <= N จะไม่รับประกัน strong consistency

- **โมเดล (Models):**
  - **Strong Consistency:** operation การอ่านจะคืนค่าที่ตรงกับผลลัพธ์ของข้อมูลที่ถูกเขียนล่าสุด
  - **Weak Consistency:** operation การอ่านครั้งถัดไปอาจไม่เห็นค่าที่อัปเดตล่าสุด
  - **Eventual Consistency:** เมื่อเวลาผ่านไปนานเพียงพอ การอัปเดตทั้งหมดจะถูกส่งต่อจนครบ และ replica ทั้งหมดจะสอดคล้องกัน


### 4. การแก้ไขความไม่สอดคล้องกัน (Inconsistency Resolution)
การทำ replication ให้ความพร้อมใช้งานสูง แต่ก็ทำให้เกิดความไม่สอดคล้องกันระหว่าง replica การทำ versioning และ
vector clock ถูกนำมาใช้เพื่อแก้ปัญหาความไม่สอดคล้องกัน
- **Versioning:**
    - ใช้ **vector clock** เพื่อติดตามเวอร์ชันของข้อมูลและแก้ไขความขัดแย้ง (conflict)
    - Versioning หมายถึงการปฏิบัติต่อการแก้ไขข้อมูลแต่ละครั้งเสมือนเป็นเวอร์ชันใหม่ที่ไม่เปลี่ยนแปลง (immutable) ของข้อมูล
        <div>
        <img src="./images/consistent-server.png"   alt="Consisten hashing" width="400">
        <img src="./images/inconsistent-server.png"   alt="Inconsistent server" height="230">
        </div>
    
    - เซิร์ฟเวอร์ 1 เปลี่ยนชื่อ และเซิร์ฟเวอร์ 2 ก็เปลี่ยนชื่อเช่นกัน การเปลี่ยนแปลงทั้งสองนี้เกิดขึ้นพร้อมกัน ตอนนี้เราจะมีค่าที่ขัดแย้งกัน เรียกว่าเวอร์ชัน v1 และ v2


- **Vector Clock**
    1. **การตั้งค่า (Setup):** vector clock คือคู่ [server, version] ที่ผูกอยู่กับข้อมูลชิ้นหนึ่ง สามารถใช้เพื่อตรวจสอบ
        ว่าเวอร์ชันหนึ่งเกิดก่อน เกิดหลัง หรือขัดแย้งกับเวอร์ชันอื่นหรือไม่
        - สมมติว่า vector clock แสดงด้วย D([S1, v1], [S2, v2], …, [Sn, vn]) หากข้อมูล D ถูกเขียนไปยังเซิร์ฟเวอร์
        Si ระบบจะต้องทำอย่างใดอย่างหนึ่งต่อไปนี้
        - โดยที่: `D` คือข้อมูลชิ้นนั้น `Si` คือหมายเลขระบุเซิร์ฟเวอร์ `vi` คือตัวนับเวอร์ชันของข้อมูลบนเซิร์ฟเวอร์ `Si`

    2. **การอัปเดต Vector Clock:** เมื่อข้อมูลชิ้นหนึ่งถูกแก้ไขที่เซิร์ฟเวอร์ตัวหนึ่ง:
        - หากเซิร์ฟเวอร์นั้นมีอยู่แล้วใน vector clock ตัวนับเวอร์ชันของมันจะถูกเพิ่มขึ้น
        - หากไม่มี จะมีการเพิ่ม entry ใหม่เข้าไปใน vector clock

    3. **การตรวจจับความขัดแย้ง (Conflict Detection):**
        - **ไม่มีความขัดแย้ง:** เวอร์ชัน X ถือเป็นบรรพบุรุษ (ancestor) ของเวอร์ชัน Y หากตัวนับทุกตัวใน X มีค่าน้อยกว่าหรือเท่ากับตัวนับใน Y
        - **มีความขัดแย้ง:** สองเวอร์ชันจะถือเป็น sibling กัน หากมีตัวนับอย่างน้อยหนึ่งตัวใน Y ที่มีค่าน้อยกว่าตัวนับที่ตรงกันใน X

    4. **การแก้ไขความขัดแย้ง (Conflict Resolution):** เมื่อตรวจพบความขัดแย้ง (เวอร์ชันแบบ sibling) ระบบจะต้องพึ่งพา logic เฉพาะของแอปพลิเคชัน หรือให้ไคลเอนต์เข้ามาช่วยประสาน (reconcile) ข้อมูล

        <p align="center">
        <img src="./images/vector-clock.png"  alt="Server hashing" width="500">
        </p>

- **ความท้าทาย:**
  - เพิ่มความซับซ้อนให้กับฝั่งไคลเอนต์
  - ขนาดของ vector clock อาจเติบโตขึ้นเมื่อมีการอัปเดตจำนวนมาก จึงต้องใช้กลยุทธ์การตัด (trimming) เพื่อจำกัดขนาด


### 5. การจัดการความล้มเหลว (Handling Failures)

#### a. การตรวจจับความล้มเหลว (Failure Detection)
การเชื่อว่าเซิร์ฟเวอร์หนึ่งล่มเพียงเพราะเซิร์ฟเวอร์อีกตัวบอกว่ามันล่มนั้นไม่เพียงพอ โดยทั่วไปจำเป็นต้องมีแหล่งข้อมูลอิสระอย่างน้อยสองแหล่งเพื่อยืนยันว่าเซิร์ฟเวอร์นั้นล่มจริง
- **Gossip Protocol:**
    <div style="margin-left:3rem">
        <img src="./images/gossip-protocol.png"  alt="Gossip protocol" width="600">
    </div>

    - แต่ละ node เก็บ member ID และ heartbeat counter ของตัวเอง
    - แต่ละ node เพิ่มค่า heartbeat counter ของตัวเองเป็นระยะ
    - แต่ละ node ส่ง heartbeat ไปยังกลุ่ม node แบบสุ่มเป็นระยะ
    - หาก heartbeat ไม่เพิ่มขึ้นเป็นเวลานานกว่าช่วงเวลาที่กำหนดไว้ล่วงหน้า สมาชิกนั้นจะ
    ถูกถือว่าออฟไลน์



#### b. ความล้มเหลวชั่วคราว (Temporary Failures)
- **Sloppy Quorum:** ใช้ node ที่ยังทำงานได้ปกติ (healthy) เพื่อรักษาการทำงานของระบบไว้ชั่วคราว
        <p align="center">
        <img src="./images/sloppy-quorum.png"   alt="Sloppy Quorum" width="400">
        </p>

    - หลังจากตรวจพบความล้มเหลว ระบบจำเป็นต้องใช้กลไกบางอย่างเพื่อรับประกัน availability
    - แทนที่จะบังคับใช้ข้อกำหนดของ quorum ระบบจะเลือกเซิร์ฟเวอร์ที่ยังทำงานได้ปกติ W ตัวแรกสำหรับการเขียน และ R ตัวแรกสำหรับการอ่านบน hash ring
    - เซิร์ฟเวอร์ที่ออฟไลน์จะถูกข้ามไป หากเซิร์ฟเวอร์ตัวหนึ่งไม่พร้อมใช้งาน เซิร์ฟเวอร์อีกตัวจะประมวลผล request แทนไปก่อนชั่วคราว


- **Hinted Handoff:** เซิร์ฟเวอร์ที่ออฟไลน์จะตามให้ทันการเปลี่ยนแปลงเมื่อกลับมาออนไลน์
    - เมื่อเซิร์ฟเวอร์ที่ล่มกลับมาออนไลน์อีกครั้ง การเปลี่ยนแปลงต่าง ๆ จะถูกส่งกลับไปยังเซิร์ฟเวอร์นั้น เพื่อให้ข้อมูลสอดคล้องกัน

#### c. ความล้มเหลวถาวร (Permanent Failures)
- ใช้ **Merkle Tree** เพื่อทำ synchronization ระหว่าง replica อย่างมีประสิทธิภาพ
    **Merkle Tree** (หรือ hash tree) คือโครงสร้างข้อมูลที่ใช้ตรวจจับและแก้ไขความไม่สอดคล้องกันระหว่าง replica ได้อย่างมีประสิทธิภาพในระหว่างที่เกิดความล้มเหลวถาวร

- การทำงาน
    1. **โครงสร้าง (Structure):**
        - **Leaf Node** จัดเก็บ hash ของแต่ละ data block
        - **Non-Leaf Node** จัดเก็บ hash ของ child node ของมัน
        - **Root hash** แสดงถึงสถานะโดยรวมของข้อมูลทั้งหมดใน tree

    2. **การสร้าง Merkle Tree:**
        - **ขั้นที่ 1:** แบ่ง key space ออกเป็น bucket
            
            <img src="./images/key-bucket.png"   alt="Key Bucket" width="500">

        - **ขั้นที่ 2:** ทำ hash แต่ละ key ใน bucket โดยใช้ uniform hashing

            <img src="./images/hash-key-bucket.png"   alt="Hash Key Bucket" width="500">

        - **ขั้นที่ 3:** สร้าง hash เดียวสำหรับแต่ละ bucket
        
            <img src="./images/hash-bucket.png"   alt="Hash Bucket" width="500">

        - **ขั้นที่ 4:** รวม hash ของ bucket ต่าง ๆ เพื่อคำนวณ hash ในระดับที่สูงขึ้น ไปจนถึง root hash

            <img src="./images/merkel-tree.png"   alt="Merkel Tree" width="500">



    3. **การ Synchronization:**
        - ในการ synchronize replica สองตัว:
            - เปรียบเทียบ root hash ของทั้งสอง
            - หาก root hash ตรงกัน แสดงว่า replica สอดคล้องกัน
            - หาก root hash ต่างกัน ให้เปรียบเทียบ hash ของ child แบบวนซ้ำ (recursively) เพื่อระบุ bucket ที่ไม่สอดคล้องกัน
        - มีเพียงข้อมูลที่ไม่สอดคล้องกันเท่านั้นที่จะถูก synchronize

- ข้อดี
    - **ประสิทธิภาพ (Efficiency):** มีเพียงข้อมูลที่ไม่สอดคล้องกันเท่านั้นที่ถูก synchronize ช่วยลดการส่งข้อมูล
    - **ความสามารถในการขยายระบบ (Scalability):** มีประสิทธิภาพสำหรับชุดข้อมูลขนาดใหญ่ โดยมี overhead ในการ synchronization น้อย
    - **ความน่าเชื่อถือ (Reliability):** รับประกันความสอดคล้องของข้อมูลระหว่าง replica


### 6. การจัดการเมื่อ Data Center ล่ม (Handling Data Center Outages)
- ทำสำเนาข้อมูลไปยังหลาย data center เพื่อรับประกัน availability ในระหว่างที่เกิดเหตุขัดข้อง

---

## เส้นทางการเขียนและการอ่าน (Write and Read Paths)
### 1. เส้นทางการเขียน (Write Path) (อ้างอิงจากสถาปัตยกรรมของ Cassandra)

<div style="margin-left:3rem">
    <img src="./images/write-path.png"   alt="Hash Bucket" width="500">
</div>

- บันทึกการเขียนลงใน **commit log**
- บันทึกข้อมูลลงใน **memory cache**
- flush ข้อมูลลงสู่ **SSTable** (Sorted String Table) บนดิสก์ เมื่อ cache เต็ม

   

### 2. เส้นทางการอ่าน (Read Path)
<div style="margin-left:3rem">
    <img src="./images/read-path.png"   alt="Hash Bucket" width="500">
    <img src="./images/read-path-without-cache.png"   alt="Hash Bucket" width="500">
</div>

- ตรวจสอบ **memory cache** เพื่อหาข้อมูล
- หากไม่มี ให้ใช้ **Bloom Filter** เพื่อระบุตำแหน่งข้อมูลใน SSTable
- ดึงข้อมูลและส่งกลับ


---

## สถาปัตยกรรมสุดท้าย (Final Architecture)

<p align="center">
<img src="./images/final-architecture.png"   alt="Hash Bucket" width="500">
</p>


-  ไคลเอนต์สื่อสารกับ key-value store ผ่าน API ที่เรียบง่าย ได้แก่ get(key) และ put(key,
value)
- coordinator คือ node ที่ทำหน้าที่เป็น proxy ระหว่างไคลเอนต์กับ key-value store
- node ต่าง ๆ ถูกกระจายอยู่บน ring โดยใช้ consistent hashing
- ระบบทำงานแบบกระจายอำนาจอย่างสมบูรณ์ (completely decentralized) ทำให้การเพิ่มและย้าย node สามารถทำได้โดยอัตโนมัติ
- ข้อมูลถูกทำสำเนาไว้ที่หลาย node
- ไม่มี single point of failure เนื่องจากทุก node มีชุดหน้าที่รับผิดชอบ (responsibility) ชุดเดียวกัน
