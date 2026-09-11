# บทที่ 25: กระดานคะแนนเกมแบบเรียลไทม์ (Real-time Gaming Leaderboard)

## บทนำ

เราจะออกแบบ **กระดานคะแนน (leaderboard)** สำหรับเกมมือถือแบบออนไลน์:

<div style="margin-left:3rem">
    <img src="./images/leaderboard.png" alt="leaderboard" width="500" />
</div>

---

## ขั้นตอนที่ 1: ทำความเข้าใจปัญหาและกำหนดขอบเขตการออกแบบ (Understand the Problem and Establish Design Scope)

- C: คะแนนสำหรับ leaderboard คำนวณอย่างไร?
- I: ผู้ใช้จะได้รับ 1 คะแนนทุกครั้งที่ชนะการแข่งขัน
- C: ผู้เล่นทุกคนถูกรวมอยู่ใน leaderboard หรือไม่?
- I: ใช่
- C: มีการแบ่งช่วงเวลา (time segment) ที่เกี่ยวข้องกับ leaderboard หรือไม่?
- I: ทุกเดือนจะมีการแข่งขัน (tournament) ใหม่เริ่มขึ้น ซึ่งจะเริ่ม leaderboard ใหม่ด้วย
- C: เราสามารถสมมติว่าเราสนใจแค่ 10 อันดับแรก (top 10) ได้หรือไม่?
- I: เราต้องการแสดงผู้ใช้ 10 อันดับแรก พร้อมทั้งตำแหน่ง (position) ของผู้ใช้คนใดคนหนึ่งด้วย หากมีเวลาเหลือ เราสามารถพูดคุยเรื่องการแสดงผู้ใช้ที่อยู่รอบ ๆ ตำแหน่งของผู้ใช้คนนั้นใน leaderboard ได้
- C: มีผู้ใช้กี่คนในหนึ่ง tournament?
- I: 5 ล้าน DAU และ 25 ล้าน MAU
- C: โดยเฉลี่ยมีการแข่งขัน (match) กี่ครั้งในระหว่างหนึ่ง tournament?
- I: ผู้เล่นแต่ละคนเล่นเฉลี่ย 10 แมตช์ต่อวัน
- C: เราจะกำหนดอันดับ (rank) อย่างไรหากผู้เล่นสองคนมีคะแนนเท่ากัน?
- I: ในกรณีนั้น อันดับของทั้งคู่จะเท่ากัน หากมีเวลาเหลือ เราสามารถพูดคุยเรื่องการตัดสินเมื่อคะแนนเสมอกัน (breaking ties) ได้
- C: leaderboard จำเป็นต้องเป็นแบบ real-time หรือไม่?
- I: ใช่ เราต้องการแสดงผลลัพธ์แบบ real-time หรือใกล้เคียง real-time ที่สุดเท่าที่จะทำได้ การแสดงผลลัพธ์ประวัติแบบเป็นชุด (batched result history) นั้นไม่เป็นที่ยอมรับ

### **ความต้องการด้านฟังก์ชันการทำงาน (Functional requirements)**

- แสดง 10 อันดับผู้เล่นแรกใน leaderboard
- แสดงอันดับเฉพาะของผู้ใช้คนหนึ่ง ๆ
- แสดงผู้ใช้ที่อยู่สูงกว่าและต่ำกว่าผู้ใช้ที่กำหนด 4 อันดับ (bonus)

### **ความต้องการที่ไม่ใช่ฟังก์ชันการทำงาน (Non-functional requirements)**

- การอัปเดตคะแนนแบบ real-time
- การอัปเดตคะแนนต้องสะท้อนบน leaderboard แบบ real-time
- ความสามารถในการขยาย (scalability), ความพร้อมใช้งาน (availability), ความน่าเชื่อถือ (reliability) โดยทั่วไป

### **การประมาณการแบบคร่าว ๆ (Back-of-the-envelope estimation)**

ด้วย 50 ล้าน DAU หากเกมมีการกระจายตัวของผู้เล่นอย่างสม่ำเสมอตลอด 24 ชั่วโมง เราจะมีผู้ใช้เฉลี่ย 50 คนต่อวินาที
อย่างไรก็ตาม เนื่องจากการกระจายตัวมักไม่สม่ำเสมอ เราสามารถประมาณได้ว่าจำนวนผู้ใช้ออนไลน์ที่ peak จะอยู่ที่ 250 คนต่อวินาที

QPS สำหรับผู้ใช้ที่ทำคะแนน - จากค่าเฉลี่ย 10 เกมต่อวัน, 50 ผู้ใช้/วินาที * 10 = 500 QPS โดย peak QPS = 2500

QPS สำหรับการดึงข้อมูล leaderboard 10 อันดับแรก - สมมติว่าผู้ใช้เปิดดูโดยเฉลี่ยวันละครั้ง QPS จะอยู่ที่ 50

---

## ขั้นตอนที่ 2: นำเสนอการออกแบบระดับสูงและขอความเห็นชอบ (Propose High-Level Design and Get Buy-In)

### **การออกแบบ API (API Design)**

API แรกที่เราต้องการคือ API สำหรับอัปเดตคะแนนของผู้ใช้:

```
POST /v1/scores
```

API นี้รับพารามิเตอร์สองตัว - `user_id` และ `points` ที่ได้จากการชนะเกม

API นี้ควรเข้าถึงได้เฉพาะจาก game server เท่านั้น ไม่ใช่จากไคลเอนต์ปลายทาง

ถัดไปคือ API สำหรับดึงข้อมูลผู้เล่น 10 อันดับแรกของ leaderboard:

```
GET /v1/scores
```

ตัวอย่าง response:

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

นอกจากนี้เรายังสามารถดึงคะแนนของผู้ใช้คนใดคนหนึ่งได้ด้วย:

```
GET /v1/scores/{:user_id}
```

ตัวอย่าง response:

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **สถาปัตยกรรมระดับสูง (High-level architecture)**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- เมื่อผู้เล่นชนะเกม ไคลเอนต์จะส่ง request ไปยัง game service
- game service ตรวจสอบว่าการชนะนั้นถูกต้องหรือไม่ และเรียก leaderboard service เพื่ออัปเดตคะแนนของผู้เล่น
- leaderboard service อัปเดตคะแนนของผู้ใช้ใน leaderboard store
- ผู้เล่นเรียก leaderboard service เพื่อดึงข้อมูล leaderboard เช่น ผู้เล่น 10 อันดับแรกและอันดับของผู้เล่นคนนั้น

การออกแบบทางเลือกอีกแบบหนึ่งที่ถูกพิจารณาคือ ให้ไคลเอนต์อัปเดตคะแนนของตัวเองโดยตรงภายใน leaderboard service:

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="alternative-design" width="500" />
</div>

ทางเลือกนี้ไม่ปลอดภัย เนื่องจากมีความเสี่ยงต่อการโจมตีแบบ man-in-the-middle ผู้เล่นสามารถตั้ง proxy และเปลี่ยนคะแนนของตัวเองได้ตามใจชอบ

ข้อควรระวังเพิ่มเติมอีกข้อหนึ่งคือ สำหรับเกมที่ตรรกะของเกม (game logic) ถูกจัดการโดยเซิร์ฟเวอร์ ไคลเอนต์ไม่จำเป็นต้องเรียกเซิร์ฟเวอร์อย่างชัดเจนเพื่อบันทึกการชนะของตัวเอง
เซิร์ฟเวอร์จะทำให้โดยอัตโนมัติตามตรรกะของเกม

ข้อพิจารณาเพิ่มเติมอีกอย่างคือ เราควรใส่ message queue ระหว่าง game server และ leaderboard service หรือไม่ วิธีนี้จะมีประโยชน์หากมีบริการอื่น ๆ ที่สนใจผลลัพธ์ของเกม แต่นั่นไม่ใช่ความต้องการที่ชัดเจนในการสัมภาษณ์นี้ ดังนั้นจึงไม่ถูกรวมอยู่ในการออกแบบ:

<div style="margin-left:3rem">
    <img src="./images/message-queue-based-comm.png" alt="message-queue-based-comm" width="500" />
</div>

### **โมเดลข้อมูล (Data models)**

มาพูดคุยกันถึงตัวเลือกที่เรามีสำหรับการจัดเก็บข้อมูล leaderboard - relational database, Redis, NoSQL

โซลูชัน NoSQL จะถูกกล่าวถึงในหัวข้อเจาะลึกการออกแบบ

#### โซลูชันฐานข้อมูลเชิงสัมพันธ์ (Relational database solution)

หากขนาดของระบบไม่ใช่ประเด็นสำคัญและเราไม่มีผู้ใช้มากนัก ฐานข้อมูลเชิงสัมพันธ์ก็ตอบโจทย์ได้ดีพอสมควร

เราสามารถเริ่มจากตาราง leaderboard แบบง่าย ๆ หนึ่งตารางต่อหนึ่งเดือน (หมายเหตุส่วนตัวของผู้เขียน - แนวคิดนี้ไม่ค่อยสมเหตุสมผล คุณสามารถเพิ่มคอลัมน์ `month` เพื่อหลีกเลี่ยงความยุ่งยากในการดูแลรักษาตารางใหม่ทุกเดือนได้):

<div style="margin-left:3rem">
    <img src="./images/leaderboard-table.png" alt="leaderboard-table" width="500" />
</div>

ยังมีข้อมูลเพิ่มเติมที่ควรใส่ในตารางนี้ แต่ไม่เกี่ยวข้องกับ query ที่เราจะรัน จึงถูกละไว้

จะเกิดอะไรขึ้นเมื่อผู้ใช้ทำคะแนนได้?

<div style="margin-left:3rem">
    <img src="./images/user-wins-point.png" alt="user-wins-point" width="500" />
</div>

หากผู้ใช้ยังไม่มีอยู่ในตาราง เราต้องแทรก (insert) ผู้ใช้นั้นก่อน:

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

ในการเรียกครั้งถัดไป เราแค่อัปเดตคะแนนของผู้ใช้นั้น:

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

เราจะหาผู้เล่นอันดับต้น ๆ ของ leaderboard ได้อย่างไร?

<div style="margin-left:3rem">
    <img src="./images/find-leaderboard-position.png" alt="find-leaderboard-position" width="500" />
</div>

เราสามารถรัน query นี้ได้:

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

อย่างไรก็ตาม query นี้ไม่มีประสิทธิภาพ เนื่องจากมันต้องสแกนตารางทั้งหมด (table scan) เพื่อเรียงลำดับ record ทั้งหมดในตาราง

เราสามารถปรับปรุงให้ดีขึ้นได้ด้วยการเพิ่ม index บนคอลัมน์ `score` และใช้ operation `LIMIT` เพื่อหลีกเลี่ยงการสแกนทั้งหมด:

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

อย่างไรก็ตาม วิธีนี้ไม่สามารถขยาย (scale) ได้ดี หากผู้ใช้ไม่ได้อยู่ในอันดับต้น ๆ ของ leaderboard และเราต้องการหาอันดับของผู้ใช้นั้น

#### โซลูชัน Redis (Redis solution)

เราต้องการหาโซลูชันที่ทำงานได้ดีแม้กับผู้เล่นนับล้านคน โดยไม่ต้องพึ่งพา query ฐานข้อมูลที่ซับซ้อน

Redis เป็น in-memory data store ซึ่งเร็วเนื่องจากทำงานใน memory และมีโครงสร้างข้อมูลที่เหมาะกับความต้องการของเรา - sorted set

sorted set เป็นโครงสร้างข้อมูลที่คล้ายกับ set ในภาษาโปรแกรมมิ่งทั่วไป ซึ่งช่วยให้เราเก็บโครงสร้างข้อมูลแบบเรียงลำดับตามเกณฑ์ที่กำหนดได้
ภายในถูก implement โดยใช้ hash-map เพื่อรักษาความสัมพันธ์ระหว่าง key (user_id) และ value (score) และ skip list ซึ่ง map คะแนนไปยังผู้ใช้ในลำดับที่เรียงแล้ว:

<div style="margin-left:3rem">
    <img src="./images/sorted-set.png" alt="sorted-set" width="500" />
</div>

skip list ทำงานอย่างไร?
- มันเป็น linked list ที่รองรับการค้นหาแบบรวดเร็ว
- ประกอบด้วย linked list ที่เรียงลำดับแล้วและ index หลายชั้น (multi-level index)

<div style="margin-left:3rem">
    <img src="./images/skip-list.png" alt="skip-list" width="500" />
</div>

โครงสร้างนี้ช่วยให้เราสามารถค้นหาค่าที่ต้องการได้อย่างรวดเร็ว เมื่อชุดข้อมูลมีขนาดใหญ่มากพอ
ในตัวอย่างด้านล่าง (64 node) การค้นหาค่าที่กำหนดต้องเดินผ่าน 62 node ใน linked list พื้นฐาน แต่ใน skip list ต้องเดินผ่านเพียง 11 node เท่านั้น:

<div style="margin-left:3rem">
    <img src="./images/skip-list-performance.png" alt="skip-list-performance" width="500" />
</div>

sorted set มีประสิทธิภาพมากกว่าฐานข้อมูลเชิงสัมพันธ์ เนื่องจากข้อมูลถูกเรียงลำดับไว้ตลอดเวลา โดยแลกกับ operation การเพิ่มและการค้นหาที่มี time complexity O(logN)

ในทางตรงกันข้าม นี่คือตัวอย่าง nested query ที่เราต้องรันเพื่อหาอันดับของผู้ใช้คนหนึ่งในฐานข้อมูลเชิงสัมพันธ์:

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

operation ใดบ้างที่เราต้องใช้เพื่อดำเนินการ leaderboard ของเราใน Redis?
- **ZADD** - แทรกผู้ใช้เข้าไปใน set หากยังไม่มีอยู่ มิฉะนั้นจะอัปเดตคะแนน มี time complexity O(logN)
- **ZINCRBY** - เพิ่มคะแนนของผู้ใช้ตามจำนวนที่กำหนด หากผู้ใช้ยังไม่มีอยู่ คะแนนจะเริ่มต้นที่ศูนย์ มี time complexity O(logN)
- **ZRANGE/ZREVRANGE** - ดึงข้อมูลผู้ใช้ตามช่วง (range) ที่เรียงลำดับตามคะแนน เราสามารถระบุลำดับ (ASC/DESC), offset และขนาดผลลัพธ์ได้ มี time complexity O(logN+M) โดย M คือขนาดผลลัพธ์
- **ZRANK/ZREVRANK** - ดึงตำแหน่ง (rank) ของผู้ใช้ที่กำหนดในลำดับ ASC/DESC มี time complexity O(logN)

จะเกิดอะไรขึ้นเมื่อผู้ใช้ทำคะแนนได้?

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

จะมี leaderboard ใหม่ถูกสร้างขึ้นทุกเดือน ในขณะที่ leaderboard เก่าจะถูกย้ายไปเก็บใน historical storage

จะเกิดอะไรขึ้นเมื่อผู้ใช้ดึงข้อมูลผู้เล่น 10 อันดับแรก?

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

ตัวอย่างผลลัพธ์:

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

แล้วผู้ใช้ที่ดึงข้อมูลตำแหน่งของตัวเองใน leaderboard ล่ะ?

<div style="margin-left:3rem">
    <img src="./images/leaderboard-position-of-user.png" alt="leaderboard-position-of-user" width="500" />
</div>

สามารถทำได้ง่าย ๆ ด้วย query นี้ หากเราทราบตำแหน่งของผู้ใช้ใน leaderboard อยู่แล้ว:

```
ZREVRANGE leaderboard_feb_2021 357 365
```

ตำแหน่งของผู้ใช้สามารถดึงได้ด้วย `ZREVRANK <user-id>`

มาสำรวจกันว่าความต้องการด้านการจัดเก็บของเราคืออะไร:
- สมมติกรณีเลวร้ายที่สุด (worst-case) คือ MAU ทั้งหมด 25 ล้านคนเข้าร่วมเล่นเกมในเดือนนั้น
- ID เป็น string ยาว 24 ตัวอักษรและคะแนนเป็นจำนวนเต็ม 16-bit เราต้องใช้พื้นที่ 26 byte * 25 ล้าน = ~650MB สำหรับการจัดเก็บ
- แม้จะเพิ่มต้นทุนการจัดเก็บเป็นสองเท่าเนื่องจาก overhead ของ skip list ก็ยังคงสามารถใส่ใน redis cluster สมัยใหม่ได้อย่างสบาย ๆ

ความต้องการที่ไม่ใช่ฟังก์ชันการทำงานอีกข้อที่ต้องพิจารณาคือการรองรับการอัปเดต 2500 ครั้งต่อวินาที ซึ่งอยู่ในขีดความสามารถของ Redis server ตัวเดียวได้สบาย ๆ

ข้อควรระวังเพิ่มเติม:
- เราสามารถสร้าง Redis replica เพื่อหลีกเลี่ยงการสูญเสียข้อมูล เมื่อ redis server ตัวหลักล่ม
- เรายังคงสามารถใช้ประโยชน์จาก Redis persistence เพื่อไม่ให้สูญเสียข้อมูลเมื่อเกิด crash
- เราจะต้องมีตารางสนับสนุนอีกสองตารางใน MySQL เพื่อดึงรายละเอียดของผู้ใช้ เช่น username, display name เป็นต้น รวมถึงจัดเก็บข้อมูลเมื่อผู้ใช้ชนะเกม
- ตารางที่สองใน MySQL สามารถใช้เพื่อสร้าง leaderboard ขึ้นใหม่ในกรณีที่โครงสร้างพื้นฐานล้มเหลว
- เพื่อเพิ่มประสิทธิภาพเล็กน้อย เราสามารถแคชรายละเอียดของผู้ใช้ 10 อันดับแรก เนื่องจากจะถูกเข้าถึงบ่อยครั้ง

---

## ขั้นตอนที่ 3: เจาะลึกการออกแบบ (Design Deep Dive)

### **การเลือกใช้ผู้ให้บริการ cloud หรือไม่ (To use a cloud provider or not)**

เราสามารถเลือก deploy และจัดการบริการเองได้ หรือใช้ผู้ให้บริการ cloud มาจัดการแทนเรา

หากเราเลือกจัดการบริการด้วยตัวเอง เราจะใช้ redis สำหรับข้อมูล leaderboard, mysql สำหรับ user profile และอาจใช้ cache สำหรับ user profile หากต้องการขยาย (scale) ฐานข้อมูล:

<div style="margin-left:3rem">
    <img src="./images/manage-services-ourselves.png" alt="manage-services-ourselves" width="500" />
</div>

ในทางกลับกัน เราสามารถใช้บริการของ cloud เพื่อจัดการหลายบริการแทนเราได้ ตัวอย่างเช่น เราสามารถใช้ AWS API Gateway เพื่อ route API call ไปยัง AWS Lambda function:

<div style="margin-left:3rem">
    <img src="./images/api-gateway-mapping.png" alt="api-gateway-mapping" width="500" />
</div>

AWS Lambda ช่วยให้เราสามารถรันโค้ดได้โดยไม่ต้องจัดการหรือจัดเตรียมเซิร์ฟเวอร์เอง มันจะรันเฉพาะเมื่อจำเป็นและขยายตัวเอง (scale) โดยอัตโนมัติ

ตัวอย่างเมื่อผู้ใช้ทำคะแนนได้:

<div style="margin-left:3rem">
    <img src="./images/user-scoring-point-lambda.png" alt="user-scoring-point-lambda" width="500" />
</div>

ตัวอย่างเมื่อผู้ใช้ดึงข้อมูล leaderboard:

<div style="margin-left:3rem">
    <img src="./images/user-retrieve-leaderboard.png" alt="user-retrieve-leaderboard" width="500" />
</div>

Lambda เป็นการนำสถาปัตยกรรมแบบ serverless มาใช้ เราไม่จำเป็นต้องจัดการเรื่องการขยาย (scaling) และการตั้งค่าสภาพแวดล้อม (environment setup)

ผู้เขียนแนะนำให้ใช้แนวทางนี้หากเราสร้างเกมขึ้นมาใหม่ตั้งแต่ต้น

### **การขยาย (scaling) Redis**

ด้วย 5 ล้าน DAU เราสามารถใช้ Redis instance เพียงตัวเดียวได้ ทั้งในแง่ของการจัดเก็บและ QPS

แต่หากลองสมมติว่าฐานผู้ใช้เติบโตขึ้น 10 เท่า เป็น 500 ล้าน DAU เราจะต้องการพื้นที่จัดเก็บ 65GB และ QPS จะเพิ่มขึ้นเป็น 250,000

ขนาดในระดับนี้จำเป็นต้องใช้การทำ sharding

วิธีหนึ่งในการบรรลุผลนี้คือการทำ range-partitioning ข้อมูล:

<div style="margin-left:3rem">
    <img src="./images/range-partition.png" alt="range-partition" width="500" />
</div>

ในตัวอย่างนี้ เราจะทำ shard ตามคะแนนของผู้ใช้ เราจะรักษาความสัมพันธ์ระหว่าง user_id และ shard ไว้ในโค้ดของแอปพลิเคชัน
เราสามารถทำได้ทั้งผ่าน MySQL หรือ cache อีกตัวสำหรับความสัมพันธ์นี้เอง

ในการดึงผู้เล่น 10 อันดับแรก เราจะ query shard ที่มีคะแนนสูงสุด (`[900-1000]`)

ในการดึงอันดับของผู้ใช้ เราจะต้องคำนวณอันดับภายใน shard ของผู้ใช้นั้น แล้วบวกจำนวนผู้ใช้ทั้งหมดที่มีคะแนนสูงกว่าใน shard อื่น ๆ เข้าไป
การคำนวณอย่างหลังนี้เป็น operation แบบ O(1) เนื่องจากจำนวน record ทั้งหมดต่อ shard สามารถเข้าถึงได้อย่างรวดเร็วผ่านคำสั่ง info keyspace

อีกทางเลือกหนึ่ง เราสามารถใช้ hash partitioning ผ่าน Redis Cluster ซึ่งเป็น proxy ที่กระจายข้อมูลไปยัง redis node ต่าง ๆ โดยอิงตาม partitioning ที่คล้ายกับ consistent hashing แต่ไม่เหมือนกันเสียทีเดียว:

<div style="margin-left:3rem">
    <img src="./images/hash-partition.png" alt="hash-partition" width="500" />
</div>

การคำนวณผู้เล่น 10 อันดับแรกนั้นทำได้ยากขึ้นด้วยวิธีนี้ เราจะต้องดึงผู้เล่น 10 อันดับแรกของแต่ละ shard แล้วนำผลลัพธ์มารวมกันในแอปพลิเคชัน:

<div style="margin-left:3rem">
    <img src="./images/top-10-players-calculation.png" alt="top-10-players-calculation" width="500" />
</div>

มีข้อจำกัดบางประการเมื่อใช้ hash partitioning:
- หากเราต้องการดึงผู้เล่น K อันดับแรกที่มีค่า K สูง latency จะเพิ่มขึ้น เนื่องจากเราต้องดึงข้อมูลจำนวนมากจากทุก shard
- latency จะเพิ่มขึ้นเมื่อจำนวน partition เพิ่มมากขึ้น
- ไม่มีวิธีที่ตรงไปตรงมาในการหาอันดับของผู้ใช้

ด้วยเหตุนี้ ผู้เขียนจึงมีแนวโน้มที่จะใช้ fixed partition สำหรับปัญหานี้

ข้อควรระวังอื่น ๆ:
- แนวปฏิบัติที่ดีคือจัดสรรหน่วยความจำเป็นสองเท่าของที่จำเป็น สำหรับ redis node ที่มีการเขียนหนัก เพื่อรองรับการทำ snapshot หากจำเป็น
- เราสามารถใช้เครื่องมือชื่อ Redis-benchmark เพื่อติดตามประสิทธิภาพของการตั้งค่า redis และตัดสินใจโดยอิงจากข้อมูล

### **โซลูชันทางเลือก: NoSQL (Alternative solution: NoSQL)**

โซลูชันทางเลือกอีกแบบที่ควรพิจารณาคือการใช้ฐานข้อมูล NoSQL ที่เหมาะสม ซึ่งถูก optimize สำหรับ:
- การเขียนที่หนักหน่วง (heavy writes)
- การเรียงลำดับรายการภายใน partition เดียวกันตามคะแนนอย่างมีประสิทธิภาพ

DynamoDB, Cassandra หรือ MongoDB ล้วนเป็นตัวเลือกที่เหมาะสม

ในบทนี้ ผู้เขียนตัดสินใจใช้ DynamoDB ซึ่งเป็นฐานข้อมูล NoSQL แบบ fully-managed ที่มอบประสิทธิภาพที่น่าเชื่อถือและความสามารถในการขยายที่ยอดเยี่ยม
มันยังช่วยให้เราสามารถใช้ global secondary index ได้ เมื่อเราต้องการ query field ที่ไม่ใช่ส่วนหนึ่งของ primary key

<div style="margin-left:3rem">
    <img src="./images/dynamo-db.png" alt="dynamo-db" width="500" />
</div>

มาเริ่มจากตารางสำหรับจัดเก็บ leaderboard ของเกมหมากรุก (chess):

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-1.png" alt="chess-game-leaderboard-table-1" width="500" />
</div>

วิธีนี้ใช้ได้ดี แต่ไม่สามารถขยายได้ดีหากเราต้องการ query ตามคะแนน ดังนั้นเราสามารถใส่คะแนนเป็น sort key แทน:

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-2.png" alt="chess-game-leaderboard-table-2" width="500" />
</div>

ปัญหาอีกข้อหนึ่งของการออกแบบนี้คือ เราทำ partition ตามเดือน ซึ่งนำไปสู่ปัญหา hotspot partition เนื่องจากเดือนล่าสุดจะถูกเข้าถึงอย่างไม่สม่ำเสมอเมื่อเทียบกับเดือนอื่น ๆ

เราสามารถใช้เทคนิคที่เรียกว่า write sharding ซึ่งเราจะต่อท้าย partition number ในแต่ละ key โดยคำนวณจาก `user_id % num_partitions`:

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-3.png" alt="chess-game-leaderboard-table-3" width="500" />
</div>

trade-off ที่สำคัญที่ต้องพิจารณาคือจำนวน partition ที่เราควรใช้:
- ยิ่งมี partition มากเท่าไร ความสามารถในการขยายด้านการเขียน (write scalability) ก็จะยิ่งสูงขึ้น
- อย่างไรก็ตาม ความสามารถในการขยายด้านการอ่าน (read scalability) จะแย่ลง เนื่องจากเราต้อง query หลาย partition เพื่อรวบรวมผลลัพธ์แบบ aggregate

การใช้แนวทางนี้ต้องอาศัยเทคนิค "scatter-gather" ที่เราเห็นมาก่อนหน้านี้ ซึ่งจะมี time complexity เพิ่มขึ้นเมื่อเราเพิ่ม partition มากขึ้น:

<div style="margin-left:3rem">
    <img src="./images/scatter-gather-2.png" alt="scatter-gather-2" width="500" />
</div>

เพื่อประเมินจำนวน partition ที่เหมาะสม เราจำเป็นต้องทำการ benchmark

แนวทาง NoSQL นี้ยังคงมีข้อเสียสำคัญข้อหนึ่ง - เป็นเรื่องยากที่จะคำนวณอันดับ (rank) เฉพาะของผู้ใช้แต่ละคน

หากเรามีขนาดระบบมากพอที่จำเป็นต้อง shard เราก็อาจสามารถบอกผู้ใช้ได้ว่าคะแนนของเขาอยู่ใน "เปอร์เซ็นไทล์ (percentile)" เท่าไร

cron job สามารถรันเป็นระยะ ๆ เพื่อวิเคราะห์การกระจายตัวของคะแนน ซึ่งจะใช้กำหนดเปอร์เซ็นไทล์ของผู้ใช้แต่ละคน ตัวอย่างเช่น:

```
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
```

---

## ขั้นตอนที่ 4: สรุป (Wrap Up)

เรื่องอื่น ๆ ที่สามารถพูดคุยได้เพิ่มเติม หากมีเวลาเหลือ:
- **การดึงข้อมูลที่รวดเร็วขึ้น (Faster retrieval)** - เราสามารถแคช user object ผ่าน Redis hash โดย mapping `user_id -> user object` วิธีนี้ช่วยให้การดึงข้อมูลเร็วขึ้นเมื่อเทียบกับการ query ฐานข้อมูล
- **การตัดสินเมื่อคะแนนเสมอกัน (Breaking ties)** - เมื่อผู้เล่นสองคนมีคะแนนเท่ากัน เราสามารถตัดสินโดยเรียงลำดับตามเกมที่เล่นล่าสุด
- **การกู้คืนระบบเมื่อล้มเหลว (System failure recovery)** - ในกรณีที่ Redis เกิดขัดข้องในวงกว้าง เราสามารถสร้าง leaderboard ขึ้นใหม่ได้ โดยไล่ดู MySQL WAL entry แล้วสร้างขึ้นใหม่ผ่าน ad-hoc script
