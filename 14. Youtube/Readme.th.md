# บทที่ 14: การออกแบบ YouTube (Design YouTube)

## บทนำ
YouTube เป็นแพลตฟอร์ม video streaming ขนาดใหญ่ที่รองรับการอัปโหลดวิดีโอ การเล่นวิดีโอ และการโต้ตอบต่าง ๆ บทนี้จะเน้นการออกแบบระบบ video streaming ที่สามารถขยายตัว (scalable) ได้ โดยมีฟีเจอร์หลักดังนี้:
- **การอัปโหลดวิดีโอที่รวดเร็ว (Fast video uploads)**
- **การสตรีมวิดีโอที่ราบรื่น (Smooth video streaming)**
- **ความสามารถในการเปลี่ยนคุณภาพวิดีโอ (Ability to change video quality)**
- **ค่าใช้จ่ายด้าน infrastructure ที่ต่ำ (Low infrastructure cost)**
- **ความพร้อมใช้งานและความน่าเชื่อถือสูง (High availability and reliability)**

### สถิติที่สำคัญ (Key Statistics) (2020)
- **ผู้ใช้งานที่ active ต่อเดือน (monthly active users) 2 พันล้านคน**
- **มีการรับชมวิดีโอ 5 พันล้านครั้งต่อวัน**
- **37% ของทราฟฟิกอินเทอร์เน็ตบนมือถือมาจาก YouTube**
- รองรับ **80 ภาษา**
- **รายได้จากโฆษณา 15.1 พันล้านดอลลาร์** ในปี 2019

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและขอบเขต (Understand the Problem and Scope)

### ฟังก์ชันการทำงานหลัก (Core Functionalities)
1. อัปโหลดวิดีโอ
2. รับชมวิดีโอ

### แพลตฟอร์มที่รองรับ (Supported Platforms)
- แอปพลิเคชันมือถือ, เว็บเบราว์เซอร์ และสมาร์ททีวี

### สมมติฐาน (Assumptions)
- **ผู้ใช้งานที่ active ต่อวัน (Daily Active Users - DAU):** 5 ล้านคน
- **ขนาดวิดีโอโดยเฉลี่ย (Average Video Size):** 300 MB
- **ข้อจำกัดการอัปโหลด (Upload Limits):** สูงสุด 1 GB ต่อวิดีโอ
- **ความต้องการพื้นที่จัดเก็บต่อวัน (Daily Storage Need):** 150 TB
- **ค่าใช้จ่าย CDN (CDN Costs):** 5 ล้าน * 5 วิดีโอ * 0.3GB * $0.02 = $150,000/วัน (ใช้ Amazon CloudFront)

---

## ขั้นตอนที่ 2: การออกแบบระดับสูง (High-Level Design)

### คอมโพเนนต์ (Components)

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="High Level Design" width="400">
</div>

1. **Client:** อุปกรณ์ต่าง ๆ เช่น สมาร์ทโฟน คอมพิวเตอร์ และทีวี
2. **CDN (Content Delivery Network):** จัดเก็บและสตรีมวิดีโอ
3. **API Servers:** จัดการการโต้ตอบทั้งหมดของผู้ใช้ ยกเว้นการสตรีมวิดีโอ (เช่น การอัปโหลด, การอัปเดต metadata)
4. **Metadata Database:** จัดเก็บ metadata ของวิดีโอ (เช่น ชื่อเรื่อง, คำอธิบาย, ขนาด)
5. **Original Storage:** blob storage สำหรับวิดีโอที่อัปโหลดเข้ามา
6. **Transcoding Servers:** แปลงวิดีโอให้เป็นหลายความละเอียดและหลายรูปแบบ (format)
7. **Transcoded Storage:** blob storage สำหรับวิดีโอที่ผ่านการ transcode แล้ว


---

### ลำดับการทำงานหลัก (Core Workflows)
#### 1. ขั้นตอนการอัปโหลดวิดีโอ (Video Uploading Flow)
- **กระบวนการที่ทำงานแบบขนาน (Parallel Processes):**
  1. อัปโหลดวิดีโอไปยัง original storage
  2. อัปเดต metadata ของวิดีโอในฐานข้อมูล

- **การอัปโหลดวิดีโอ (ขั้นตอน) (Video Upload (Steps)):**

    <div style="margin-left:3rem">
        <img src="./images/video-uploading-flow.png" alt="Video Upload Flow" width="500">
    </div>

    - [1] วิดีโอจะถูกอัปโหลดไปยัง blob storage
    - [2] Transcoding servers แปลงวิดีโอให้เป็นหลายรูปแบบ
    - [3] เมื่อการ transcode เสร็จสิ้น ขั้นตอนสองขั้นตอนต่อไปนี้จะถูกดำเนินการแบบขนาน
        - [3a] วิดีโอที่ transcode แล้วจะถูกส่งไปยัง transcoded storage
        - [3b] เหตุการณ์การ transcode เสร็จสิ้นจะถูกเข้าคิว (queued) ใน completion queue
    - [3a.1] วิดีโอจะถูกกระจายไปยัง CDN
    - [3b.1] completion handlers จะอัปเดต metadata และแจ้งผู้ใช้



- **การอัปโหลด Metadata (ขั้นตอน) (Metadata Upload (Steps)):**

    <div style="margin-left:3rem">
        <img src="./images/metadata-upload.png" alt="Metadata Upload" height="500">
    </div>

    - Client จะส่ง request แบบขนานเพื่ออัปเดต metadata ของวิดีโอ
    - Request นี้มีข้อมูล metadata ของวิดีโอ ได้แก่ ชื่อไฟล์ ขนาด รูปแบบ ฯลฯ




#### 2. ขั้นตอนการสตรีมวิดีโอ (Video Streaming Flow)

<div style="margin-left: 3em;">
  <img src="./images/video-streaming-flow.png" alt="Video Streaming Flow" height="400">
</div>

- วิดีโอจะถูกสตรีมโดยตรงจาก CDN โดยใช้ edge servers เพื่อลด latency ให้เหลือน้อยที่สุด
- โปรโตคอลการสตรีมที่ได้รับความนิยม ได้แก่ MPEG_DASH, Apple HLS, Adobe HDS
- *โปรโตคอลการสตรีมที่ต่างกันจะรองรับ video encoding และ playback player ที่ต่างกัน*


---

## ขั้นตอนที่ 3: การออกแบบเชิงลึก (Design Deep Dive)

### การ Transcode วิดีโอ (Video Transcoding)
#### ความสำคัญ (Importance)
1. วิดีโอดิบ (raw video) ใช้พื้นที่จัดเก็บข้อมูลจำนวนมาก การ transcode ช่วยลดพื้นที่จัดเก็บ
2. รับประกันความเข้ากันได้ (compatibility) ระหว่างอุปกรณ์และเบราว์เซอร์ต่าง ๆ
3. ปรับคุณภาพวิดีโอให้เข้ากับสภาพเครือข่าย

#### คอมโพเนนต์ (Components)
- **Container:** ห่อหุ้ม (encapsulate) วิดีโอ, เสียง, และ metadata (เช่น MP4, AVI)
- **Codecs:** อัลกอริทึมสำหรับบีบอัดและคลายการบีบอัด (compression and decompression) (เช่น H.264, VP9)

#### แบบจำลอง Directed Acyclic Graph (DAG Model)
<div style="margin-left: 3em;">
    <img src="./images/dag-video-transcoding.png" alt="DAG Video Transcoding" width="600">
</div>

- การ transcode วิดีโอใช้ทรัพยากรในการประมวลผลสูงและใช้เวลานาน
- DAG Model นิยาม task ต่าง ๆ เช่น การเข้ารหัส (encoding), การสร้าง thumbnail, และการใส่ watermark
- ช่วยให้เกิด parallelism สูงในการประมวลผลวิดีโอ


- วิดีโอต้นฉบับจะถูกแยกออกเป็นวิดีโอ เสียง และ metadata
    - Video encodings: วิดีโอถูกแปลงให้รองรับความละเอียด, codec, และ bitrate ที่แตกต่างกัน
    - Thumbnail: สามารถอัปโหลดโดยผู้ใช้เองหรือให้ระบบสร้างขึ้นโดยอัตโนมัติ
    - Watermark: ภาพซ้อนทับ (overlay) บนวิดีโอที่มีข้อมูลระบุตัวตนของวิดีโอนั้น

---

### สถาปัตยกรรมการ Transcode วิดีโอ (Video Transcoding Architecture)

<div style="margin-left: 3em;">
<img src="./images/video-transcoding-architecture.png" alt="Video Transcoding" width="600">
</div>

1. **Preprocessor:** แยกวิดีโอออกเป็นชิ้นเล็ก ๆ (GOP alignment) มีหน้าที่รับผิดชอบ 4 อย่าง

    <div style="margin-left: 3em;">
        <img src="./images/dag-config.png" alt="DAG Config" width="500">
    </div>

    - Video splitting: video stream จะถูกแบ่งหรือแบ่งย่อยเพิ่มเติมตาม Group of Pictures (GOP) alignment
    - แบ่งวิดีโอตาม GOP alignment สำหรับไคลเอนต์รุ่นเก่า
    - สร้าง DAG จาก configuration file ที่โปรแกรมเมอร์ฝั่งไคลเอนต์เป็นผู้เขียน
    - จัดเก็บ GOP และ metadata ไว้ใน temporary storage เพื่อว่าหากการเข้ารหัสล้มเหลว ระบบสามารถใช้ข้อมูลที่บันทึกไว้เพื่อ retry ได้


2. **DAG Scheduler:** จัดระเบียบ task ให้อยู่ใน stage ที่ทำงานตามลำดับหรือแบบขนาน
    <div style="margin-left: 3em;">
        <img src="./images/dag-scheduler.png" alt="DAG Scheduler" width="500">
    </div>

    - แบ่งกราฟ DAG ออกเป็น stage ของ task ต่าง ๆ และนำเข้าไปไว้ใน task queue ภายใน resource manager
    - Stage 1: วิดีโอ, เสียง, และ metadata
    - ไฟล์วิดีโอจะถูกแบ่งเพิ่มเติมออกเป็นสอง task ใน stage 2: video encoding และ thumbnail


3. **Resource Manager:** รับผิดชอบการจัดการประสิทธิภาพของการจัดสรรทรัพยากร ประกอบด้วย 3 queue และ task scheduler
    <div style="margin-left: 3em;">
        <img src="./images/resource-manager.png" alt="Resource Manager" width="700">
    </div>

    - Task queue: priority queue ที่เก็บ task ที่รอการดำเนินการ
    - Worker queue: priority queue ที่เก็บข้อมูลการใช้งาน (utilization) ของ worker
    - Running queue: เก็บ task ที่กำลังทำงานอยู่ในปัจจุบัน และ worker ที่กำลังรัน task เหล่านั้น
    - Task scheduler: เลือก task/worker ที่เหมาะสมที่สุด และสั่งให้ task worker ที่เลือกไว้ดำเนินการ job นั้น


4. **Task Workers:** ทำหน้าที่ transcode และดำเนินการอื่น ๆ
    <div style="margin-left: 3em;">
        <img src="./images/task-worker.png" alt="Task Worker" width="250">
   </div>

    - Task worker แต่ละตัวอาจรัน task ที่แตกต่างกันไป


5. **Temporary Storage:** จัดเก็บข้อมูลระหว่างขั้นตอน (intermediate data) สำหรับการ retry
    - การเลือกระบบจัดเก็บข้อมูลขึ้นอยู่กับปัจจัยต่าง ๆ เช่น ประเภทข้อมูล, ขนาดข้อมูล, ความถี่ในการเข้าถึง, อายุของข้อมูล ฯลฯ
6. **Output:** วิดีโอที่ transcode แล้วและพร้อมสำหรับการกระจาย (distribution)


---

## การปรับปรุงประสิทธิภาพของระบบ (System Optimizations)

### การปรับปรุงด้านความเร็ว (Speed Optimizations)
1. **การอัปโหลดวิดีโอแบบขนาน (Parallel Video Uploads):** แบ่งวิดีโอออกเป็นชิ้นเล็ก ๆ เพื่อให้อัปโหลดได้เร็วขึ้นและสามารถ resume ได้

    <img src="./images/video-split.png" alt="Video Split" width="600">

2. **ศูนย์กลางการอัปโหลดแบบกระจาย (Distributed Upload Centers):** ใช้ CDN เป็นจุดอัปโหลดที่อยู่ใกล้ผู้ใช้
3. **การประมวลผลแบบขนาน (Parallel Processing):** แยกส่วน (decouple) โมดูลต่าง ๆ ออกจากกันโดยใช้ message queue เพื่อให้เกิด parallelism สูง

    <img src="./images/message-queue1.png" alt="Message Queue" width="600">
    <img src="./images/message-queue2.png" alt="Message Queue" height="170" width="500">

### การปรับปรุงด้านความปลอดภัย (Safety Optimizations)
1. **Pre-Signed URLs:** จำกัดการอัปโหลดวิดีโอให้ทำได้เฉพาะผู้ใช้ที่ได้รับอนุญาต

    <img src="./images/pres-signed-urls.png" alt="Pre Signed" width="500">

2. **การปกป้องวิดีโอ (Protect Videos):**
   - **ระบบ DRM** (เช่น Apple FairPlay, Google Widevine)
   - **การเข้ารหัส AES (AES Encryption)**
   - **การใส่ Watermark**

### การปรับปรุงด้านการประหยัดค่าใช้จ่าย (Cost-Saving Optimizations)
1. ให้บริการเฉพาะวิดีโอที่ได้รับความนิยมผ่าน CDN ส่วนวิดีโอที่ไม่ค่อยได้รับความนิยมให้บริการจากเซิร์ฟเวอร์ที่มีความจุสูง
2. เข้ารหัสตามความต้องการ (on-demand) สำหรับวิดีโอที่แทบไม่มีคนเข้าถึง
3. กระจายวิดีโอตามภูมิภาค (regionalize) โดยอิงตามความนิยม
4. สร้าง CDN แบบกำหนดเอง (custom) และร่วมมือกับ ISP เพื่อลดค่าใช้จ่ายด้าน bandwidth

---

## การจัดการข้อผิดพลาด (Error Handling)
### ข้อผิดพลาดที่สามารถกู้คืนได้ (Recoverable Errors)
- ทำการ retry การอัปโหลด, การ transcode, หรือ task การจัดสรรทรัพยากรที่ล้มเหลว

### ข้อผิดพลาดที่ไม่สามารถกู้คืนได้ (Non-Recoverable Errors)
- หยุดการประมวลผลวิดีโอที่มีรูปแบบผิดพลาด (malformed) และส่งคืนรหัสข้อผิดพลาด (error code)
