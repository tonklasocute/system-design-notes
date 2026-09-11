# บทที่ 24: ระบบจัดเก็บข้อมูลแบบ Object แบบ S3 (S3-like Object Storage)

## บทนำ

ในบทนี้ เราจะออกแบบบริการจัดเก็บข้อมูลแบบ **object (object storage)** ที่คล้ายกับ **Amazon S3**

ระบบจัดเก็บข้อมูล (storage system) สามารถแบ่งออกเป็น 3 หมวดหมู่หลัก ได้แก่:
- **Block storage**
- **File storage**
- **Object storage**

**Block storage** คืออุปกรณ์ที่เกิดขึ้นตั้งแต่ยุค 1960s เช่น HDD และ SSD
อุปกรณ์เหล่านี้มักจะเชื่อมต่อกับเซิร์ฟเวอร์โดยตรงทางกายภาพ แม้ว่าจะสามารถเชื่อมต่อผ่านเครือข่ายความเร็วสูง (network-attached) ได้เช่นกัน
เซิร์ฟเวอร์สามารถ format บล็อกดิบ (raw block) เหล่านั้นและใช้งานเป็นระบบไฟล์ (file system) หรือส่งมอบการควบคุมให้กับเซิร์ฟเวอร์โดยตรงก็ได้

**File storage** ถูกสร้างขึ้นบนพื้นฐานของ block storage โดยให้ระดับ abstraction ที่สูงกว่า ทำให้การจัดการโฟลเดอร์และไฟล์ทำได้ง่ายขึ้น

**Object storage** ยอมสละประสิทธิภาพ (performance) เพื่อแลกกับความทนทาน (durability) สูง ความสามารถในการขยาย (scale) ที่มหาศาล และต้นทุนที่ต่ำ
มันมุ่งเป้าไปที่ข้อมูล "cold" และถูกใช้งานหลักในการเก็บถาวร (archival) และสำรองข้อมูล (backup)
ไม่มีโครงสร้างไดเรกทอรีแบบลำดับชั้น (hierarchical directory structure) ข้อมูลทั้งหมดถูกจัดเก็บเป็น object ในโครงสร้างแบบแบน (flat structure)
มันมีความเร็วค่อนข้างช้าเมื่อเทียบกับ storage ประเภทอื่น ผู้ให้บริการ cloud ส่วนใหญ่มีบริการ object storage เช่น Amazon S3, Google GCS เป็นต้น

<div style="margin-left:3rem">
    <img src="./images/storage-comparison.png" alt="storage-comparison" width="500" />
</div>

|                 | Block Storage                    | File Storage                            | Object Storage                 |
|-----------------|----------------------------------|-----------------------------------------|--------------------------------|
| เนื้อหาที่แก้ไขได้ (Mutable Content) | Y                                | Y                                       | N (มี object versioning)     |
| ต้นทุน (Cost)            | สูง                             | ปานกลางถึงสูง                          | ต่ำ                            |
| ประสิทธิภาพ (Performance)     | ปานกลางถึงสูง, สูงมาก        | ปานกลางถึงสูง                          | ต่ำถึงปานกลาง                  |
| ความสอดคล้องของข้อมูล (Consistency)     | Strong consistency               | Strong consistency                      | Strong consistency [5]         |
| การเข้าถึงข้อมูล (Data access)     | SAS/iSCSI/FC                     | Standard file access, CIFS/SMB, และ NFS | RESTful API                    |
| ความสามารถในการขยาย (Scalability)     | ปานกลาง               | สูง                        | มหาศาล               |
| เหมาะสำหรับ (Good for)        | Virtual machines (VM), ฐานข้อมูล | การเข้าถึงระบบไฟล์แบบทั่วไป (general-purpose) | ข้อมูลไบนารี, ข้อมูลไม่มีโครงสร้าง (unstructured data) |

คำศัพท์บางส่วนที่เกี่ยวข้องกับ object storage:
- **Bucket** - container เชิงตรรกะ (logical) สำหรับเก็บ object ชื่อของ bucket จะไม่ซ้ำกันทั่วทั้งระบบ (globally unique)
- **Object** - ข้อมูลชิ้นหนึ่งที่ถูกจัดเก็บอยู่ใน bucket ประกอบด้วยข้อมูล object และ metadata
- **Versioning** - คุณสมบัติที่เก็บรักษาหลายเวอร์ชันของ object เดียวกันไว้ใน bucket เดียวกัน
- **Uniform Resource Identifier (URI)** - ทรัพยากรแต่ละชิ้นจะถูกระบุอย่างไม่ซ้ำกันด้วย URI
- **Service-level Agreement (SLA)** - ข้อตกลงระหว่างผู้ให้บริการและลูกค้า

SLA ของ Amazon S3 Standard-Infrequent Access storage class:
- ความทนทาน (durability) 99.999999999% ครอบคลุมหลาย Availability Zone
- ข้อมูลยังคงอยู่ (resilient) แม้ทั้ง Availability Zone ถูกทำลาย
- ออกแบบมาให้มีความพร้อมใช้งาน (availability) 99.9%

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)

- C: ควรมีฟีเจอร์ใดบ้าง?
- I: การสร้าง bucket, การอัปโหลด/ดาวน์โหลด object, versioning, การแสดงรายการ object ใน bucket
- C: ขนาดข้อมูลโดยทั่วไปเป็นเท่าไร?
- I: เราต้องรองรับทั้ง object ขนาดใหญ่มากและ object ขนาดเล็กอย่างมีประสิทธิภาพ
- C: เราจัดเก็บข้อมูลกี่หน่วยต่อปี?
- I: 100 petabyte
- C: เราสามารถสมมติว่าความทนทานของข้อมูล (data durability) อยู่ที่ 6 nines (99.9999%) และความพร้อมใช้งานของบริการ (service availability) อยู่ที่ 4 nines (99.99%) ได้หรือไม่?
- I: ได้ ฟังดูสมเหตุสมผล

### **ความต้องการที่ไม่ใช่ฟังก์ชันการทำงาน (Non-functional requirements)**

- **ข้อมูล 100 PB**
- **ความทนทานของข้อมูล 6 nines**
- **ความพร้อมใช้งานของบริการ 4 nines**
- ประสิทธิภาพในการจัดเก็บ (storage efficiency) ลดต้นทุนการจัดเก็บ ในขณะที่ยังคงรักษาความน่าเชื่อถือ (reliability) และประสิทธิภาพในระดับสูง

### **การประมาณการแบบคร่าว ๆ (Back-of-the-envelope estimation)**

Object storage มีแนวโน้มที่จะเจอ bottleneck ที่ความจุดิสก์ (disk capacity) หรือจำนวนการเข้าถึงต่อวินาที (IOPS)

สมมติฐาน:
- เรามี object ขนาดเล็ก (น้อยกว่า 1MB) 20%, ขนาดกลาง (1-64MB) 60% และขนาดใหญ่ (มากกว่า 64MB) 20%
- ฮาร์ดดิสก์หนึ่งลูก (SATA, 7200rpm) สามารถทำ random seek ได้ 100-150 ครั้งต่อวินาที (100-150 IOPS)

จากสมมติฐานข้างต้น เราสามารถประมาณจำนวน object ทั้งหมดที่ระบบสามารถจัดเก็บได้
- ใช้ขนาดมัธยฐาน (median size) ของแต่ละประเภท object เพื่อให้การคำนวณง่ายขึ้น - 0.5MB สำหรับขนาดเล็ก, 32MB สำหรับขนาดกลาง, 200MB สำหรับขนาดใหญ่
- เมื่อมีพื้นที่จัดเก็บ 100PB (10^11 MB) และใช้พื้นที่จัดเก็บ 40% จะได้ประมาณ 0.68 พันล้าน object
- หากสมมติว่า metadata มีขนาด 1KB เราต้องการพื้นที่ 0.68TB สำหรับจัดเก็บข้อมูล metadata

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

มาสำรวจคุณสมบัติที่น่าสนใจของ object storage ก่อนที่จะเจาะลึกลงไปในการออกแบบ:
- **ความไม่เปลี่ยนแปลงของ object (Object immutability)** - object ใน object storage นั้นไม่เปลี่ยนแปลง (immutable) ซึ่งแตกต่างจากระบบจัดเก็บข้อมูลอื่น ๆ เราสามารถลบหรือแทนที่ object ได้ แต่ไม่สามารถ update ได้
- **Key-value store** - URI ของ object คือ key ของมัน และเราสามารถดึงเนื้อหาของมันได้โดยการเรียก HTTP call
- **เขียนครั้งเดียว อ่านหลายครั้ง (Write once, read many times)** - รูปแบบการเข้าถึงข้อมูลคือเขียนครั้งเดียวแล้วอ่านซ้ำหลายครั้ง จากงานวิจัยของ LinkedIn พบว่า 95% ของ operation เป็นการอ่าน
- รองรับทั้ง object ขนาดเล็กและขนาดใหญ่

ปรัชญาการออกแบบของ object storage นั้นคล้ายกับ UNIX - เมื่อเราบันทึกไฟล์ มันจะสร้างชื่อไฟล์ในโครงสร้างข้อมูลที่เรียกว่า inode และข้อมูลไฟล์จะถูกจัดเก็บในตำแหน่งดิสก์ที่แตกต่างกัน
inode จะมีรายการของ file block pointer ซึ่งชี้ไปยังตำแหน่งต่าง ๆ บนดิสก์

เมื่อเข้าถึงไฟล์ เราจะดึง metadata จาก inode ก่อน แล้วจึงดึงเนื้อหาไฟล์ทีหลัง

Object storage ทำงานในลักษณะคล้ายกัน - metadata store ถูกใช้สำหรับข้อมูลของไฟล์ แต่เนื้อหาจริงถูกจัดเก็บอยู่บนดิสก์:

<div style="margin-left:3rem">
    <img src="./images/object-store-vs-unix.png" alt="object-store-vs-unix" width="500" />
</div>

การแยก metadata ออกจากเนื้อหาไฟล์ ทำให้เราสามารถขยาย (scale) แต่ละ store ได้อย่างอิสระต่อกัน:

<div style="margin-left:3rem">
    <img src="./images/bucket-and-object.png" alt="bucket-and-object" width="500" />
</div>

### **การออกแบบระดับสูง (High-level design)**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

- **Load balancer** - กระจาย API request ไปยัง replica ของบริการ
- **API service** - เซิร์ฟเวอร์แบบ stateless ทำหน้าที่ orchestrate การเรียกไปยัง metadata store, object store รวมถึง IAM service
- **Identity and access management (IAM)** - จุดศูนย์กลางสำหรับ authentication, authorization และ access control
- **Data store** - จัดเก็บและดึงข้อมูลจริง โดย operation ต่าง ๆ อ้างอิงจาก object ID (UUID)
- **Metadata store** - จัดเก็บ metadata ของ object

### **การอัปโหลด object (Uploading an object)**

<div style="margin-left:3rem">
    <img src="./images/uploading-object.png" alt="uploading-object" width="500" />
</div>

- สร้าง bucket ชื่อ "bucket-to-share" ผ่าน HTTP PUT request
- API service เรียก IAM เพื่อตรวจสอบว่าผู้ใช้ได้รับอนุญาตและมีสิทธิ์เขียน (write permission)
- API service เรียก metadata store เพื่อสร้าง entry ของ bucket เมื่อสร้างเสร็จ จะส่ง success response กลับมา
- หลังจากสร้าง bucket แล้ว จะมีการส่ง HTTP PUT เพื่อสร้าง object ชื่อ "script.txt"
- API service ตรวจสอบตัวตนของผู้ใช้และยืนยันว่าผู้ใช้มีสิทธิ์เขียน
- เมื่อผ่านการตรวจสอบแล้ว payload ของ object จะถูกส่งผ่าน HTTP PUT ไปยัง data store โดย data store จะจัดเก็บข้อมูลและส่ง UUID กลับมา
- API service เรียก metadata store เพื่อสร้าง entry ใหม่ที่มี object_id, bucket_id และ bucket_name รวมถึง metadata อื่น ๆ

ตัวอย่าง request สำหรับการอัปโหลด object:

```
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
```

### **การดาวน์โหลด object (Downloading an object)**

Bucket ไม่มีโครงสร้างไดเรกทอรี (directory hierarchy) แต่เราสามารถสร้างโครงสร้างเชิงตรรกะ (logical hierarchy) ได้โดยการต่อชื่อ bucket และชื่อ object เข้าด้วยกันเพื่อจำลองโครงสร้างแบบโฟลเดอร์

ตัวอย่าง GET request สำหรับดึง object:

```
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
```

<div style="margin-left:3rem">
    <img src="./images/download-object.png" alt="download-object" width="500" />
</div>

- ไคลเอนต์ส่ง HTTP GET request ไปยัง load balancer เช่น `GET /bucket-to-share/script.txt`
- API service ตรวจสอบกับ IAM เพื่อยืนยันว่าผู้ใช้มีสิทธิ์ที่ถูกต้องในการอ่าน bucket
- เมื่อตรวจสอบผ่านแล้ว UUID ของ object จะถูกดึงมาจาก metadata store
- payload ของ object จะถูกดึงมาจาก data store โดยอิงตาม UUID และส่งกลับไปยังไคลเอนต์

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### **Data store**

นี่คือวิธีที่ API service ปฏิสัมพันธ์กับ data store:

<div style="margin-left:3rem">
    <img src="./images/data-store-interactions.png" alt="data-store-interactions" width="500" />
</div>

องค์ประกอบหลักของ data store:

<div style="margin-left:3rem">
    <img src="./images/data-store-main-components.png" alt="data-store-main-components" width="500" />
</div>

data routing service ให้บริการ RESTful หรือ gRPC API เพื่อเข้าถึง data node cluster
มันเป็นบริการแบบ stateless ซึ่งสามารถขยาย (scale) ได้โดยการเพิ่มเซิร์ฟเวอร์

หน้าที่หลักของมันคือ:
- ร้องขอไปยัง placement service เพื่อหา data node ที่ดีที่สุดสำหรับจัดเก็บข้อมูล
- อ่านข้อมูลจาก data node และส่งกลับไปยัง API service
- เขียนข้อมูลไปยัง data node

placement service เป็นผู้กำหนดว่า data node ใดควรจัดเก็บ object ใด
มันดูแลรักษา virtual cluster map ซึ่งกำหนดโครงสร้างทางกายภาพ (physical topology) ของ cluster

<div style="margin-left:3rem">
    <img src="./images/virtual-cluster-map.png" alt="virtual-cluster-map" width="500" />
</div>

บริการนี้ยังส่ง heartbeat ไปยัง data node ทั้งหมด เพื่อพิจารณาว่าควรถูกลบออกจาก virtual cluster หรือไม่

เนื่องจากบริการนี้มีความสำคัญอย่างมาก จึงแนะนำให้มี cluster จำนวน 5 หรือ 7 replica ซึ่งซิงค์กันด้วย consensus algorithm อย่าง Paxos หรือ Raft
ตัวอย่างเช่น cluster ที่มี 7 node จะสามารถทนต่อการล้มเหลวของ node ได้ถึง 3 node

data node จัดเก็บข้อมูล object จริง
ความน่าเชื่อถือ (reliability) และความทนทาน (durability) ได้รับการรับประกันด้วยการทำสำเนาข้อมูล (replicate) ไปยัง data node หลายตัว

แต่ละ data node มี daemon ทำงานอยู่ ซึ่งจะส่ง heartbeat ไปยัง placement service

heartbeat จะประกอบด้วย:
- data node จัดการดิสก์ไดรฟ์ (HDD หรือ SSD) กี่ตัว?
- แต่ละไดรฟ์มีข้อมูลจัดเก็บอยู่เท่าไร?

#### กระบวนการเก็บข้อมูลอย่างถาวร (Data persistence flow)

<div style="margin-left:3rem">
    <img src="./images/data-persistence-flow.png" alt="data-persistence-flow" width="500" />
</div>

- API service ส่งต่อข้อมูล object ไปยัง data store
- data routing service ส่งข้อมูลไปยัง primary data node
- primary data node บันทึกข้อมูลไว้ในเครื่อง (locally) และทำสำเนาข้อมูล (replicate) ไปยัง secondary data node อีก 2 ตัว โดยจะส่ง response กลับหลังจากการทำสำเนาสำเร็จ
- UUID ของ object จะถูกส่งกลับไปยัง API service

ข้อควรระวัง:
- กลุ่มการทำสำเนา (replication group) ของ object แต่ละตัว จะถูกกำหนดอย่างแน่นอน (deterministic) โดยใช้ consistent hashing ตาม UUID ของ object
- ในขั้นตอนที่ 4 primary data node จะทำสำเนาข้อมูล object ก่อนที่จะส่ง response กลับ วิธีนี้เลือก strong consistency มากกว่าจะเน้น latency ที่ต่ำ

<div style="margin-left:3rem">
    <img src="./images/consistency-vs-latency.png" alt="consistency-vs-latency" width="500" />
</div>

#### วิธีจัดระเบียบข้อมูล (How data is organized)

วิธีที่ง่ายที่สุดในการจัดการข้อมูลคือการจัดเก็บแต่ละ object ไว้ในไฟล์แยกกัน

วิธีนี้ใช้ได้ แต่ไม่มีประสิทธิภาพเมื่อระบบไฟล์มีไฟล์ขนาดเล็กจำนวนมาก:
- data block บน HDD จะถูกใช้อย่างสิ้นเปลือง เนื่องจากทุกไฟล์ใช้พื้นที่เต็ม block size ซึ่งโดยทั่วไปมีขนาด 4KB
- มีไฟล์จำนวนมากหมายถึงมี inode จำนวนมาก ระบบปฏิบัติการไม่ค่อยจัดการกับ inode ที่มีจำนวนมากเกินไปได้ดี และยังมีข้อจำกัดสูงสุดของจำนวน inode ด้วย

ปัญหาเหล่านี้สามารถแก้ไขได้ด้วยการรวมไฟล์ขนาดเล็กจำนวนมากเข้าเป็นไฟล์ที่ใหญ่ขึ้นผ่าน write-ahead log (WAL) เมื่อไฟล์ถึงความจุที่กำหนด (โดยทั่วไปคือไม่กี่ GB) จะมีการสร้างไฟล์ใหม่:

<div style="margin-left:3rem">
    <img src="./images/wal-optimization.png" alt="wal-optimization" width="500" />
</div>

ข้อเสียของวิธีนี้คือการเข้าถึงเพื่อเขียน (write access) ไปยังไฟล์นั้นต้องทำแบบเรียงลำดับ (serialize) หลาย core ที่เข้าถึงไฟล์เดียวกันต้องรอซึ่งกันและกัน
เพื่อแก้ปัญหานี้ เราสามารถจำกัดให้ไฟล์แต่ละไฟล์ผูกกับ core เฉพาะ เพื่อหลีกเลี่ยง lock contention

#### การค้นหา object (Object lookup)

เพื่อรองรับการจัดเก็บหลาย object ไว้ในไฟล์เดียวกัน เราต้องดูแลรักษาตาราง (table) ที่บอก data node ว่า:
- `object_id`
- `filename` ที่ object ถูกจัดเก็บอยู่
- `file_offset` ตำแหน่งที่ object เริ่มต้น
- `object_size`

เราสามารถนำตารางนี้ไปใช้งานใน file-based db อย่าง RocksDB หรือฐานข้อมูลเชิงสัมพันธ์แบบดั้งเดิม (traditional relational database)
เนื่องจากรูปแบบการเข้าถึงคือเขียนน้อยแต่อ่านมาก (low write+high read) ฐานข้อมูลเชิงสัมพันธ์จึงเหมาะสมกว่า

เราควร deploy มันอย่างไร?
เราอาจ deploy ฐานข้อมูลและ scale แยกต่างหากใน cluster ซึ่งเข้าถึงได้โดย data node ทุกตัว

ข้อเสีย:
- เราจำเป็นต้อง scale cluster อย่างต่อเนื่องเพื่อรองรับ request ทั้งหมด
- มี network latency เพิ่มเติมระหว่าง data node กับ db cluster

อีกทางเลือกหนึ่งคือการใช้ประโยชน์จากข้อเท็จจริงที่ว่า data node สนใจเฉพาะข้อมูลที่เกี่ยวข้องกับตัวมันเองเท่านั้น
ดังนั้นเราสามารถ deploy ฐานข้อมูลเชิงสัมพันธ์ไว้ภายใน data node เองได้

SQLite เป็นตัวเลือกที่ดี เนื่องจากเป็นฐานข้อมูลเชิงสัมพันธ์แบบ file-based ที่มีน้ำหนักเบา (lightweight)

#### กระบวนการเก็บข้อมูลอย่างถาวรที่ปรับปรุงแล้ว (Updated data persistence flow)

<div style="margin-left:3rem">
    <img src="./images/updated-data-persistence-flow.png" alt="updated-data-persistence-flow" width="500" />
</div>

- API Service ส่ง request เพื่อบันทึก object ใหม่
- data node service ต่อท้าย object ใหม่ที่ปลายไฟล์ชื่อ "/data/c"
- record ใหม่ของ object จะถูกแทรกเข้าไปในตาราง object mapping

#### ความทนทาน (Durability)

ความทนทานของข้อมูล (data durability) เป็นข้อกำหนดที่สำคัญในการออกแบบของเรา เพื่อให้ได้ความทนทาน 6 nines กรณีความล้มเหลวทุกแบบต้องได้รับการพิจารณาอย่างถี่ถ้วน

ปัญหาแรกที่ต้องแก้ไขคือความล้มเหลวของฮาร์ดแวร์ เราสามารถแก้ปัญหานี้ได้โดยการทำสำเนา data node เพื่อลดความน่าจะเป็นของความล้มเหลว
แต่นอกจากนั้น เรายังจำเป็นต้องทำสำเนาข้าม failure domain ที่แตกต่างกันด้วย (ข้าม rack, ข้าม data center, เครือข่ายแยกกัน เป็นต้น)
เหตุการณ์วิกฤต (critical event) หนึ่งครั้งอาจทำให้ฮาร์ดแวร์ล้มเหลวพร้อมกันหลายตัวภายใน domain เดียวกันได้:

<div style="margin-left:3rem">
    <img src="./images/failure-domain-isolation.png" alt="failure-domain-isolation" width="500" />
</div>

หากสมมติว่าอัตราความล้มเหลวรายปี (annual failure rate) ของ HDD ทั่วไปอยู่ที่ 0.81% การทำสำเนา 3 ชุดจะให้ความทนทาน 6 nines

การทำสำเนา data node แบบนี้ทำให้เราได้ความทนทานตามที่ต้องการ แต่เราก็สามารถใช้ erasure coding เพื่อลดต้นทุนการจัดเก็บได้เช่นกัน

Erasure coding ช่วยให้เราสามารถใช้ parity bit ซึ่งทำให้เราสามารถกู้คืน bit ที่สูญหายได้ในกรณีเกิดความล้มเหลว:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding.png" alt="erasure-coding" width="500" />
</div>

ลองจินตนาการว่า bit เหล่านั้นคือ data node หากมี 2 ตัวล้มเหลว เราสามารถกู้คืนได้โดยใช้อีก 4 ตัวที่เหลือ

มี erasure coding scheme ที่แตกต่างกันหลายแบบ ในกรณีของเรา เราอาจใช้ 8+4 erasure coding โดยกระจายไปยัง failure domain ที่แตกต่างกันเพื่อเพิ่มความน่าเชื่อถือให้สูงสุด:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-across-failure-domains.png" alt="erasure-coding-across-failure-domains" width="500" />
</div>

Erasure coding ช่วยให้เราลดต้นทุนการจัดเก็บได้มาก (ปรับปรุงได้ถึง 50%) แต่แลกมาด้วยความเร็วในการเข้าถึงข้อมูลที่ลดลง เนื่องจาก data routing service ต้องรวบรวมข้อมูลจากหลายตำแหน่ง:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-vs-replication.png" alt="erasure-coding-vs-replication" width="500" />
</div>

ข้อควรระวังอื่น ๆ:
- การทำ replication ต้องใช้ storage overhead 200% (ในกรณี 3 replica) ในขณะที่ erasure coding ใช้เพียง 50%
- Erasure coding [ให้ความทนทานถึง 11 nines](https://github.com/Backblaze/erasure-coding-durability) เทียบกับ 6 nines ของ replication
- Erasure coding ต้องใช้การคำนวณมากขึ้นในการคำนวณและจัดเก็บ parity

โดยสรุปแล้ว replication มีประโยชน์มากกว่าสำหรับแอปพลิเคชันที่ไวต่อ latency ในขณะที่ erasure coding น่าสนใจสำหรับความคุ้มค่าด้านต้นทุนการจัดเก็บและความทนทาน
Erasure coding ยังยากกว่ามากในการนำไปใช้งานจริง (implement)

#### การตรวจสอบความถูกต้อง (Correctness verification)

หากดิสก์ล้มเหลวทั้งหมด การตรวจจับความล้มเหลวนั้นทำได้ง่าย แต่หากมีเพียงบางส่วนของหน่วยความจำในดิสก์ที่เสียหาย (corrupted) การตรวจจับจะไม่ตรงไปตรงมาเช่นนั้น

ในการตรวจจับปัญหานี้ เราสามารถใช้ checksum - ค่า hash ของเนื้อหาไฟล์ ซึ่งสามารถใช้ตรวจสอบความสมบูรณ์ (integrity) ของไฟล์ได้

ในกรณีของเรา เราจะจัดเก็บ checksum สำหรับแต่ละไฟล์และแต่ละ object:

<div style="margin-left:3rem">
    <img src="./images/checksums-for-correctness.png" alt="checksums-for-correctness" width="500" />
</div>

ในกรณีของ erasure coding (8+4) เราจำเป็นต้องดึงข้อมูลแต่ละส่วนใน 8 ส่วนแยกกัน และตรวจสอบ checksum ของแต่ละส่วน

### **โมเดลข้อมูล metadata (Metadata data model)**

โครงสร้างตาราง (Table schemas):

<div style="margin-left:3rem">
    <img src="./images/metadata-data-model.png" alt="metadata-data-model" width="500" />
</div>

Query ที่เราต้องรองรับ:
- ค้นหา object ID จากชื่อ
- แทรก/ลบ object ตามชื่อ
- แสดงรายการ object ใน bucket ที่มี prefix เดียวกัน

โดยทั่วไปจะมีการจำกัดจำนวน bucket ที่ผู้ใช้แต่ละคนสามารถสร้างได้ ดังนั้นขนาดของตาราง bucket จึงมีขนาดเล็กและสามารถใส่ในเซิร์ฟเวอร์ฐานข้อมูลตัวเดียวได้
แต่เรายังคงต้อง scale เซิร์ฟเวอร์เพื่อรองรับ read throughput

อย่างไรก็ตาม ตาราง object นั้นอาจมีขนาดใหญ่เกินกว่าจะใส่ในเซิร์ฟเวอร์ฐานข้อมูลตัวเดียวได้ ดังนั้นเราสามารถ scale ตารางนี้ผ่านการทำ sharding:
- การทำ sharding ตาม bucket_id จะทำให้เกิดปัญหา hotspot เนื่องจาก bucket หนึ่งอาจมี object นับพันล้านชิ้น
- การทำ sharding ตาม object_id ทำให้ภาระงาน (load) กระจายอย่างสม่ำเสมอมากขึ้น แต่ query ของเราจะช้าลง
- เราเลือกทำ sharding ด้วย `hash(bucket_name, object_name)` เนื่องจาก query ส่วนใหญ่อ้างอิงจากชื่อ object/bucket

แม้จะใช้ scheme การ sharding แบบนี้ การแสดงรายการ object ใน bucket ก็ยังคงช้าอยู่ดี

### **การแสดงรายการ object ใน bucket (Listing objects in a bucket)**

ในฐานข้อมูลตัวเดียว การแสดงรายการ object ตาม prefix ของมัน (ดูเหมือนไดเรกทอรี) ทำงานได้ดังนี้:

```
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
```

สิ่งนี้ทำได้ยากขึ้นเมื่อฐานข้อมูลถูก shard เพื่อให้บรรลุผล เราสามารถรัน query นี้บนทุก shard แล้วรวมผลลัพธ์ในหน่วยความจำ (in-memory)
วิธีนี้ทำให้ pagination ทำได้ยาก เนื่องจากแต่ละ shard มีขนาดผลลัพธ์ต่างกัน และเราต้องรักษา limit/offset แยกกันสำหรับแต่ละ shard

เราสามารถใช้ประโยชน์จากข้อเท็จจริงที่ว่าโดยทั่วไป object store ไม่ได้ถูก optimize สำหรับการแสดงรายการ object ดังนั้นเราจึงยอมสละประสิทธิภาพในการแสดงรายการได้
เรายังสามารถสร้างตารางแบบ denormalize สำหรับการแสดงรายการ object ที่ shard ตาม bucket ID
วิธีนี้จะทำให้ query สำหรับการแสดงรายการทำงานได้เร็วเพียงพอ เนื่องจากถูกจำกัดอยู่ในฐานข้อมูลเพียงตัวเดียว

### **การทำ versioning ของ object (Object versioning)**

Versioning ทำงานได้ด้วยการมีคอลัมน์เพิ่มเติมชื่อ `object_version` ซึ่งเป็นชนิด TIMEUUID ทำให้เราสามารถเรียงลำดับ record ตามคอลัมน์นี้ได้

แต่ละเวอร์ชันใหม่จะสร้าง `object_id` ใหม่:

<div style="margin-left:3rem">
    <img src="./images/object-versioning.png" alt="object-versioning" width="500" />
</div>

การลบ object จะสร้างเวอร์ชันใหม่ที่มี `object_id` พิเศษ ซึ่งบ่งบอกว่า object นั้นถูกลบไปแล้ว query ที่มาถามหา object นี้จะได้รับ 404:

<div style="margin-left:3rem">
    <img src="./images/deleting-versioned-object.png" alt="deleting-versioned-object" width="500" />
</div>

### **การปรับปรุงประสิทธิภาพการอัปโหลดไฟล์ขนาดใหญ่ (Optimizing uploads of large files)**

การอัปโหลดไฟล์ขนาดใหญ่สามารถปรับปรุงให้มีประสิทธิภาพขึ้นได้ด้วยการทำ multipart upload - การแบ่งไฟล์ขนาดใหญ่ออกเป็นหลาย chunk แล้วอัปโหลดแยกกันอย่างอิสระ:

<div style="margin-left:3rem">
    <img src="./images/multipart-upload.png" alt="multipart-upload" width="500" />
</div>

- ไคลเอนต์เรียกบริการเพื่อเริ่มการทำ multipart upload
- data store ส่ง upload ID ที่ระบุการอัปโหลดนี้อย่างไม่ซ้ำกันกลับมา
- ไคลเอนต์แบ่งไฟล์ขนาดใหญ่ออกเป็นหลาย chunk แล้วอัปโหลดแยกกันโดยใช้ upload id
- เมื่อ chunk หนึ่งถูกอัปโหลดแล้ว data store จะส่ง etag กลับมา ซึ่งเป็น md5 checksum ที่ระบุ chunk การอัปโหลดนั้น
- หลังจากทุกส่วนถูกอัปโหลดครบแล้ว ไคลเอนต์จะส่ง complete multipart upload request ซึ่งประกอบด้วย upload_id, หมายเลขส่วน (part number) และ etag ทั้งหมด
- data store ประกอบ object กลับจากส่วนต่าง ๆ (parts) กระบวนการนี้อาจใช้เวลาสองสามนาที หลังจากนั้นจะมี success response ส่งกลับไปยังไคลเอนต์

ส่วน (part) เก่าที่ไม่มีประโยชน์แล้วสามารถลบออกได้ในขั้นตอนนี้ เราสามารถนำ garbage collector มาใช้จัดการกับสิ่งนี้ได้

### **การเก็บกวาดขยะ (Garbage collection)**

Garbage collection คือกระบวนการเรียกคืนพื้นที่จัดเก็บที่ไม่ถูกใช้งานแล้ว มีหลายวิธีที่ทำให้ข้อมูลกลายเป็นขยะ:
- **การลบ object แบบ lazy (lazy object deletion)** - object ถูก mark ว่าลบแล้ว โดยไม่ได้ถูกลบออกจริง
- **ข้อมูลกำพร้า (orphan data)** - เช่น การอัปโหลดล้มเหลวระหว่างทางและส่วน (part) เก่าต้องถูกลบ
- **ข้อมูลเสียหาย (corrupted data)** - ข้อมูลที่ตรวจสอบ checksum แล้วไม่ผ่าน

garbage collector ยังทำหน้าที่เรียกคืนพื้นที่ที่ไม่ถูกใช้งานใน replica ด้วย
เมื่อใช้ replication ข้อมูลจะถูกลบจากทั้ง primary และ replica เมื่อใช้ erasure coding (8+4) ข้อมูลจะถูกลบจากทั้ง 12 node

เพื่ออำนวยความสะดวกในการลบ เราจะใช้กระบวนการที่เรียกว่า compaction:
- garbage collector คัดลอก object ที่ยังไม่ถูกลบจาก "data/b" ไปยัง "data/d"
- ตาราง `object_mapping` จะถูกอัปเดตเมื่อการคัดลอกเสร็จสิ้น โดยใช้ database transaction
- เพื่อหลีกเลี่ยงการสร้างไฟล์ขนาดเล็กมากเกินไป การทำ compaction จะทำเมื่อไฟล์มีขนาดเกินเกณฑ์ที่กำหนด

<div style="margin-left:3rem">
    <img src="./images/compaction.png" alt="compaction" width="500" />
</div>

---

## ขั้นตอนที่ 4: สรุป (Wrap Up)

สิ่งที่เราได้พูดถึง:
- การออกแบบระบบจัดเก็บข้อมูลแบบ object คล้าย S3
- เปรียบเทียบความแตกต่างระหว่าง object, block และ file storage
- ครอบคลุมการอัปโหลด, ดาวน์โหลด, การแสดงรายการ, การทำ versioning ของ object ใน bucket
- เจาะลึกในการออกแบบ - data store และ metadata store, การทำ replication และ erasure coding, multipart upload, sharding
