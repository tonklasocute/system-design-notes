# บทที่ 5: การออกแบบ Consistent Hashing (Design Consistent Hashing)

## บทนำ
บทนี้จะสำรวจ **Consistent Hashing** ซึ่งเป็นเทคนิคที่จำเป็นสำหรับการทำ horizontal scaling อย่างมีประสิทธิภาพ ด้วยการกระจาย request และข้อมูลไปยังเซิร์ฟเวอร์ต่าง ๆ เทคนิคนี้ช่วยลดการกระจายข้อมูลใหม่ (data redistribution) เมื่อมีการเพิ่มหรือลบเซิร์ฟเวอร์ และช่วยให้ข้อมูลถูกกระจายอย่างสม่ำเสมอ เพื่อลดปัญหาอย่าง server hotspot

## ปัญหาของการ Rehashing (The Rehashing Problem)
### คำอธิบาย
ในวิธี hashing แบบดั้งเดิม เช่น `serverIndex = hash(key) % N` การกระจายข้อมูลใหม่จะกลายเป็นปัญหาเมื่อจำนวนเซิร์ฟเวอร์เปลี่ยนแปลง ตัวอย่างเช่น:
- การเอาเซิร์ฟเวอร์ออกทำให้ key ส่วนใหญ่ต้องถูกกำหนดใหม่ (reassign) ส่งผลให้เกิด cache miss
- การเพิ่มเซิร์ฟเวอร์ทำให้เกิดการกระจาย key ใหม่โดยไม่จำเป็น

  <img src="./images/server-hashing.png"  alt="Server hashing" width="450">

- วิธีนี้ใช้ได้ดีเมื่อขนาดของ server pool คงที่ แต่จะเกิดปัญหาเมื่อมีการเพิ่มเซิร์ฟเวอร์ใหม่ หรือมีการเอาเซิร์ฟเวอร์ที่มีอยู่ออก

  <img src="./images/server-hashing-miss.png"  alt="Server hashing Miss" width="450">

### ปัญหาสำคัญ (Key Issue)
การกระจาย key ส่วนใหญ่ใหม่เมื่อจำนวนเซิร์ฟเวอร์เปลี่ยนแปลง ทำให้เกิดความไม่มีประสิทธิภาพและภาระงานล้น (overload)

## Consistent Hashing
### นิยาม (Definition)
Consistent Hashing ทำให้มั่นใจได้ว่า เมื่อมีการเพิ่มหรือลบเซิร์ฟเวอร์ จะมีเพียงเศษส่วนหนึ่งของ key เท่านั้นที่ต้องถูก remap ซึ่งช่วยลดการรบกวนระบบ (disruption) และเพิ่มความสามารถในการขยายระบบ (scalability)

### แนวคิดหลัก (Key Concepts)
1. **Hash Space และ Ring:** hash space จะก่อตัวเป็นวง (ring) ต่อเนื่อง โดยค่า hash จะถูกกระจายตั้งแต่ `0` ถึง `2^160-1` (เช่น ใช้ hash function อย่าง SHA-1) เมื่อเชื่อมปลายทั้งสองด้านเข้าด้วยกัน จะได้ ring
    <p align="center">
    <img src="./images/hash-ring.png"  alt="Hash Ring" width="450">
    </p>

- ใช้ hash function เดียวกัน f เพื่อ map เซิร์ฟเวอร์ (โดยอิงจาก server IP หรือชื่อ) ลงบน ring

    <p align="center">
    <img src="./images/server-ring.png"  alt="Server Ring" width="450">
    </p>

1. **การค้นหาเซิร์ฟเวอร์ (Server Lookup)**
- เซิร์ฟเวอร์ของ key หนึ่ง ๆ จะถูกกำหนดโดยการไล่ตามเข็มนาฬิกา (clockwise) บน ring จนกว่าจะพบเซิร์ฟเวอร์

  <p align="center">
  <img src="./images/server-lookup.png"  alt="Server Lookup" width="450">
  </p>

2. **การเพิ่มและลบเซิร์ฟเวอร์ (Adding and Removing Servers)**
- การเพิ่มเซิร์ฟเวอร์จะกระจายข้อมูลใหม่เฉพาะ key ที่อยู่ใกล้เคียงเท่านั้น มีเพียงเศษส่วนหนึ่งของ key ที่ถูกกระจายไปยังเซิร์ฟเวอร์ใหม่

  <p align="center">
  <img src="./images/adding-server.png"  alt="Adding Server" width="450">
  </p>

- การลบเซิร์ฟเวอร์ส่งผลกระทบเฉพาะ key ที่อยู่ในช่วงของเซิร์ฟเวอร์นั้นเท่านั้น มีเพียง key จากเซิร์ฟเวอร์ที่ถูกลบเท่านั้นที่จะถูกกำหนดใหม่ไปยังเซิร์ฟเวอร์ตัวถัดไปตามเข็มนาฬิกา

  <p align="center">
  <img src="./images/removing-server.png"  alt="Removing Server" width="450">
  </p>

## ความท้าทายและวิธีแก้ไข (Challenges and Solutions)
### ปัญหาสองประการในวิธีการพื้นฐาน (Two Issues in Basic Approach)
1. **ขนาด Partition ที่ไม่เท่ากัน (Uneven Partition Sizes):** เซิร์ฟเวอร์อาจมี partition ของข้อมูลไม่เท่ากัน
2. **การกระจาย Key ที่ไม่สม่ำเสมอ (Non-uniform Key Distribution):** เซิร์ฟเวอร์บางตัวอาจได้รับ key มากกว่าตัวอื่นอย่างมีนัยสำคัญ

### วิธีแก้ไข: Virtual Nodes
- เซิร์ฟเวอร์แต่ละตัวจะถูกแทนด้วย virtual node หลายตัวบน ring ซึ่งกระจายอย่างสม่ำเสมอบน ring
- Virtual node ช่วยปรับปรุงการกระจาย key และปรับสมดุลภาระงาน (load) เมื่อจำนวน virtual node เพิ่มขึ้น การกระจายของ key จะสมดุลมากขึ้น เนื่องจากค่าเบี่ยงเบนมาตรฐาน (standard deviation) จะเล็กลงเมื่อมี virtual node มากขึ้น ส่งผลให้การกระจายข้อมูลสมดุล

  <p align="center">
  <img src="./images/virtual-nodes.png"   alt="Virtual Nodes" width="450">
  </p>

## Key ที่ได้รับผลกระทบ (Affected Keys)
เมื่อมีการเพิ่มหรือลบเซิร์ฟเวอร์:
- **เซิร์ฟเวอร์ที่ถูกเพิ่ม (Added Server):** key ที่ได้รับผลกระทบคือ key ที่อยู่ระหว่างเซิร์ฟเวอร์ใหม่กับเซิร์ฟเวอร์ก่อนหน้า (predecessor)
  ในตัวอย่างต่อไปนี้ เซิร์ฟเวอร์ 4 ถูกเพิ่มเข้ามาบน ring ช่วงที่ได้รับผลกระทบจะเริ่มจาก s4 (node
  ที่เพิ่มใหม่) และเคลื่อนที่ทวนเข็มนาฬิกา (anticlockwise) รอบ ring จนกว่าจะพบเซิร์ฟเวอร์ (s3) ดังนั้น key
  ที่อยู่ระหว่าง s3 และ s4 จะต้องถูกกระจายใหม่ไปยัง s4

  <p align="center">
  <img src="./images/server-addition.png"   alt="Server Addition" width="450">
  </p>

- **เซิร์ฟเวอร์ที่ถูกลบ (Removed Server):** key ที่ได้รับผลกระทบคือ key ที่อยู่ระหว่างเซิร์ฟเวอร์ที่ถูกลบกับเซิร์ฟเวอร์ก่อนหน้า ในตัวอย่างต่อไปนี้ เมื่อเซิร์ฟเวอร์ (s1) ถูกลบ ช่วงที่ได้รับผลกระทบจะเริ่มจาก s1
(node ที่ถูกลบ) และเคลื่อนที่ทวนเข็มนาฬิกา รอบ ring จนกว่าจะพบเซิร์ฟเวอร์ (s0) ดังนั้น key ที่อยู่ระหว่าง s0 และ s1 จะต้องถูกกระจายใหม่ไปยัง s2

  <p align="center">
  <img src="./images/server-removed.png"   alt="Server Removed" width="450">
  </p>

## ประโยชน์ของ Consistent Hashing (Benefits of Consistent Hashing)
- **ลดการกระจายข้อมูลใหม่ให้น้อยที่สุด (Minimized Redistribution):** มีเพียงเศษส่วนหนึ่งของ key เท่านั้นที่ถูกกำหนดใหม่
- **ความสามารถในการขยายระบบ (Scalability):** รองรับ horizontal scaling ได้
- **ลดปัญหา Hotspot (Mitigates Hotspots):** ปรับสมดุลการกระจายข้อมูลเพื่อหลีกเลี่ยงภาระงานล้นของเซิร์ฟเวอร์

## การใช้งานจริง (Real-World Applications)
- Amazon Dynamo DB
- Apache Cassandra
- Discord
- Akamai CDN
- Maglev Load Balancer
