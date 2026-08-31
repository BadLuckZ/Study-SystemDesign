---
tags:
  - system
  - systemdesign
  - news_feed_system
  - cache
  - load_balancer
  - cdn
  - fanout
  - graphdb
  - message_queue
---
วันนี้จะเป็นระบบ Feeds ละกันนะ คิดภาพเหมือน Facebook กับ Instagram อ่ะที่จะมีการแสดง Content ต่างๆ ในหน้า Homepage อยู่ตลอด ไม่ว่าจะเปิดกี่ครั้งต่อกี่ครั้งก็ตาม โดยแต่ละ Feed ก็จะมีพวก Content, รูป, Video, จำนวน Likes, Comments และอื่นๆ อีกพอควร แล้วเราจะสร้างขึ้นมาได้ยังไง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Feeds นี้จะให้เป็น Web App, Mobile App หรือว่าทั้งคู่อ่ะ?
A1: ทั้งคู่ๆ

Q2: Feature ที่สำคัญที่สุดของระบบนี้คืออะไรเหรอ?
A2: ที่ต้องมีแน่ๆ คือ User จะต้องสร้างและปล่อย Post ได้ และสามารถเห็น Post ของเพื่อนได้

Q3: จะเรียงลำดับของ Post ยังไงดี? เรียงตามเวลาที่ปล่อย Post เหรอ หรือเรียงตามความสนิทระหว่างตนกับคน Post + ความสนใจของคน Post หรือยังไงดี
A3: เอาแบบชิวๆ อย่างเรียงตามเวลา post ละกันนะ

Q4: Traffic ประมาณไหนเหรอครับ?
A4: ประมาณ 10 ล้าน DAU ละกัน (Daily Active Users)

Q5: Post หรือ Feed เนี่ย มีรูป หรือ Video มั้ย?
A5: มีครับ มีทั้งคู่เลย

---

## 2. Sketch ภาพระบบคร่าวๆ

### Feed APIs

ก็..API มันจะมีที่สำคัญๆ อยู่ 2 เส้นอ่ะนะ คือ

1. **POST** `/api/v1/me/feed` - API สำหรับการสร้าง Feed ของ Users
```swagger
POST /api/v1/me/feed
	header: {
		auth_token
	},
	request parameter: {
		content
	}
```

2. **GET** `/api/v1/me/feed` - API สำหรับการดึง Feeds มาแสดงให้ Users ดู
```swagger
GET /api/v1/me/feed
	header: {
		auth_token
	},
```

### Post Publishing

Flow การใช้งานเวลาจะสร้าง Feed ใหม่ให้ User จะเป็นประมาณนี้
1. User ระบุ content เข้ามา จากนั้นเรียก API เพื่อส่งข้อมูล
2. ระบบนำ Request จาก User ส่งไปยัง Server ที่เหมาะสมด้วย Load Balancer
3. Server ดังกล่าวส่งข้อมูลต่อไปให้ **Post Service** เพื่อให้ Post Service สร้าง Feed ใหม่ให้
4. เก็บ Post ใหม่ที่สร้างไว้ใน Database ของ Post Service และเก็บ Post ID ไว้ใน **Post Cache** 
	- ที่เก็บไว้ใน Post Cache เพราะจะได้เอามาแสดงให้ User เห็นได้ทันทีตอนกลับมาหน้าหลัก
	- ที่เก็บเฉพาะ ID เพราะ 1 Feed มันมีทั้ง Content, Link รูป, Link Video, จำนวน Likes/Comments อีก เวลามีการแก้ไขมันก็ต้องอัพเดทตรงนี้ด้วย ซึ่งยุ่งยากเกินจำเป็น
5. [ตรงนี้สำคัญ!] ระบบจะนำ Post ID อันใหม่นี้ไปใส่ไว้ใน Feed Cache ของเพื่อนทุกคนที่ควรเห็น Post นี้ด้วย ผ่านกระบวนการใน **Fanout Service**
6. ระบบจะมีการยิง Notification ไปหาเพื่อนด้วย ผ่าน **Notification Service**

### Feed Building

เมื่อเราสร้าง Post เข้ามาแล้ว สิ่งถัดมาคือการดู Post ซึ่งระบบจะมีการทำงานดังนี้
1. เมื่อ User เปิดหน้าหลัก ระบบจะมีการดึง Post ต่างๆ ผ่าน API เพื่อนำมาสร้างเป็น Feed ของ User
2. ระบบส่งข้อมูลของ User ไปยัง Servers ผ่าน Load Balancer
3. Servers ส่ง Request ไปให้ Feed Service เพื่อดึง ID ออกมาจาก Feed Cache
4. เอา List ของ Feed ID ที่ได้มา ไปยิง Request ต่อเพื่อดึงข้อมูลแบบเต็มๆ ของแต่ละ Post
5. เอาข้อมูลทั้งหมดจากขั้นตอนที่ 4 มาประกอบร่างรวมกันเป็น Feed ที่สมบูรณ์ ตามลำดับที่ Feed Cache เรียงไว้ (ใน Case นี้คือเรียงตามเวลา Post ตามที่ตกลง Scope กันไว้)
6. ส่ง Feed ที่ประกอบร่างเสร็จแล้วกลับไปให้ User แสดงผลที่หน้าหลัก

ดังนั้น จากทั้ง 2 Flow เลยวาด Diagram ออกมาได้ตามภาพ
![[feed_system_v1.png]]

---

## 3. เจาะลึกจุดสำคัญๆ

### Fanout

จาก Flow การใช้งาน จุดสำคัญของระบบนี้ คือ Fanout Service เพราะมันเป็น Service สำคัญที่ทำให้ User ที่เป็นเพื่อนของ User เห็น Post เราได้ แต่ User ที่ไม่ได้เป็นก็จะไม่ได้เห็น ทำได้ยังไงมาดูกัน

ก่อนอื่นเลย **Fanout** คือกระบวนการกระจาย Post ID ของ Post ใหม่ ไปใส่ไว้ใน Feed Cache ของทุกคนที่ควรเห็น Post นั้น (เช่น เพื่อนของคนโพสต์) วิธีการทำมีอยู่ 2 แบบหลักๆ
#### 1. Fanout on Write

ทำตอนที่มี Post ใหม่เข้ามาเลย คือพอ User โพสต์ปุ๊บ ระบบจะไปหา List เพื่อนทั้งหมดของ User คนนั้น ผ่านการใช้ GraphDB (Database ที่เก็บความสัมพันธ์ระหว่าง User เช่นใครเป็นเพื่อนใคร) แล้วเอา Feed ID เข้าไปใส่ใน Feed Cache ของเพื่อนทุกคนทันที

**ข้อดี:**
- ตอน User เปิดหน้า Feed จะเร็วมาก เพราะ Feed Cache เตรียมพร้อมรอไว้แล้ว
**ข้อเสีย:** 
- ถ้าคนเพิ่มโพสต์มีผู้ติดตามหลักล้านคน จะทำให้การ Fanout ไปหาทุกคนพร้อมกันจะกิน Resource มหาศาลและช้ามาก - Hotkey Problem
- บาง Follower อาจไม่ได้เปิดแอปมาดูเลยด้วยซ้ำอีก ก็เปลือง Resources แบบไม่จำเป็น
#### 2. Fanout on Read

ไม่ทำอะไรตอน Post เลย แค่บันทึก Post ไว้เฉยๆ พอ User คนไหนเปิดหน้า Feed ค่อย "Pull" (ดึง) Post ล่าสุดจากเพื่อนทุกคนมารวมกัน แล้วเรียงลำดับให้สดๆ ตอนนั้นเลย

**ข้อดี:**
- ไม่เปลืองทรัพยากรไปกับคนที่ไม่ได้เข้ามาดู Feed เพราะคำนวณเฉพาะตอนที่มีคน Request จริงๆ 
**ข้อเสีย:** 
- ตอนเปิดหน้า Feed จะช้ากว่า เพราะต้องไปรวบรวมและคำนวณสดๆ ทุกครั้ง โดยเฉพาะถ้า User มีเพื่อนเยอะมาก

ด้วยเหตุนี้ เราเลยทำระบบ Hybrid ขึ้นมา โดยจะปรับ Algorithm ตามจำนวนเพื่อนของ User
- User ทั่วไป (เพื่อนไม่เยอะ) → ใช้ **Fanout on Write** เพราะ Read เร็ว และ Cost การ Push ไม่สูงมาก
- Celebrity / User ที่ผู้ติดตามเยอะ → ใช้ **Fanout on Read** แทน คือไม่ Push ออกไปหาทุกคน แต่ให้ Follower ที่เปิด Feed ค่อยไป Pull Post ของ Celebrity มารวมกับ Feed ปกติของตัวเองตอนนั้นเลย

Diagram ในส่วนของ Fanout Service เลยจะออกมาเป็นแบบนี้
![[feed_system_fanout.png]]

1. **ดึง List เพื่อนจาก Graph DB**
2. **กรองเพื่อนตาม Setting** (Mute, แชร์เฉพาะบางคน/ซ่อนบางคน) เช่น ถ้า A Mute B ไว้ Post ของ B จะไม่ขึ้น Feed ของ A แม้ยังเป็นเพื่อนกันอยู่ เป็นต้น
3. **ส่ง List เพื่อน + Post ID เข้า Message Queue**
4. **Fanout Worker ดึงมาเก็บใน News Feed Cache** โดยทำเป็นคู่ `<post_id, user_id>` พอมี Post ใหม่ก็ Append แถวเข้าไป โดย**เก็บแค่ ID เพื่อประหยัด Memory**
### CDN (Content Delivery Network)
อ่านเนื้อหาเพิ่มเติมได้ที่ [[07 CDN (Content Delivery Network)|CDN]]

![[feed_system_cdn.png]]

เรามีการใช้ CDN เพื่อทำให้การดึงข้อมูลที่มีขนาดใหญ่ เช่น รูป หรือ Video ทำได้เร็วขึ้น โดยเมื่อนำ CDN มาใช้ การดึง Feed ของ User จึงมี Flow ประมาณนี้
1. User ส่ง Request ขอ Feed
2. Load Balancer กระจาย Request
3. Web Server เรียก News Feed Service
4. ดึง List Post ID จาก News Feed Cache: ข้อมูลที่ได้จะเป็น `<post_id, user_id>`
5. ดึงข้อมูลเต็มจาก User Cache ด้วย `user_id` + Post Cache ด้วย `post_id` มา Hydrate เพราะ Feed ต้องมี Username, รูปโปรไฟล์, เนื้อหา ฯลฯ ไม่ใช่แค่ ID
6. ส่งกลับเป็น **JSON** ให้ Client ไป Render

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ News Feed System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
