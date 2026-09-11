# บทที่ 20: ระบบตรวจสอบเมตริกและการแจ้งเตือน (Metrics Monitoring and Alerting System)

## บทนำ
บทนี้มุ่งเน้นการออกแบบ **ระบบตรวจสอบเมตริกและการแจ้งเตือน (Metrics Monitoring and Alerting System)** ที่สามารถขยายตัว (scale) ได้สูง ซึ่งมีความสำคัญอย่างยิ่งต่อการรักษาความพร้อมใช้งานสูง (high availability) และความน่าเชื่อถือ (reliability) ของระบบ

---

## หัวข้อที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)
ระบบตรวจสอบเมตริกสามารถหมายถึงสิ่งที่แตกต่างกันได้หลายอย่าง เช่น เราอาจไม่ต้องการออกแบบระบบรวบรวม log (logs aggregation system) หากผู้สัมภาษณ์สนใจเฉพาะเมตริกของ infrastructure เท่านั้น

มาลองทำความเข้าใจปัญหากันก่อน:
 - C: เรากำลังสร้างระบบนี้ให้ใคร? เป็นระบบตรวจสอบภายในสำหรับบริษัทเทคโนโลยีขนาดใหญ่ หรือเป็น SaaS อย่าง DataDog?
 - I: เรากำลังสร้างสำหรับใช้งานภายในเท่านั้น
 - C: เราต้องการรวบรวมเมตริกแบบใดบ้าง?
 - I: เมตริกของระบบปฏิบัติการ (operational system metrics) เช่น CPU load, Memory, พื้นที่ดิสก์ (data disk space) แต่ก็รวมถึงเมตริกระดับสูง (high-level metrics) เช่น จำนวน request ต่อวินาที ส่วนเมตริกทางธุรกิจ (business metrics) ไม่อยู่ในขอบเขต
 - C: ขนาดของ infrastructure ที่เราจะตรวจสอบมีเท่าใด?
 - I: ผู้ใช้งานประจำวัน (DAU) 100 ล้านคน, server pool 1000 ชุด, เครื่อง 100 เครื่องต่อ pool
 - C: เราควรเก็บข้อมูลไว้นานเท่าใด?
 - I: สมมติว่าเก็บไว้ 1 ปี
 - C: เราสามารถลดความละเอียด (resolution) ของข้อมูลเมตริกสำหรับการจัดเก็บระยะยาวได้หรือไม่?
 - I: เก็บเมตริกที่เพิ่งได้รับไว้ 7 วัน จากนั้น roll up เป็นความละเอียด 1 นาทีในอีก 30 วันถัดไป และ roll up ต่อไปเป็นความละเอียด 1 ชั่วโมงหลังจากผ่านไป 30 วัน
 - C: ช่องทางการแจ้งเตือนที่รองรับมีอะไรบ้าง?
 - I: Email, โทรศัพท์, PagerDuty หรือ webhook
 - C: เราต้องรวบรวม log เช่น error log หรือ access log หรือไม่?
 - I: ไม่
 - C: เราต้องรองรับ distributed system tracing หรือไม่?
 - I: ไม่

### **ความต้องการระดับสูงและข้อสมมติฐาน (High-level requirements and assumptions)**
Infrastructure ที่ถูกตรวจสอบมีขนาดใหญ่มาก:
 - DAU 100 ล้านคน
 - server pool 1000 ชุด * เครื่อง 100 เครื่อง * ~100 เมตริกต่อเครื่อง -> ~10 ล้านเมตริก
 - เก็บรักษาข้อมูล (data retention) 1 ปี
 - นโยบายการเก็บรักษาข้อมูล - ข้อมูลดิบเก็บ 7 วัน, ความละเอียด 1 นาทีเก็บ 30 วัน, ความละเอียด 1 ชั่วโมงเก็บ 1 ปี

เมตริกที่สามารถตรวจสอบได้มีความหลากหลาย:
 - CPU load
 - จำนวน request
 - การใช้งานหน่วยความจำ (memory usage)
 - จำนวนข้อความใน message queue

### **ความต้องการที่ไม่ใช่ฟังก์ชัน (Non-functional requirements)**
 - **ความสามารถในการขยาย (Scalability)**: ระบบควรขยายได้เพื่อรองรับเมตริกและการแจ้งเตือนที่เพิ่มขึ้น
 - **ความหน่วงต่ำ (Low latency)**: ระบบต้องมี query latency ต่ำสำหรับ dashboard และการแจ้งเตือน
 - **ความน่าเชื่อถือ (Reliability)**: ระบบต้องมีความน่าเชื่อถือสูงเพื่อหลีกเลี่ยงการพลาดการแจ้งเตือนที่สำคัญ
 - **ความยืดหยุ่น (Flexibility)**: ระบบควรสามารถผสานเทคโนโลยีใหม่ ๆ ในอนาคตได้อย่างง่ายดาย

มีความต้องการใดบ้างที่อยู่นอกขอบเขต?
 - **การตรวจสอบ log (Log monitoring)**: ELK stack เป็นที่นิยมมากสำหรับกรณีการใช้งานนี้
 - **Distributed system tracing**: หมายถึงการรวบรวมข้อมูลเกี่ยวกับวงจรชีวิตของ request ขณะที่ไหลผ่านหลายบริการภายในระบบ

---

## หัวข้อที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

### **พื้นฐาน (Fundamentals)**
มีคอมโพเนนต์หลัก 5 อย่างที่เกี่ยวข้องในระบบตรวจสอบเมตริกและการแจ้งเตือน:

<div style="margin-left:3rem">
    <img src="./images/metrics-monitoring-core-components.png" alt="metrics-monitoring-core-components" width="500" />
</div>

 - **การรวบรวมข้อมูล (Data collection)**: รวบรวมข้อมูลเมตริกจากแหล่งต่าง ๆ
 - **การส่งข้อมูล (Data transmission)**: ส่งข้อมูลจากแหล่งที่มาไปยังระบบตรวจสอบเมตริก
 - **การจัดเก็บข้อมูล (Data storage)**: จัดระเบียบและจัดเก็บข้อมูลที่เข้ามา
 - **การแจ้งเตือน (Alerting)**: วิเคราะห์ข้อมูลที่เข้ามา ตรวจจับความผิดปกติ (anomaly) และสร้างการแจ้งเตือน
 - **การแสดงผล (Visualization)**: นำเสนอข้อมูลในรูปแบบกราฟ, แผนภูมิ เป็นต้น

### **โมเดลข้อมูล (Data model)**
ข้อมูลเมตริกมักถูกบันทึกในรูปแบบอนุกรมเวลา (time-series) ซึ่งประกอบด้วยชุดค่าพร้อม timestamp
อนุกรม (series) สามารถระบุได้ด้วยชื่อและชุดของแท็ก (tag) ซึ่งเป็นทางเลือก

ตัวอย่างที่ 1 - CPU load บนเซิร์ฟเวอร์ production instance i631 ที่เวลา 20:00 เป็นเท่าใด?

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1.png" alt="metrics-example-1" width="500" />
</div>

ข้อมูลสามารถระบุได้ด้วยตารางต่อไปนี้:

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1-data.png" alt="metrics-example-1-data" width="500" />
</div>

อนุกรมเวลา (time series) ถูกระบุด้วยชื่อเมตริก, label และจุดข้อมูลเดี่ยว ณ เวลาที่ระบุ

ตัวอย่างที่ 2 - ค่าเฉลี่ย CPU load ของเว็บเซิร์ฟเวอร์ทั้งหมดในภูมิภาค us-west ในช่วง 10 นาทีที่ผ่านมาเป็นเท่าใด?

```
CPU.load host=webserver01,region=us-west 1613707265 50

CPU.load host=webserver01,region=us-west 1613707265 62

CPU.load host=webserver02,region=us-west 1613707265 43

CPU.load host=webserver02,region=us-west 1613707265 53

...

CPU.load host=webserver01,region=us-west 1613707265 76

CPU.load host=webserver01,region=us-west 1613707265 83
```

นี่คือตัวอย่างข้อมูลที่เราอาจดึงมาจาก storage เพื่อตอบคำถามนั้น
ค่าเฉลี่ย CPU load สามารถคำนวณได้จากการเฉลี่ยค่าในคอลัมน์สุดท้ายของแต่ละแถว

รูปแบบที่แสดงข้างต้นเรียกว่า line protocol และถูกใช้โดยซอฟต์แวร์ตรวจสอบระบบที่ได้รับความนิยมหลายตัวในตลาด เช่น Prometheus, OpenTSDB

สิ่งที่ time series แต่ละตัวประกอบด้วย:

<div style="margin-left:3rem">
    <img src="./images/time-series-data-example.png" alt="time-series-data-example" width="500" />
</div>

วิธีที่ดีในการแสดงภาพว่าข้อมูลมีลักษณะอย่างไร:

<div style="margin-left:3rem">
    <img src="./images/time-series-data-viz.png" alt="time-series-data-viz" width="500" />
</div>

 - แกน x คือเวลา
 - แกน y คือมิติ (dimension) ที่คุณกำลัง query - เช่น ชื่อเมตริก, tag เป็นต้น

รูปแบบการเข้าถึงข้อมูล (data access pattern) เป็นแบบเขียนหนัก (write-heavy) และอ่านแบบพุ่งสูงเป็นช่วง ๆ (spiky reads) เนื่องจากเรารวบรวมเมตริกจำนวนมาก แต่ถูกเข้าถึงไม่บ่อยนัก แม้ว่าจะเข้าถึงเป็น burst เมื่อ เช่น มีเหตุการณ์ผิดปกติ (incident) เกิดขึ้น

ระบบจัดเก็บข้อมูล (data storage system) คือหัวใจของการออกแบบนี้
 - ไม่แนะนำให้ใช้ฐานข้อมูลอเนกประสงค์ (general-purpose database) สำหรับปัญหานี้ แม้ว่าคุณจะสามารถบรรลุ scale ที่ดีได้ด้วยการ tuning ระดับผู้เชี่ยวชาญ
 - การใช้ฐานข้อมูล NoSQL อาจใช้ได้ในทางทฤษฎี แต่ยากที่จะออกแบบ schema ที่ขยายได้เพื่อจัดเก็บและ query ข้อมูล time-series อย่างมีประสิทธิภาพ

มีฐานข้อมูลจำนวนมากที่ถูกออกแบบมาโดยเฉพาะสำหรับจัดเก็บข้อมูล time-series หลายตัวรองรับ query interface แบบกำหนดเอง (custom) ซึ่งช่วยให้ query ข้อมูล time-series ได้อย่างมีประสิทธิภาพ
 - OpenTSDB เป็นฐานข้อมูล time-series แบบกระจาย (distributed) แต่อิงอยู่บน Hadoop และ HBase หากคุณไม่มี infrastructure เหล่านั้นพร้อมใช้งาน ก็จะยากที่จะใช้เทคโนโลยีนี้
 - Twitter ใช้ MetricsDB ในขณะที่ Amazon เสนอ Timestream
 - ฐานข้อมูล time-series ที่ได้รับความนิยมมากที่สุดสองตัวคือ InfluxDB และ Prometheus
 - ทั้งสองถูกออกแบบมาเพื่อจัดเก็บข้อมูล time-series ปริมาณมหาศาล ทั้งคู่อิงอยู่บน in-memory cache ร่วมกับการจัดเก็บบนดิสก์ (on-disk storage)

ตัวอย่างขนาด scale ของ InfluxDB - มากกว่า 250,000 การเขียนต่อวินาที เมื่อจัดเตรียมด้วย 8 core และ RAM 32GB:

<div style="margin-left:3rem">
    <img src="./images/influxdb-scale.png" alt="influxdb-scale" width="500" />
</div>

ไม่คาดหวังให้คุณเข้าใจกลไกภายใน (internals) ของฐานข้อมูลเมตริก เนื่องจากเป็นความรู้เฉพาะทาง (niche knowledge) คุณอาจถูกถามเรื่องนี้ก็ต่อเมื่อคุณระบุไว้ในเรซูเม่เท่านั้น

สำหรับวัตถุประสงค์ของการสัมภาษณ์ เพียงพอแล้วที่จะเข้าใจว่าเมตริกคือข้อมูล time-series และตระหนักถึงฐานข้อมูล time-series ที่ได้รับความนิยม เช่น InfluxDB

คุณสมบัติที่ดีอย่างหนึ่งของฐานข้อมูล time-series คือการรวม (aggregation) และวิเคราะห์ข้อมูล time-series ปริมาณมากตาม label ได้อย่างมีประสิทธิภาพ
ตัวอย่างเช่น InfluxDB สร้าง index สำหรับแต่ละ label

อย่างไรก็ตาม เป็นสิ่งสำคัญอย่างยิ่งที่จะต้องรักษาค่า cardinality ของ label ให้ต่ำ - กล่าวคือ ไม่ใช้ label ที่มีค่าไม่ซ้ำ (unique) มากเกินไป

### **การออกแบบระดับสูง (High-level Design)**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

 - **แหล่งที่มาของเมตริก (Metrics source)**: อาจเป็น application server, ฐานข้อมูล SQL, message queue เป็นต้น
 - **ตัวรวบรวมเมตริก (Metrics collector)**: รวบรวมข้อมูลเมตริกและเขียนลงในฐานข้อมูล time-series
 - **ฐานข้อมูล Time-series**: จัดเก็บเมตริกในรูปแบบ time-series พร้อมให้บริการ query interface แบบกำหนดเองสำหรับวิเคราะห์เมตริกจำนวนมาก
 - **บริการ Query (Query service)**: ทำให้การ query และดึงข้อมูลจากฐานข้อมูล time-series ทำได้ง่ายขึ้น อาจถูกแทนที่ทั้งหมดด้วย interface ของฐานข้อมูลเอง หากมันมีความสามารถเพียงพอ
 - **ระบบแจ้งเตือน (Alerting system)**: ส่งการแจ้งเตือนไปยังปลายทางต่าง ๆ
 - **ระบบแสดงผล (Visualization system)**: แสดงเมตริกในรูปแบบกราฟ/แผนภูมิ

---

## หัวข้อที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)
มาเจาะลึกส่วนที่น่าสนใจมากขึ้นของระบบกัน

### **การรวบรวมเมตริก (Metrics collection)**
สำหรับการรวบรวมเมตริก การสูญเสียข้อมูลเป็นครั้งคราวไม่ใช่เรื่องวิกฤต เป็นที่ยอมรับได้ที่ไคลเอนต์จะส่งข้อมูลแบบ fire-and-forget

<div style="margin-left:3rem">
    <img src="./images/metrics-collection.png" alt="metrics-collection" width="500" />
</div>

มีสองวิธีในการดำเนินการรวบรวมเมตริก - แบบ pull หรือ push

นี่คือลักษณะของโมเดล pull:

<div style="margin-left:3rem">
    <img src="./images/pull-model-example.png" alt="pull-model-example" width="500" />
</div>

สำหรับโซลูชันนี้ ตัวรวบรวมเมตริกต้องคงรายการบริการและ endpoint ของเมตริกที่เป็นปัจจุบันไว้เสมอ
เราสามารถใช้ Zookeeper หรือ etcd เพื่อวัตถุประสงค์นี้ได้ - การค้นพบบริการ (service discovery)

Service discovery มีกฎการตั้งค่า (configuration rules) เกี่ยวกับเวลาและตำแหน่งในการรวบรวมเมตริก:

<div style="margin-left:3rem">
    <img src="./images/service-discovery-example.png" alt="service-discovery-example" width="500" />
</div>

นี่คือคำอธิบายโดยละเอียดของขั้นตอนการรวบรวมเมตริก:

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-flow.png" alt="metrics-collection-flow" width="500" />
</div>

 - ตัวรวบรวมเมตริก (metrics collector) ดึง configuration metadata จาก service discovery ซึ่งรวมถึง pulling interval, IP address, timeout และพารามิเตอร์การ retry
 - ตัวรวบรวมเมตริก ดึงข้อมูลเมตริกผ่าน http endpoint ที่กำหนดไว้ล่วงหน้า (เช่น `/metrics`) โดยทั่วไปดำเนินการผ่าน client library
 - หรืออีกทางหนึ่ง ตัวรวบรวมเมตริกสามารถลงทะเบียนการแจ้งเตือนเหตุการณ์การเปลี่ยนแปลง (change event notification) กับ service discovery เพื่อรับการแจ้งเตือนเมื่อ endpoint ของบริการเปลี่ยนแปลง
 - อีกทางเลือกหนึ่งคือให้ตัวรวบรวมเมตริก poll การเปลี่ยนแปลง configuration ของ endpoint เมตริกเป็นระยะ

ที่ scale ของเรา ตัวรวบรวมเมตริกเพียงตัวเดียวไม่เพียงพอ จำเป็นต้องมีหลาย instance
อย่างไรก็ตาม ก็จำเป็นต้องมีการซิงโครไนซ์ (synchronization) บางอย่างระหว่างพวกมัน เพื่อไม่ให้ตัวรวบรวมสองตัวรวบรวมเมตริกเดียวกันซ้ำกัน

โซลูชันหนึ่งสำหรับปัญหานี้คือการวางตัวรวบรวมและเซิร์ฟเวอร์บน consistent hash ring และเชื่อมโยงชุดของเซิร์ฟเวอร์เข้ากับตัวรวบรวมเพียงตัวเดียว:

<div style="margin-left:3rem">
    <img src="./images/consistent-hash-ring.png" alt="consistent-hash-ring" width="500" />
</div>

ในทางกลับกัน ในโมเดล push บริการต่าง ๆ จะ push เมตริกของตนเองไปยังตัวรวบรวมเมตริกเชิงรุก (proactively):

<div style="margin-left:3rem">
    <img src="./images/push-model-example.png" alt="push-model-example" width="500" />
</div>

ในแนวทางนี้ โดยทั่วไปจะมีการติดตั้ง collection agent ควบคู่ไปกับ service instance ต่าง ๆ
Agent จะรวบรวมเมตริกจากเซิร์ฟเวอร์และ push เมตริกเหล่านั้นไปยังตัวรวบรวมเมตริก

<div style="margin-left:3rem">
    <img src="./images/metrics-collector-agent.png" alt="metrics-collector-agent" width="500" />
</div>

ด้วยโมเดลนี้ เราสามารถรวม (aggregate) เมตริกก่อนที่จะส่งไปยังตัวรวบรวมได้ ซึ่งจะช่วยลดปริมาณข้อมูลที่ตัวรวบรวมต้องประมวลผล

ในอีกด้านหนึ่ง ตัวรวบรวมเมตริกอาจปฏิเสธคำขอ push เนื่องจากไม่สามารถรองรับ load ได้
ดังนั้นจึงสำคัญที่จะต้องเพิ่มตัวรวบรวมเข้าไปใน auto-scaling group ที่อยู่หลัง load balancer

แล้วแบบไหนดีกว่ากัน? มี trade-off ระหว่างทั้งสองแนวทาง และระบบต่าง ๆ ก็ใช้แนวทางที่แตกต่างกัน:
 - Prometheus ใช้สถาปัตยกรรมแบบ pull
 - Amazon Cloud Watch และ Graphite ใช้สถาปัตยกรรมแบบ push

ต่อไปนี้คือความแตกต่างหลัก ๆ ระหว่าง push และ pull:
|                                        | Pull                                                                                                                                                                                                    | Push                                                                                                                                                                                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| การ debug ที่ทำได้ง่าย                         | endpoint `/metrics` บน application server ที่ใช้สำหรับ pull เมตริกสามารถใช้ดูเมตริกได้ตลอดเวลา คุณสามารถทำได้แม้กระทั่งบนแล็ปท็อปของคุณ Pull ชนะ                                          | หากตัวรวบรวมเมตริกไม่ได้รับเมตริก ปัญหาอาจเกิดจากปัญหาเครือข่าย                                                                                                                                        |
| การตรวจสุขภาพ (Health check)                           | หาก application server ไม่ตอบสนองต่อการ pull คุณสามารถระบุได้อย่างรวดเร็วว่า application server นั้นล่มหรือไม่ Pull ชนะ                                                                           | หากตัวรวบรวมเมตริกไม่ได้รับเมตริก ปัญหาอาจเกิดจากปัญหาเครือข่าย                                                                                                                                        |
| งานที่มีอายุสั้น (Short-lived jobs)                       |                                                                                                                                                                                                         | batch job บางงานอาจมีอายุสั้นและไม่ยืนยาวพอที่จะถูก pull ได้ Push ชนะ ปัญหานี้สามารถแก้ไขได้โดยการนำ push gateway มาใช้กับโมเดล pull [22]                                                                 |
| Firewall หรือการตั้งค่าเครือข่ายที่ซับซ้อน | การให้เซิร์ฟเวอร์ pull เมตริกต้องการให้ endpoint ของเมตริกทั้งหมดสามารถเข้าถึงได้ (reachable) ซึ่งอาจเป็นปัญหาในสถานการณ์ที่มีหลาย data center และอาจต้องการ network infrastructure ที่ซับซ้อนมากขึ้น | หากตัวรวบรวมเมตริกถูกตั้งค่าด้วย load balancer และ auto-scaling group ก็เป็นไปได้ที่จะรับข้อมูลจากที่ใดก็ได้ Push ชนะ                                                                                                                                             |
| ประสิทธิภาพ (Performance)                            | วิธี Pull โดยทั่วไปใช้ TCP                                                                                                                                                                         | วิธี Push โดยทั่วไปใช้ UDP ซึ่งหมายความว่าวิธี push ให้การขนส่งเมตริกที่มี latency ต่ำกว่า ข้อโต้แย้งในที่นี้คือ ความพยายามในการสร้างการเชื่อมต่อ TCP นั้นน้อยมากเมื่อเทียบกับการส่ง payload ของเมตริก |
| ความถูกต้องแท้จริงของข้อมูล (Data authenticity)                      | application server ที่จะเก็บรวบรวมเมตริกถูกกำหนดไว้ล่วงหน้าใน config file เมตริกที่รวบรวมจากเซิร์ฟเวอร์เหล่านั้นได้รับการรับประกันว่าเป็นของแท้ (authentic)                                                 | ไคลเอนต์ประเภทใดก็ได้สามารถ push เมตริกไปยังตัวรวบรวมเมตริกได้ ปัญหานี้สามารถแก้ไขได้ด้วยการทำ whitelist เซิร์ฟเวอร์ที่จะรับเมตริกจาก หรือด้วยการกำหนดให้ต้องมีการยืนยันตัวตน (authentication)                                                                   |

ไม่มีผู้ชนะที่ชัดเจน องค์กรขนาดใหญ่มักจำเป็นต้องรองรับทั้งสองแบบ อาจไม่มีทางที่จะติดตั้ง push agent ได้ตั้งแต่แรกในบางกรณี

### **การขยาย pipeline การส่งเมตริก (Scale the metrics transmission pipeline)**

<div style="margin-left:3rem">
    <img src="./images/metrics-transmission-pipeline.png" alt="metrics-transmission-pipeline" width="500" />
</div>

ตัวรวบรวมเมตริกถูกจัดเตรียมไว้ใน auto-scaling group ไม่ว่าเราจะใช้โมเดล push หรือ pull ก็ตาม

อย่างไรก็ตาม มีโอกาสที่ข้อมูลจะสูญหายได้หากฐานข้อมูล time-series ล่ม เพื่อลดผลกระทบนี้ เราจะจัดเตรียมกลไกคิว (queuing mechanism):

<div style="margin-left:3rem">
    <img src="./images/queuing-mechanism.png" alt="queuing-mechanism" width="500" />
</div>

 - ตัวรวบรวมเมตริก push ข้อมูลเมตริกเข้าไปใน Kafka
 - Consumer หรือบริการประมวลผลแบบ stream เช่น Apache Storm, Flink หรือ Spark ประมวลผลข้อมูลและ push ไปยังฐานข้อมูล time-series

แนวทางนี้มีข้อดีหลายประการ:
 - Kafka ถูกใช้เป็นแพลตฟอร์มส่งข้อความแบบกระจาย (distributed message platform) ที่มีความน่าเชื่อถือสูงและขยายได้
 - ช่วยแยก (decouple) การรวบรวมข้อมูลและการประมวลผลข้อมูลออกจากกัน
 - สามารถป้องกันการสูญเสียข้อมูลได้ ด้วยการเก็บรักษาข้อมูลไว้ใน Kafka

Kafka สามารถถูกตั้งค่าให้มีหนึ่ง partition ต่อชื่อเมตริก เพื่อให้ consumer สามารถรวม (aggregate) ข้อมูลตามชื่อเมตริกได้
เพื่อขยาย (scale) ต่อไปอีก เราสามารถแบ่ง partition เพิ่มเติมตาม tag/label และจัดหมวดหมู่/จัดลำดับความสำคัญของเมตริกที่จะถูกรวบรวมก่อน

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-kafka.png" alt="metrics-collection-kafka" width="500" />
</div>

ข้อเสียหลักของการใช้ Kafka สำหรับปัญหานี้คือภาระในการดูแลรักษา/ดำเนินงาน (maintenance/operation overhead)
ทางเลือกหนึ่งคือการใช้ระบบนำเข้าข้อมูลขนาดใหญ่ (large-scale ingestion system) อย่าง [Gorilla](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
อาจกล่าวได้ว่าการใช้ระบบดังกล่าวสามารถขยายได้ (scalable) เทียบเท่ากับการใช้ Kafka สำหรับการทำคิว

### **ตำแหน่งที่การรวมข้อมูลสามารถเกิดขึ้นได้ (Where aggregations can happen)**
เมตริกสามารถถูกรวม (aggregate) ได้ในหลายจุด มี trade-off ระหว่างตัวเลือกต่าง ๆ:
 - **Collection agent**: collection agent ฝั่งไคลเอนต์รองรับเพียง logic การรวมข้อมูลแบบง่าย ๆ เท่านั้น เช่น เก็บ counter เป็นเวลา 1 นาทีแล้วส่งไปยังตัวรวบรวมเมตริก
 - **Ingestion pipeline**: การจะรวมข้อมูลก่อนเขียนลงฐานข้อมูล เราต้องใช้ stream processing engine อย่าง Flink วิธีนี้ช่วยลดปริมาณการเขียน แต่เราจะสูญเสียความละเอียดของข้อมูล เนื่องจากเราไม่ได้เก็บข้อมูลดิบ
 - **ฝั่ง Query**: เราสามารถรวมข้อมูลได้เมื่อรัน query ผ่านระบบแสดงผลของเรา วิธีนี้ไม่มีการสูญเสียข้อมูล แต่ query อาจช้าเนื่องจากต้องประมวลผลข้อมูลจำนวนมาก

### **บริการ Query (Query Service)**
การแยกบริการ query ออกจากฐานข้อมูล time-series ช่วยแยก (decouple) ระบบแสดงผลและระบบแจ้งเตือนออกจากฐานข้อมูล ซึ่งช่วยให้เราสามารถแยกฐานข้อมูลออกจากไคลเอนต์และเปลี่ยนแปลงมันได้ตามต้องการ

เราสามารถเพิ่มชั้น cache ตรงนี้เพื่อลดภาระบนฐานข้อมูล time-series ได้:

<div style="margin-left:3rem">
    <img src="./images/cache-layer-query-service.png" alt="cache-layer-query-service" width="500" />
</div>

เรายังสามารถหลีกเลี่ยงการเพิ่มบริการ query ไปเลยก็ได้ เนื่องจากระบบแสดงผลและแจ้งเตือนส่วนใหญ่มีปลั๊กอินที่ทรงพลังในการผสานกับฐานข้อมูล time-series ส่วนใหญ่
หากเลือกฐานข้อมูล time-series ได้ดี เราก็อาจไม่จำเป็นต้องนำเสนอชั้น cache ของเราเองเพิ่มเติมด้วยเช่นกัน

ฐานข้อมูล time-series ส่วนใหญ่ไม่รองรับ SQL ด้วยเหตุผลง่าย ๆ ว่ามันไม่มีประสิทธิภาพสำหรับการ query ข้อมูล time-series นี่คือตัวอย่าง SQL query สำหรับคำนวณ exponential moving average:

```
select id,
       temp,
       avg(temp) over (partition by group_nr order by time_read) as rolling_avg
from (
  select id,
         temp,
         time_read,
         interval_group,
         id - row_number() over (partition by interval_group order by time_read) as group_nr
  from (
    select id,
    time_read,
    "epoch"::timestamp + "900 seconds"::interval * (extract(epoch from time_read)::int4 / 900) as interval_group,
    temp
    from readings
  ) t1
) t2
order by time_read;
```

นี่คือ query เดียวกันในภาษา Flux - query language ที่ใช้ใน InfluxDB:

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### **ชั้นการจัดเก็บข้อมูล (Storage layer)**
สำคัญมากที่จะต้องเลือกฐานข้อมูล time-series อย่างระมัดระวัง

จากงานวิจัยที่เผยแพร่โดย Facebook พบว่า ~85% ของ query ที่ส่งไปยัง operational store เป็นการ query ข้อมูลของ 26 ชั่วโมงที่ผ่านมา

หากเราเลือกฐานข้อมูลที่ใช้ประโยชน์จากคุณสมบัตินี้ได้ ก็อาจส่งผลกระทบอย่างมีนัยสำคัญต่อประสิทธิภาพของระบบ InfluxDB เป็นหนึ่งในตัวเลือกดังกล่าว

ไม่ว่าเราจะเลือกฐานข้อมูลใดก็ตาม ก็ยังมี optimization บางอย่างที่เราสามารถนำมาใช้ได้

การเข้ารหัสข้อมูล (data encoding) และการบีบอัด (compression) สามารถลดขนาดข้อมูลได้อย่างมีนัยสำคัญ คุณสมบัติเหล่านี้มักถูกสร้างไว้ในตัวของฐานข้อมูล time-series ที่ดีอยู่แล้ว

<div style="margin-left:3rem">
    <img src="./images/double-delta-encoding.png" alt="double-delta-encoding" width="500" />
</div>

ในตัวอย่างข้างต้น แทนที่จะเก็บ timestamp เต็มรูปแบบ เราสามารถเก็บผลต่าง (delta) ของ timestamp แทนได้

อีกเทคนิคหนึ่งที่เราสามารถใช้ได้คือ การลดความละเอียด (down-sampling) - การแปลงข้อมูลความละเอียดสูงให้เป็นความละเอียดต่ำ เพื่อลดการใช้พื้นที่ดิสก์

เราสามารถใช้วิธีนี้กับข้อมูลเก่า และทำให้กฎเกณฑ์สามารถกำหนดค่าได้ (configurable) โดยนักวิทยาศาสตร์ข้อมูล (data scientist) เช่น:
 - 7 วัน - ไม่มีการ down-sampling
 - 30 วัน - down-sample เป็นความละเอียด 1 นาที
 - 1 ปี - down-sample เป็นความละเอียด 1 ชั่วโมง

ตัวอย่างเช่น นี่คือตารางเมตริกที่มีความละเอียด 10 วินาที:
| metric | timestamp            | hostname | Metric_value |
|--------|----------------------|----------|--------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 10           |
| cpu    | 2021-10-24T19:00:10Z | host-a   | 16           |
| cpu    | 2021-10-24T19:00:20Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 30           |
| cpu    | 2021-10-24T19:00:40Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:50Z | host-a   | 30           |

หลังจาก down-sample เป็นความละเอียด 30 วินาที:
| metric | timestamp            | hostname | Metric_value (avg) |
|--------|----------------------|----------|--------------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 19                 |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 25                 |

สุดท้ายนี้ เรายังสามารถใช้ cold storage สำหรับข้อมูลเก่าที่ไม่ได้ใช้งานแล้วได้ ต้นทุนทางการเงินของ cold storage นั้นต่ำกว่ามาก

### **ระบบแจ้งเตือน (Alerting system)**

<div style="margin-left:3rem">
    <img src="./images/alerting-system.png" alt="alerting-system" width="500" />
</div>

Configuration ถูกโหลดไปยัง cache server กฎ (rule) มักถูกกำหนดในรูปแบบ YAML ต่อไปนี้คือตัวอย่าง:

```
- name: instance_down
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

Alert manager ดึง alert configuration จาก cache โดยยึดตามกฎการตั้งค่า (configuration rules) มันจะเรียกบริการ query ในช่วงเวลาที่กำหนดไว้ล่วงหน้าด้วยเช่นกัน
หากกฎเป็นจริงตามเงื่อนไข alert event จะถูกสร้างขึ้น

ความรับผิดชอบอื่น ๆ ของ alert manager ได้แก่:
 - การกรอง (filtering), การผสาน (merging) และการลบข้อมูลซ้ำ (deduplicating) ของการแจ้งเตือน เช่น หากการแจ้งเตือนของ instance เดียวถูกทริกเกอร์หลายครั้ง จะมีเพียง alert event เดียวเท่านั้นที่ถูกสร้างขึ้น
 - การควบคุมการเข้าถึง (Access control) - สำคัญมากที่จะจำกัดการดำเนินการจัดการการแจ้งเตือน (alert-management operations) ให้เฉพาะบุคคลบางกลุ่มเท่านั้น
 - การ Retry - manager รับประกันว่าการแจ้งเตือนจะถูกส่งต่อ (propagated) อย่างน้อยหนึ่งครั้ง

Alert store คือฐานข้อมูลแบบ key-value อย่าง Cassandra ซึ่งเก็บสถานะของการแจ้งเตือนทั้งหมด มันรับประกันว่าการแจ้งเตือนจะถูกส่งอย่างน้อยหนึ่งครั้ง
เมื่อการแจ้งเตือนถูกทริกเกอร์ มันจะถูก publish ไปยัง Kafka

สุดท้ายนี้ alert consumer จะดึงข้อมูลการแจ้งเตือนจาก Kafka และส่งการแจ้งเตือนไปยังช่องทางต่าง ๆ - Email, ข้อความ (text message), PagerDuty, webhook

ในโลกความเป็นจริง มีโซลูชันสำเร็จรูป (off-the-shelf) มากมายสำหรับระบบแจ้งเตือน ยากที่จะให้เหตุผลสนับสนุนการสร้างระบบขึ้นเองภายในองค์กร

### **ระบบแสดงผล (Visualization system)**
ระบบแสดงผลแสดงเมตริกและการแจ้งเตือนในช่วงเวลาหนึ่ง นี่คือตัวอย่าง dashboard ที่สร้างด้วย Grafana:

<div style="margin-left:3rem">
    <img src="./images/grafana-dashboard.png" alt="grafana-dashboard" width="500" />
</div>

ระบบแสดงผลคุณภาพสูงสร้างได้ยากมาก ยากที่จะให้เหตุผลสนับสนุนการไม่ใช้โซลูชันสำเร็จรูปอย่าง Grafana

---

## หัวข้อที่ 4: สรุปส่งท้าย (Wrap up)
นี่คือการออกแบบขั้นสุดท้ายของเรา:

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>
