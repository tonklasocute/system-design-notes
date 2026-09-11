# บทที่ 28: ตลาดหลักทรัพย์ (Stock Exchange)

## บทนำ
เราจะออกแบบ **ตลาดหลักทรัพย์อิเล็กทรอนิกส์ (electronic stock exchange)** ในบทนี้

หน้าที่พื้นฐานของมันคือการจับคู่ (match) ผู้ซื้อและผู้ขายอย่างมีประสิทธิภาพ

ตลาดหลักทรัพย์รายใหญ่ ได้แก่ **NYSE**, **NASDAQ** และอื่น ๆ

<div style="margin-left:3rem">
    <img src="./images/world-stock-exchanges.png" alt="world-stock-exchanges" width="500" />
</div>

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)
 * C: เราจะซื้อขายหลักทรัพย์ประเภทใด? หุ้น, option หรือ futures?
 * I: ให้เป็นหุ้น (stocks) เท่านั้นเพื่อความง่าย
 * C: ประเภทคำสั่งซื้อขาย (order type) ใดบ้างที่รองรับ - วาง (place), ยกเลิก (cancel), แทนที่ (replace)? แล้ว limit, market, conditional order ล่ะ?
 * I: เราต้องรองรับการวางและยกเลิกคำสั่ง เราจะพิจารณาเฉพาะ limit order เท่านั้นสำหรับประเภทคำสั่ง
 * C: ระบบต้องรองรับการซื้อขายนอกเวลาทำการ (after hours trading) หรือไม่?
 * I: ไม่ เฉพาะเวลาซื้อขายปกติเท่านั้น
 * C: คุณช่วยอธิบายฟังก์ชันพื้นฐานของตลาดหลักทรัพย์ได้ไหม?
 * I: ลูกค้าสามารถวางหรือยกเลิก limit order และรับผลการจับคู่ (matched trades) แบบ real-time พวกเขาควรสามารถเห็น order book แบบ real-time ได้ด้วย
 * C: ขนาดของตลาดหลักทรัพย์นี้เป็นเท่าไร?
 * I: ผู้ใช้หลายหมื่นคนซื้อขายพร้อมกันและมีประมาณ 100 สัญลักษณ์หุ้น (symbol) หลายพันล้านคำสั่งต่อวัน เรายังต้องรองรับการตรวจสอบความเสี่ยง (risk check) เพื่อการปฏิบัติตามกฎระเบียบด้วย
 * C: การตรวจสอบความเสี่ยงแบบไหน?
 * I: ให้เป็นการตรวจสอบความเสี่ยงแบบง่าย ๆ เช่น จำกัดผู้ใช้ให้ซื้อขายหุ้น Apple ได้ไม่เกิน 1 ล้านหุ้นต่อวัน
 * C: แล้วเรื่องการมีส่วนร่วมของกระเป๋าเงินผู้ใช้ (user wallet) ล่ะ?
 * I: เราต้องมั่นใจว่าลูกค้ามีเงินทุนเพียงพอก่อนวางคำสั่งซื้อ เงินสำหรับคำสั่งที่ยังค้างอยู่ (pending order) ต้องถูกกันไว้ (withhold) จนกว่าคำสั่งจะเสร็จสมบูรณ์

### **ความต้องการที่ไม่ใช่ฟังก์ชันการทำงาน (Non-functional requirements)**
ขนาดที่ผู้สัมภาษณ์กล่าวถึงบ่งบอกว่าเรากำลังออกแบบตลาดหลักทรัพย์ขนาดเล็กถึงขนาดกลาง
เรายังต้องมั่นใจว่ามีความยืดหยุ่นเพียงพอที่จะรองรับสัญลักษณ์หุ้นและผู้ใช้ที่มากขึ้นในอนาคต

ความต้องการที่ไม่ใช่ฟังก์ชันการทำงานอื่น ๆ:
 * ความพร้อมใช้งาน (Availability) - อย่างน้อย 99.99% เวลาที่ระบบหยุดทำงาน (downtime) สามารถทำลายชื่อเสียงได้
 * ความทนทานต่อความล้มเหลว (Fault tolerance) - จำเป็นต้องมีกลไกความทนทานต่อความล้มเหลวและการกู้คืนที่รวดเร็ว เพื่อจำกัดผลกระทบของเหตุการณ์ในระบบ production
 * ความหน่วง (Latency) - round-trip latency ควรอยู่ในระดับมิลลิวินาที โดยเน้นที่ 99th percentile latency ที่สูงอย่างต่อเนื่องในระดับ 99th percentile ทำให้เกิดประสบการณ์ที่ไม่ดีสำหรับผู้ใช้บางส่วน
 * ความปลอดภัย (Security) - เราควรมีระบบจัดการบัญชี เพื่อการปฏิบัติตามกฎหมาย เราจำเป็นต้องรองรับ KYC เพื่อยืนยันตัวตนผู้ใช้ เรายังควรป้องกัน DDoS สำหรับทรัพยากรสาธารณะ

### **การประมาณการแบบคร่าว ๆ (Back-of-the-envelope estimation)**
 * 100 สัญลักษณ์หุ้น, 1 พันล้านคำสั่งต่อวัน
 * เวลาซื้อขายปกติคือ 09:30 ถึง 16:00 (6.5 ชั่วโมง)
 * QPS = 1 พันล้าน / 6.5 / 3600 = 43000
 * Peak QPS = 5*QPS = 215000
 * ปริมาณการซื้อขายจะสูงกว่าอย่างมีนัยสำคัญเมื่อตลาดเปิด

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

### **ความรู้พื้นฐานทางธุรกิจ 101 (Business Knowledge 101)**
มาพูดคุยกันถึงแนวคิดพื้นฐานที่เกี่ยวข้องกับตลาดหลักทรัพย์

โบรกเกอร์ (broker) เป็นตัวกลางระหว่างตลาดหลักทรัพย์และผู้ใช้ปลายทาง - เช่น Robinhood, Fidelity เป็นต้น

ลูกค้าสถาบัน (institutional clients) ซื้อขายในปริมาณมากโดยใช้ซอฟต์แวร์การซื้อขายเฉพาะทาง พวกเขาต้องการการดูแลเป็นพิเศษ
เช่น การแบ่งคำสั่ง (order splitting) เมื่อซื้อขายปริมาณมาก เพื่อหลีกเลี่ยงผลกระทบต่อตลาด

ประเภทของคำสั่งซื้อขาย:
 * Limit - ซื้อหรือขายที่ราคาคงที่ อาจไม่พบคู่จับคู่ในทันที หรืออาจถูกจับคู่เพียงบางส่วน
 * Market - ไม่ระบุราคา ถูกดำเนินการที่ราคาตลาดปัจจุบันในทันที

ราคา:
 * Bid - ราคาสูงสุดที่ผู้ซื้อยินดีจะซื้อหุ้น
 * Ask - ราคาต่ำสุดที่ผู้ขายยินดีจะขายหุ้น

ตลาดสหรัฐอเมริกามีการเสนอราคา 3 ระดับชั้น (tier) - L1, L2, L3

L1 market data ประกอบด้วยราคา bid/ask ที่ดีที่สุดและปริมาณ (quantity):

<div style="margin-left:3rem">
    <img src="./images/l1-price.png" alt="l1-price" width="500" />
</div>

L2 มีระดับราคาเพิ่มเติมมากขึ้น:

<div style="margin-left:3rem">
    <img src="./images/l2-price.png" alt="l2-price" width="500" />
</div>

L3 แสดงระดับราคาและปริมาณที่ต่อคิวอยู่ในแต่ละระดับ:

<div style="margin-left:3rem">
    <img src="./images/l3-price.png" alt="l3-price" width="500" />
</div>

แผนภูมิแท่งเทียน (candlestick) แสดงราคาเปิดและปิดของตลาด รวมถึงราคาสูงสุดและต่ำสุดในช่วงเวลาที่กำหนด:

<div style="margin-left:3rem">
    <img src="./images/candlestick.png" alt="candlestick" width="500" />
</div>

FIX เป็นโปรโตคอลสำหรับแลกเปลี่ยนข้อมูลธุรกรรมหลักทรัพย์ ถูกใช้โดยผู้ให้บริการส่วนใหญ่ ตัวอย่างธุรกรรมหลักทรัพย์:
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **การออกแบบระดับสูง (High-level design)**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

Trade flow:
 * ลูกค้าวางคำสั่งซื้อขายผ่านหน้าจอการซื้อขาย (trading interface)
 * โบรกเกอร์ส่งคำสั่งไปยังตลาดหลักทรัพย์
 * คำสั่งเข้าสู่ตลาดหลักทรัพย์ผ่าน client gateway ซึ่งตรวจสอบความถูกต้อง (validate), จำกัดอัตรา (rate limit), ยืนยันตัวตน (authenticate) เป็นต้น จากนั้นคำสั่งจะถูกส่งต่อไปยัง order manager
 * order manager ทำการตรวจสอบความเสี่ยง (risk check) ตามกฎที่กำหนดโดย risk manager
 * หลังผ่านการตรวจสอบความเสี่ยงแล้ว order manager จะตรวจสอบว่ามีเงินเพียงพอในกระเป๋าเงิน (wallet) สำหรับคำสั่งนั้นหรือไม่
 * คำสั่งถูกส่งไปยัง matching engine เมื่อพบคู่จับคู่ (match) matching engine จะสร้างการดำเนินการซื้อขาย (execution หรือเรียกว่า fill) สองรายการสำหรับฝั่งซื้อและขาย ทั้งสองคำสั่งจะถูกเรียงลำดับ (sequenced) เพื่อให้เป็นแบบ deterministic
 * การดำเนินการซื้อขายจะถูกส่งกลับไปยังลูกค้า

Market data flow (M1-M3):
 * matching engine สร้าง stream ของการดำเนินการซื้อขาย ซึ่งส่งไปยัง market data publisher
 * market data publisher สร้างแผนภูมิแท่งเทียนและส่งไปยัง data service
 * market data ถูกจัดเก็บใน storage เฉพาะทางสำหรับการวิเคราะห์แบบ real-time โบรกเกอร์เชื่อมต่อกับ data service เพื่อรับ market data ที่ทันเวลา

Reporter flow (R1-R2):
 * reporter รวบรวม field ที่จำเป็นสำหรับการรายงานจากคำสั่งและการดำเนินการซื้อขาย แล้วเขียนลงฐานข้อมูล
 * field สำหรับการรายงาน - client_id, price, quantity, order_type, filled_quantity, remaining_quantity

Trading flow อยู่บน critical path ในขณะที่ flow อื่น ๆ ไม่ได้อยู่บน critical path ดังนั้นความต้องการด้าน latency จึงแตกต่างกันระหว่างสองส่วนนี้

#### Trading flow
Trading flow อยู่บน critical path ดังนั้นจึงควรได้รับการปรับให้เหมาะสม (optimize) อย่างเต็มที่เพื่อ latency ที่ต่ำ

matching engine คือหัวใจสำคัญของมัน หรือเรียกอีกชื่อว่า cross engine ความรับผิดชอบหลัก:
 * ดูแลรักษา order book สำหรับแต่ละสัญลักษณ์หุ้น - รายการคำสั่งซื้อ/ขายสำหรับสัญลักษณ์หุ้นหนึ่งตัว
 * จับคู่คำสั่งซื้อและขาย - การจับคู่หนึ่งครั้งจะให้ผลลัพธ์เป็นการดำเนินการซื้อขาย (fill) สองรายการ หนึ่งสำหรับฝั่งซื้อและหนึ่งสำหรับฝั่งขาย ฟังก์ชันนี้ต้องรวดเร็วและแม่นยำ
 * กระจาย stream ของการดำเนินการซื้อขายในรูปแบบ market data
 * การจับคู่ต้องถูกสร้างขึ้นตามลำดับที่แน่นอน (deterministic) เป็นพื้นฐานสำคัญสำหรับความพร้อมใช้งานสูง (high availability)

ถัดมาคือ sequencer - เป็นคอมโพเนนต์หลักที่ทำให้ matching engine เป็นแบบ deterministic โดยการประทับ (stamp) sequence ID บนคำสั่งขาเข้าและ fill ขาออกแต่ละรายการ

<div style="margin-left:3rem">
    <img src="./images/sequencer.png" alt="sequencer" width="500" />
</div>

เราประทับ sequence ID บนคำสั่งขาเข้าและ fill ขาออกด้วยเหตุผลหลายประการ:
 * ความตรงเวลา (timeliness) และความเป็นธรรม (fairness)
 * การกู้คืน/replay ที่รวดเร็ว
 * การรับประกัน exactly-once

โดยแนวคิดแล้ว เราสามารถใช้ Kafka เป็น sequencer ของเราได้ เนื่องจากมันทำหน้าที่เป็น message queue ขาเข้าและขาออกได้อย่างมีประสิทธิภาพ อย่างไรก็ตาม เราจะ implement มันขึ้นเองเพื่อให้ได้ latency ที่ต่ำกว่า

order manager จัดการสถานะของคำสั่ง มันยังปฏิสัมพันธ์กับ matching engine - ส่งคำสั่งและรับ fill

ความรับผิดชอบของ order manager:
 * ส่งคำสั่งเพื่อตรวจสอบความเสี่ยง - เช่น ตรวจสอบว่าปริมาณการซื้อขายของผู้ใช้น้อยกว่า 1 ล้านหรือไม่
 * ตรวจสอบคำสั่งกับกระเป๋าเงินผู้ใช้ และยืนยันว่ามีเงินทุนเพียงพอในการดำเนินการ
 * ส่งคำสั่งไปยัง sequencer และต่อไปยัง matching engine เพื่อลดปริมาณ bandwidth มีเพียงข้อมูลคำสั่งที่จำเป็นเท่านั้นที่ถูกส่งไปยัง matching engine
 * รับการดำเนินการซื้อขาย (fill) กลับมาจาก sequencer ซึ่งจะถูกส่งไปยังโบรกเกอร์ผ่าน client gateway

ความท้าทายหลักในการ implement order manager คือการจัดการการเปลี่ยนสถานะ (state transition) Event sourcing เป็นหนึ่งในวิธีแก้ที่ใช้ได้ (จะกล่าวถึงในหัวข้อเจาะลึก)

สุดท้าย client gateway รับคำสั่งจากผู้ใช้และส่งไปยัง order manager ความรับผิดชอบของมัน:

<div style="margin-left:3rem">
    <img src="./images/client-gateway.png" alt="client-gateway" width="500" />
</div>

เนื่องจาก client gateway อยู่บน critical path มันจึงควรมีน้ำหนักเบา (lightweight)

สามารถมี client gateway หลายตัวสำหรับลูกค้าที่แตกต่างกัน เช่น colo engine เป็นเซิร์ฟเวอร์เครื่องมือการซื้อขายที่โบรกเกอร์เช่าไว้ใน data center ของตลาดหลักทรัพย์:

<div style="margin-left:3rem">
    <img src="./images/client-gateways.png" alt="client-gateways" width="500" />
</div>

#### Market data flow
market data publisher รับการดำเนินการซื้อขายจาก matching engine และสร้าง order book/แผนภูมิแท่งเทียนจาก stream ของการดำเนินการซื้อขายนั้น

ข้อมูลนั้นถูกส่งไปยัง data service ซึ่งรับผิดชอบในการแสดงข้อมูลที่รวบรวมแล้ว (aggregated data) ให้กับผู้สมัครสมาชิก (subscriber):

<div style="margin-left:3rem">
    <img src="./images/market-data.png" alt="market-data" width="500" />
</div>

#### Reporting flow
reporter ไม่ได้อยู่บน critical path แต่ก็ยังคงเป็นคอมโพเนนต์ที่สำคัญ

<div style="margin-left:3rem">
    <img src="./images/reporting-flow.png" alt="reporting-flow" width="500" />
</div>

มันรับผิดชอบเรื่องประวัติการซื้อขาย, การรายงานภาษี, การรายงานเพื่อการปฏิบัติตามกฎระเบียบ, การชำระราคา (settlement) เป็นต้น
latency ไม่ใช่ความต้องการที่สำคัญสำหรับ reporting flow แต่ความแม่นยำและการปฏิบัติตามกฎระเบียบสำคัญกว่า

### **การออกแบบ API (API Design)**
ลูกค้าปฏิสัมพันธ์กับตลาดหลักทรัพย์ผ่านโบรกเกอร์ เพื่อวางคำสั่ง, ดูการดำเนินการซื้อขาย, market data, ดาวน์โหลดข้อมูลในอดีตเพื่อการวิเคราะห์ เป็นต้น

เราใช้ RESTful API สำหรับการสื่อสารระหว่าง client gateway และโบรกเกอร์

สำหรับลูกค้าสถาบัน เราใช้โปรโตคอลเฉพาะ (proprietary protocol) เพื่อตอบสนองความต้องการด้าน latency ต่ำของพวกเขา

สร้างคำสั่งซื้อขาย:
```
POST /v1/order
```

พารามิเตอร์:
 * symbol - สัญลักษณ์หุ้น ชนิด String
 * side - ซื้อหรือขาย ชนิด String
 * price - ราคาของ limit order ชนิด Long
 * orderType - limit หรือ market (การออกแบบของเรารองรับเฉพาะ limit order เท่านั้น) ชนิด String
 * quantity - ปริมาณของคำสั่ง ชนิด Long

Response:
 * id - ID ของคำสั่ง ชนิด Long
 * creationTime - เวลาที่ระบบสร้างคำสั่งนี้ ชนิด Long
 * filledQuantity - ปริมาณที่ดำเนินการสำเร็จแล้ว ชนิด Long
 * remainingQuantity - ปริมาณที่ยังต้องดำเนินการ ชนิด Long
 * status - new/canceled/filled ชนิด String
 * attribute อื่น ๆ เหมือนกับพารามิเตอร์ที่รับเข้ามา

ดึงข้อมูลการดำเนินการซื้อขาย:
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

พารามิเตอร์:
 * symbol - สัญลักษณ์หุ้น ชนิด String
 * orderId - ID ของคำสั่ง ไม่บังคับ ชนิด String
 * startTime - เวลาเริ่มต้นของ query ในรูปแบบ epoch [11] ชนิด Long
 * endTime - เวลาสิ้นสุดของ query ในรูปแบบ epoch ชนิด Long

Response:
 * executions - array ของแต่ละ execution ในขอบเขตที่ระบุ (ดู attribute ด้านล่าง) ชนิด Array
 * id - ID ของ execution ชนิด Long
 * orderId - ID ของคำสั่ง ชนิด Long
 * symbol - สัญลักษณ์หุ้น ชนิด String
 * side - ซื้อหรือขาย ชนิด String
 * price - ราคาของ execution ชนิด Long
 * orderType - limit หรือ market ชนิด String
 * quantity - ปริมาณที่ดำเนินการสำเร็จ ชนิด Long

ดึงข้อมูล order book:
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

พารามิเตอร์:
 * symbol - สัญลักษณ์หุ้น ชนิด String
 * depth - ความลึกของ order book ต่อฝั่ง ชนิด Int

Response:
 * bids - array ของราคาและขนาด (size) ชนิด Array
 * asks - array ของราคาและขนาด (size) ชนิด Array

ดึงข้อมูลแท่งเทียน:
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

พารามิเตอร์:
 * symbol - สัญลักษณ์หุ้น ชนิด String
 * resolution - ความยาวของหน้าต่างเวลา (window length) ของแผนภูมิแท่งเทียนในหน่วยวินาที ชนิด Long
 * startTime - เวลาเริ่มต้นของหน้าต่างเวลาในรูปแบบ epoch ชนิด Long
 * endTime - เวลาสิ้นสุดของหน้าต่างเวลาในรูปแบบ epoch ชนิด Long

Response:
 * candles - array ของข้อมูลแท่งเทียนแต่ละแท่ง (attribute แสดงด้านล่าง) ชนิด Array
 * open - ราคาเปิดของแต่ละแท่งเทียน ชนิด Double
 * close - ราคาปิดของแต่ละแท่งเทียน ชนิด Double
 * high - ราคาสูงสุดของแต่ละแท่งเทียน ชนิด Double
 * low - ราคาต่ำสุดของแต่ละแท่งเทียน ชนิด Double

### **โมเดลข้อมูล (Data models)**
มีข้อมูลหลัก 3 ประเภทในตลาดหลักทรัพย์ของเรา:
 * Product, order, execution
 * order book
 * แผนภูมิแท่งเทียน (candlestick chart)

#### Product, order, execution
Product อธิบาย attribute ของสัญลักษณ์หุ้นที่ซื้อขาย - ประเภทของ product, สัญลักษณ์การซื้อขาย, สัญลักษณ์ที่แสดงใน UI เป็นต้น

ข้อมูลนี้ไม่เปลี่ยนแปลงบ่อยนัก มันถูกใช้หลักเพื่อการแสดงผลใน UI

order แทนคำสั่งซื้อ/ขาย ในขณะที่ execution คือผลลัพธ์การจับคู่ที่ส่งออก (outbound)

นี่คือโมเดลข้อมูล:

<div style="margin-left:3rem">
    <img src="./images/product-order-execution-data-model.png" alt="product-order-execution-data-model" width="500" />
</div>

เราพบ order และ execution ในทั้งสาม flow ของเรา:
 * ใน critical path ทั้งสองจะถูกประมวลผลใน memory เพื่อประสิทธิภาพสูง ทั้งสองถูกจัดเก็บและกู้คืนจาก sequencer
 * reporter เขียน order และ execution ลงฐานข้อมูลเพื่อวัตถุประสงค์ในการรายงาน
 * execution ถูกส่งต่อไปยัง market data เพื่อสร้าง order book และแผนภูมิแท่งเทียนขึ้นใหม่

#### Order book
order book คือรายการคำสั่งซื้อ/ขายสำหรับตราสาร (instrument) หนึ่งตัว จัดเรียงตามระดับราคา

โครงสร้างข้อมูลที่มีประสิทธิภาพสำหรับโมเดลนี้ต้องตอบสนอง:
 * เวลาในการค้นหาคงที่ (constant lookup time) - การหาปริมาณที่ระดับราคาใดราคาหนึ่ง หรือระหว่างระดับราคา
 * operation การเพิ่ม/ดำเนินการ/ยกเลิกที่รวดเร็ว
 * การ query ราคา bid/ask ที่ดีที่สุด
 * การวนซ้ำ (iterate) ผ่านระดับราคาต่าง ๆ

ตัวอย่างการดำเนินการของ order book:

<div style="margin-left:3rem">
    <img src="./images/order-book-execution.png" alt="order-book-execution" width="500" />
</div>

หลังจากดำเนินการคำสั่งขนาดใหญ่นี้เสร็จแล้ว ราคาจะเพิ่มขึ้นเนื่องจากส่วนต่าง (spread) ระหว่าง bid/ask กว้างขึ้น

ตัวอย่างการ implement order book ใน pseudo code:
```
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

สำหรับการ implement ที่มีประสิทธิภาพยิ่งขึ้น เราสามารถใช้ doubly-linked list แทน list มาตรฐาน:
 * การวางคำสั่งใหม่มี complexity O(1) เนื่องจากเราเพิ่มคำสั่งเข้าที่ท้าย (tail) ของ list
 * การจับคู่คำสั่งมี complexity O(1) เนื่องจากเราลบคำสั่งออกจากหัว (head)
 * การยกเลิกคำสั่งหมายถึงการลบคำสั่งออกจาก order book เราใช้ `orderMap` เพื่อการค้นหา O(1) และการลบ O(1) (เนื่องจาก `Order` มี reference ไปยัง element ก่อนหน้าใน list)

<div style="margin-left:3rem">
    <img src="./images/order-book-impl.png" alt="order-book-impl" width="500" />
</div>

โครงสร้างข้อมูลนี้ยังถูกใช้ใน market data service เพื่อสร้าง order book ขึ้นใหม่ด้วย

#### แผนภูมิแท่งเทียน (Candlestick chart)
ข้อมูลแท่งเทียนถูกคำนวณภายใน market data service โดยอิงจากการประมวลผลคำสั่งในช่วงเวลาหนึ่ง:
```
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

การปรับปรุงประสิทธิภาพบางส่วนเพื่อหลีกเลี่ยงการใช้ memory มากเกินไป:
 * ใช้ ring buffer ที่จัดสรร (pre-allocate) ไว้ล่วงหน้าเพื่อเก็บแท่งเทียน เพื่อลดจำนวนครั้งของการจัดสรร (allocation)
 * จำกัดจำนวนแท่งเทียนใน memory และเก็บที่เหลือลงดิสก์

เราจะใช้ในหน่วยความจำ columnar database (เช่น KDB) สำหรับการวิเคราะห์แบบ real-time หลังจากตลาดปิด ข้อมูลจะถูกจัดเก็บลงฐานข้อมูลในอดีต (historical database)

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)
สิ่งที่น่าสนใจข้อหนึ่งที่ควรรู้เกี่ยวกับตลาดหลักทรัพย์สมัยใหม่คือ ต่างจากซอฟต์แวร์ส่วนใหญ่ พวกมันมักรันทุกอย่างบนเซิร์ฟเวอร์ขนาดใหญ่เพียงเครื่องเดียว

มาสำรวจรายละเอียดกัน

### **ประสิทธิภาพ (Performance)**
สำหรับตลาดหลักทรัพย์ latency โดยรวมที่ดีในทุก percentile เป็นสิ่งสำคัญมาก

เราจะลด latency ได้อย่างไร?
 * ลดจำนวนงาน (task) บน critical path
 * ลดเวลาที่ใช้ในแต่ละงาน โดยการลดการใช้เครือข่าย/ดิสก์ และ/หรือลดเวลาในการดำเนินการของงาน

เพื่อบรรลุเป้าหมายแรก เราตัดความรับผิดชอบที่ไม่จำเป็นทั้งหมดออกจาก critical path แม้แต่การ logging ก็ถูกตัดออกเพื่อให้ได้ latency ที่เหมาะสมที่สุด

หากเราทำตามการออกแบบดั้งเดิม จะมี bottleneck หลายจุด - network latency ระหว่างบริการต่าง ๆ และการใช้ดิสก์ของ sequencer

ด้วยการออกแบบเช่นนี้ เราสามารถบรรลุ latency แบบ end-to-end ในระดับหลายสิบมิลลิวินาที แต่เราต้องการให้ได้ในระดับหลายสิบไมโครวินาทีแทน

ดังนั้น เราจะวางทุกอย่างไว้บนเซิร์ฟเวอร์เดียว และให้ process ต่าง ๆ สื่อสารกันผ่าน mmap ในฐานะ event store:

<div style="margin-left:3rem">
    <img src="./images/mmap-bus.png" alt="mmap-bus" width="500" />
</div>

การปรับปรุงอีกอย่างคือการใช้ application loop (while loop ที่รันงานสำคัญ) ซึ่งถูกตรึง (pin) ไว้กับ CPU ตัวเดียวกัน เพื่อหลีกเลี่ยงการ context switch:

<div style="margin-left:3rem">
    <img src="./images/application-loop.png" alt="application-loop" width="500" />
</div>

ผลข้างเคียงอีกอย่างของการใช้ application loop คือไม่มี lock contention - หลายเธรดแย่งชิงทรัพยากรเดียวกัน

ตอนนี้มาสำรวจกันว่า mmap ทำงานอย่างไร - มันเป็น UNIX syscall ซึ่ง map ไฟล์บนดิสก์เข้ากับ memory ของแอปพลิเคชัน

เคล็ดลับหนึ่งที่เราสามารถใช้ได้คือการสร้างไฟล์ใน `/dev/shm` ซึ่งย่อมาจาก "shared memory" ดังนั้นเราจะไม่มีการเข้าถึงดิสก์เลย

### **Event sourcing**
Event sourcing ถูกกล่าวถึงอย่างละเอียดใน[บทกระเป๋าเงินดิจิทัล](../chapter28) โปรดอ้างอิงบทนั้นสำหรับรายละเอียดทั้งหมด

โดยสรุปแล้ว แทนที่จะจัดเก็บสถานะปัจจุบัน เราจัดเก็บการเปลี่ยนสถานะที่ไม่เปลี่ยนแปลง (immutable state transition):

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

 * ด้านซ้าย - schema แบบดั้งเดิม
 * ด้านขวา - schema แบบ event source

นี่คือรูปแบบการออกแบบของเราจนถึงตอนนี้:

<div style="margin-left:3rem">
    <img src="./images/design-so-far.png" alt="design-so-far" width="500" />
</div>

 * โดเมนภายนอกปฏิสัมพันธ์กับ client gateway ของเราโดยใช้โปรโตคอล FIX
 * order manager รับ event คำสั่งใหม่ ตรวจสอบความถูกต้อง และเพิ่มเข้าไปในสถานะภายในของมัน คำสั่งจะถูกส่งไปยัง matching core
 * หากคำสั่งถูกจับคู่แล้ว `OrderFilledEvent` จะถูกสร้างขึ้นและส่งผ่าน mmap
 * คอมโพเนนต์อื่น ๆ สมัครสมาชิก (subscribe) event store และทำหน้าที่ของตัวเองในการประมวลผล

การปรับปรุงเพิ่มเติมอีกข้อ - คอมโพเนนต์ทั้งหมดเก็บสำเนาของ order manager ซึ่งถูกบรรจุ (package) เป็นไลบรารี เพื่อหลีกเลี่ยงการเรียก call เพิ่มเติมสำหรับการจัดการคำสั่ง

sequencer ในการออกแบบนี้ เปลี่ยนจากการเป็น event store ไปเป็นตัวเขียนเพียงตัวเดียว (single writer) ซึ่งเรียงลำดับ event ก่อนที่จะส่งต่อไปยัง event store:

<div style="margin-left:3rem">
    <img src="./images/sequencer-deep-dive.png" alt="sequencer-deep-dive" width="500" />
</div>

### **ความพร้อมใช้งานสูง (High availability)**
เรามุ่งเป้าไปที่ความพร้อมใช้งาน 99.99% - downtime เพียง 8.64 วินาทีต่อวันเท่านั้น

เพื่อบรรลุเป้าหมายนี้ เราต้องระบุจุดที่อาจเกิดความล้มเหลวได้ทั้งระบบ (single-point-of-failure) ในสถาปัตยกรรมของตลาดหลักทรัพย์:
 * ตั้งค่า instance สำรอง (backup) ของบริการสำคัญ (เช่น matching engine) ให้อยู่ในสถานะพร้อมใช้งาน (stand-by)
 * ทำให้การตรวจจับความล้มเหลวและการ failover ไปยัง instance สำรองเป็นแบบอัตโนมัติอย่างจริงจัง

บริการแบบ stateless เช่น client gateway สามารถขยาย (scale) แนวนอนได้ง่าย ๆ โดยการเพิ่มเซิร์ฟเวอร์

สำหรับคอมโพเนนต์แบบ stateful เราสามารถประมวลผล event ขาเข้าได้ แต่จะไม่เผยแพร่ (publish) event ขาออก หากเราไม่ใช่ leader:

<div style="margin-left:3rem">
    <img src="./images/leader-election.png" alt="leader-election" width="500" />
</div>

เพื่อตรวจจับว่า primary replica ล่มหรือไม่ เราสามารถส่ง heartbeat เพื่อตรวจจับว่ามันไม่ทำงานแล้ว

กลไกนี้ทำงานได้เฉพาะภายในขอบเขตของเซิร์ฟเวอร์เดียวเท่านั้น
หากเราต้องการขยายกลไกนี้ เราสามารถตั้งค่าให้ทั้งเซิร์ฟเวอร์เป็น hot/warm replica และ failover ในกรณีเกิดความล้มเหลว

เพื่อทำสำเนา event store ไปยัง replica ต่าง ๆ เราสามารถใช้ reliable UDP เพื่อการสื่อสารที่รวดเร็วกว่า

### **ความทนทานต่อความล้มเหลว (Fault tolerance)**
จะเกิดอะไรขึ้นหาก instance สำรอง (warm instance) ล่มไปด้วย? นี่เป็นเหตุการณ์ที่มีความน่าจะเป็นต่ำ แต่เราควรเตรียมพร้อมสำหรับมัน

บริษัทเทคโนโลยีขนาดใหญ่จัดการปัญหานี้ด้วยการทำสำเนาข้อมูลหลักไปยัง data center ในหลายเมือง เพื่อลดผลกระทบจากภัยพิบัติทางธรรมชาติ

คำถามที่ต้องพิจารณา:
 * หาก primary instance ล่ม เราจะ failover ไปยัง backup instance อย่างไรและเมื่อไร?
 * เราจะเลือก leader จาก backup instance ต่าง ๆ ได้อย่างไร?
 * เวลาในการกู้คืน (RTO - recovery time objective) ที่ต้องการคือเท่าไร?
 * ฟังก์ชันใดบ้างที่จำเป็นต้องกู้คืน? ระบบของเราสามารถทำงานภายใต้สภาวะที่ลดระดับลง (degraded conditions) ได้หรือไม่?

วิธีจัดการกับสิ่งเหล่านี้:
 * ระบบอาจล่มเนื่องจากบั๊ก (ซึ่งส่งผลต่อทั้ง primary และ replica) เราสามารถใช้ chaos engineering เพื่อค้นหากรณีขอบและผลลัพธ์ที่เลวร้ายเช่นนี้
 * ในเบื้องต้น เราอาจดำเนินการ failover ด้วยตนเองก่อน จนกว่าเราจะรวบรวมความรู้เพียงพอเกี่ยวกับรูปแบบความล้มเหลวของระบบ
 * leader-election สามารถถูกใช้ได้ (เช่น Raft) เพื่อกำหนดว่า replica ใดจะกลายเป็น leader ในกรณีที่ primary ล่ม

ตัวอย่างวิธีที่การทำ replication ทำงานข้ามเซิร์ฟเวอร์ต่าง ๆ:

<div style="margin-left:3rem">
    <img src="./images/replication-across-servers.png" alt="replication-across-servers" width="500" />
</div>

ตัวอย่างศัพท์เกี่ยวกับ leader-election:

<div style="margin-left:3rem">
    <img src="./images/leader-election-terms.png" alt="leader-election-terms" width="500" />
</div>

สำหรับรายละเอียดว่า Raft ทำงานอย่างไร [ดูที่นี่](https://thesecretlivesofdata.com/raft/)

สุดท้าย เรายังต้องพิจารณาความทนทานต่อการสูญเสียข้อมูล (loss tolerance) - เราสามารถสูญเสียข้อมูลได้มากเพียงใดก่อนที่จะเกิดปัญหาร้ายแรง?
สิ่งนี้จะเป็นตัวกำหนดว่าเราควร backup ข้อมูลบ่อยแค่ไหน

สำหรับตลาดหลักทรัพย์ การสูญเสียข้อมูลนั้นไม่สามารถยอมรับได้ ดังนั้นเราต้อง backup ข้อมูลบ่อยครั้ง และพึ่งพา replication ของ raft เพื่อลดความน่าจะเป็นของการสูญเสียข้อมูล

### **algorithm การจับคู่ (Matching algorithms)**
ขอออกนอกเรื่องเล็กน้อยเพื่ออธิบายว่าการจับคู่ทำงานอย่างไรผ่าน pseudo code:
```
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

algorithm การจับคู่นี้ใช้ algorithm แบบ FIFO ในการกำหนดว่าคำสั่งใดที่ระดับราคาหนึ่ง ๆ ควรถูกจับคู่ก่อน

### **ความเป็น deterministic (Determinism)**
determinism เชิงฟังก์ชัน (functional determinism) ถูกรับประกันผ่านเทคนิค sequencer ที่เราใช้

เวลาจริงที่ event เกิดขึ้นนั้นไม่สำคัญ:

<div style="margin-left:3rem">
    <img src="./images/determinism.png" alt="determinism" width="500" />
</div>

determinism ของ latency เป็นสิ่งที่เราต้องติดตาม เราสามารถคำนวณได้โดยการติดตาม latency ที่ 99 หรือ 99.99 percentile

สิ่งที่ทำให้เกิด latency spike ได้ เช่น เหตุการณ์ garbage collector ในภาษาอย่าง Java

### **การปรับปรุงประสิทธิภาพของ market data publisher (Market data publisher optimizations)**
market data publisher รับผลลัพธ์การจับคู่จาก matching engine และสร้าง order book และแผนภูมิแท่งเทียนขึ้นใหม่จากผลลัพธ์เหล่านั้น

เราเก็บแท่งเทียนเพียงบางส่วนเท่านั้น เนื่องจากเราไม่มี memory ไม่จำกัด ลูกค้าสามารถเลือกได้ว่าต้องการข้อมูลที่ละเอียด (granular) แค่ไหน ข้อมูลที่ละเอียดมากขึ้นอาจต้องเสียราคาที่สูงขึ้น:

<div style="margin-left:3rem">
    <img src="./images/market-data-publisher.png" alt="market-data-publisher" width="500" />
</div>

ring buffer (หรือเรียกว่า circular buffer) เป็นคิวขนาดคงที่ที่หัวเชื่อมต่อกับท้าย พื้นที่ถูกจัดสรรไว้ล่วงหน้าเพื่อหลีกเลี่ยงการจัดสรร (allocation) เพิ่มเติม โครงสร้างข้อมูลนี้ยังเป็นแบบ lock-free ด้วย

อีกเทคนิคหนึ่งในการปรับปรุง ring buffer คือการทำ padding ซึ่งรับประกันว่า sequence number จะไม่อยู่ใน cache line เดียวกันกับสิ่งอื่นใด

### **ความเป็นธรรมในการกระจาย market data และ multicast (Distribution fairness of market data and multicast)**
เราต้องมั่นใจว่าผู้สมัครสมาชิก (subscriber) ได้รับข้อมูลในเวลาเดียวกัน เนื่องจากหากคนหนึ่งได้รับข้อมูลก่อนอีกคนหนึ่ง จะทำให้คนนั้นได้เปรียบด้านข้อมูลเชิงลึกของตลาดที่สำคัญ ซึ่งสามารถนำไปใช้ควบคุมตลาดได้

เพื่อบรรลุสิ่งนี้ เราสามารถใช้ multicast ผ่าน reliable UDP เมื่อเผยแพร่ข้อมูลไปยังผู้สมัครสมาชิก

ข้อมูลสามารถถูกส่งผ่านอินเทอร์เน็ตได้สามวิธี:
 * Unicast - หนึ่งแหล่งที่มา หนึ่งปลายทาง
 * Broadcast - หนึ่งแหล่งที่มาไปยังทั้ง subnetwork
 * Multicast - หนึ่งแหล่งที่มาไปยังกลุ่มโฮสต์ในหลาย subnetwork ที่แตกต่างกัน

ในทางทฤษฎี การใช้ multicast จะทำให้ผู้สมัครสมาชิกทุกคนได้รับข้อมูลในเวลาเดียวกัน

อย่างไรก็ตาม UDP นั้นไม่น่าเชื่อถือ (unreliable) และข้อมูลอาจไม่ถึงทุกคน แต่สามารถปรับปรุงได้ด้วยการส่งซ้ำ (retransmission)

### **การอยู่ร่วมสถานที่ (Colocation)**
ตลาดหลักทรัพย์เสนอความสามารถให้โบรกเกอร์วางเซิร์ฟเวอร์ของตัวเองไว้ใน data center เดียวกันกับตลาดหลักทรัพย์

วิธีนี้ลด latency ได้อย่างมาก และถือได้ว่าเป็นบริการระดับ VIP

### **ความปลอดภัยของเครือข่าย (Network Security)**
DDoS เป็นความท้าทายสำหรับตลาดหลักทรัพย์ เนื่องจากมีบริการบางส่วนที่เปิดสู่อินเทอร์เน็ต นี่คือตัวเลือกของเรา:
 * แยกบริการและข้อมูลสาธารณะออกจากบริการส่วนตัว เพื่อไม่ให้การโจมตี DDoS ส่งผลกระทบต่อลูกค้าที่สำคัญที่สุด
 * ใช้ชั้นแคช (caching layer) เพื่อจัดเก็บข้อมูลที่ไม่ค่อยมีการอัปเดตบ่อยนัก
 * ทำให้ URL แข็งแกร่งขึ้นต่อการโจมตี DDoS เช่น เลือกใช้ `https://my.website.com/data/recent` แทน `https://my.website.com/data?from=123&to=456` เนื่องจากแบบแรกสามารถแคชได้ง่ายกว่า
 * จำเป็นต้องมีกลไก allowlist/blocklist ที่มีประสิทธิภาพ
 * สามารถใช้ rate limiting เพื่อลดผลกระทบจาก DDoS

---

## ขั้นตอนที่ 4: สรุป (Wrap Up)
บันทึกที่น่าสนใจอื่น ๆ:
 * ไม่ใช่ตลาดหลักทรัพย์ทุกแห่งที่พึ่งพาการวางทุกอย่างไว้บนเซิร์ฟเวอร์ขนาดใหญ่เพียงเครื่องเดียว แต่บางแห่งก็ยังคงทำเช่นนั้น
 * ตลาดหลักทรัพย์สมัยใหม่พึ่งพาโครงสร้างพื้นฐานแบบ cloud มากขึ้น รวมถึงพึ่งพา automatic market makers (AMM) เพื่อหลีกเลี่ยงการดูแลรักษา order book
