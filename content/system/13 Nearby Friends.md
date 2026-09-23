---
title: 13 Nearby Friends
tags:
  - system
  - systemdesign
  - nearby_friends
  - websocket
  - peer_to_peer
  - pub_sub
  - fanout
  - sharding
  - hash_ring
---
Nearby Friends คือ System ที่ไว้ใช้หาว่า User ที่อนุญาตให้ Device ของเราเข้าถึงตำแหน่งของ User นั้นอยู่ห่างจาก Device ของเราเป็นระยะกี่เมตร เป็นระบบที่ก็พบเห็นได้ในหลายๆ จังหวะ เช่น Find My ของ iOS หรือว่า Feature เพื่อนรอบๆ ตัวใน Facebook ไรงี้ แล้วจะ implement ขึ้นมายังไง เรามาดูกัน

![[nearby_friends_overview.png]]

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: ไอ่คำว่ารอบๆ ตัวนี่ ระยะห่างประมาณไหนเหรอครับ?
A1: เอาซัก 5km ละกัน แต่ว่า User สามารถปรับได้นะ

Q2: แล้วระยะห่างนี่... วัดแบบการกระจัด คือลากเส้นตรงจาก User ไป User เลยมั้ย หรือว่าวัดตามเส้นถนน หรือวัดแบบไหนเหรอครับ
A2: เอาเป็นการกระจัดละกัน

Q3: แล้วมีผู้ใช้งานเยอะขนาดไหนเหรอครับ?
A3: 10 Million Users ละกัน

Q4: สมมติว่า User ที่ Device ของเรารู้จักไม่ได้ใช้งาน Feature นี้เป็นเวลานานๆ User คนนั้นจะหายไปจาก List เลย หรือว่าให้ระบบ mark เป็นสถานที่ล่าสุดที่ User นั้นอยู่ และ Device มีข้อมูลของสถานที่นั้น หรือแบบไหนดีครับ?
A4: เอาเป็นว่า...เอาออกจาก List เลยละกัน

Q5: ถ้าเป็นตาม A4 แล้วควรจะมีการเก็บข้อมูลสถานที่มั้ยครับ
A5: ก็ควรอยู่ดีนะ เอาไว้ research ใน field อื่นๆ ได้ เช่น Machine Learning งี้

### Functional Requirement

- User ควรจะเห็น User คนอื่นๆ รอบตัวได้ โดยจะมีข้อมูลระยะห่าง ละก็มีการบอกว่าข้อมูลนี้มีการอัพเดทล่าสุดเมื่อไหร่
- ข้อมูลของ User คนอื่นๆ จะต้องมีการอัพเดทเองอยู่เรื่อยๆ 

### Non-Functional Requirement

- Low Latency - Delay ในการส่งข้อมูลตำแหน่งของ User คนอื่นๆ ควรจะต่ำ
- Reliability - ในภาพรวม ระบบควรจะต้องทำงานได้อย่างถูกต้อง อาจจะมีข้อมูลที่คาดเคลื่อนไปจากความจริงนิดหน่อยก็ยังถือว่ายอมรับได้

---

## 2. Sketch ภาพระบบคร่าวๆ

วิธีการนึงที่ทำได้ คือการให้ User ทุกคนคอยอัพเดทข้อมูลของตนเองอยู่เรื่อยๆ ว่าตนเองอยู่ตรงไหน แล้วให้ User คนอื่นๆ จดไว้ แล้วนำไปคำนวณ ทำแบบนี้กับทุก User ก็จะสามารถแสดงข้อมูลของทุก User ได้แล้ว 

Concept แบบนี้จะคล้ายคลึงกับหลักการ Peer-to-Peer นั่นเอง

> [!Note] Peer-to-Peer
> **Peer-to-Peer (P2P)** คือ รูปแบบการเชื่อมต่อ และแลกเปลี่ยนข้อมูลระหว่างคอมพิวเตอร์หรืออุปกรณ์หลายเครื่องโดยตรง โดยแต่ละเครื่องเป็นทั้งผู้ให้บริการ (Server) และผู้รับบริการ (Client) พร้อมกันในตัว โดยไม่ต้องพึ่งพาเซิร์ฟเวอร์กลาง
> 
> ![[nearby_friends_peer_to_peer.png]]

แต่ว่าสำหรับ Mobile นั้นมันไม่เวิร์ค เพราะ Mobile มันไม่สามารถเป็น Server ได้ ด้วยข้อจำกัดของพลังงานที่มีไม่พอจะเป็น Server ได้ ก็เลยปัดตกไป

เพราะงั้นเราก็จะย้อนกลับมาสู่วิธีการที่เรารัก อย่าง Client-Server แทน แต่พอเป็น Client-Server แล้ว... Server จะต้องทำงานหนักมากเลยนะ สมมติว่าต้องมีการ update ข้อมูลตำแหน่งของ User ทุกๆ 30 วินาทีแล้วเนี่ย... ละสมมติว่าแต่ละ User มี Nearby Friends 40 คนที่ active อยู่

เท่ากับว่าจะมีการ update location ถึงราวๆ 14 ล้านครั้งต่อวินาที ขี้แตกกันพอดีถ้าไม่จัดสรรดีๆ

### High-Level Design

ใดๆ ก็ เราจะเริ่มจาก Design สำหรับ Scale ไม่ใหญ่มากก่อนละกัน ละจะค่อยๆ add on เพิ่มไปเรื่อยๆ สำหรับ Scale ที่ใหญ่ขึ้น

![[nearby_friends_high-level_design.png]]

แน่นอนว่าด้วยความที่ต้องมีการ update ข้อมูลของเพื่อนอยู่ตลอด การใช้ Websocket ก็ตอบโจทย์ (อ่านเพิ่มเติมได้ใน [[08 Chat System#Websocket|Chat System]])

โดยจะแยกข้อมูลออกเป็น 2 ส่วน คือส่วนที่ไม่ต้อง Real-Time ก็ปล่อยให้ API Server จัดการ เช่น Profile ของเพื่อน, การเพิ่ม/แก้ไข/ลบเพื่อน เป็นต้น อีกส่วนคือ Websocket ไว้ใช้สำหรับ update ตำแหน่งของ User แบบ Real Time เพื่อส่งตำแหน่ง และข้อมูลสถานที่ให้กับ User คนอื่นๆ

ของใหม่คือเพิ่มขึ้นมา คือ Redis Pub/Sub จะลงรายละเอียดเพิ่มเติมหน่อยละกัน

![[nearby_friends_redis_pubsub.png]]

Redis Pub/Sub คือ วิธีการส่งข้อมูลแบบ Asynchronous รูปแบบหนึ่งที่เรียกว่า Publisher and Subscriber ใน Redis ซึ่ง Publisher and Subscriber จะมีหลักการสำคัญ คือการที่ Publisher จะส่งข้อมูลให้กับ Subscriber ที่ subscribe Publisher นั้นเท่านั้น โดยข้อมูลที่ Publisher ส่งมาจะถูกเก็บไว้ใน Channel แล้วรอให้ Subscriber มาดึงไปเอง (คล้ายๆ Fanout Read อ่ะ) ตามภาพประกอบด้านบนเลย

เผื่อว่าไม่เห็นภาพ... สมมติก่อนว่า User1 เป็นเพื่อนกับ User2 และ User3 และ User5 เป็นเพื่อนกับ User4, User6 ละกัน

![[nearby_friends_redis_pubsub_example.png]]

เมื่อถึงเวลาที่ต้อง update ข้อมูลตำแหน่ง User1 และ User5 ก็จะมีการส่ง Request เพื่อจะ update ข้อมูลของตนมายัง Websocket Servers จากนั้นจะแบ่งเป็น 2 ส่วน

1. Websocket Servers จะส่งข้อมูลตำแหน่งของ User1 และ User5 ไปเก็บใส่ Database ที่เก็บเรื่อง Location ไว้ รวมถึงเก็บไว้ใน Cache ที่เก็บเรื่อง Location เหมือนกัน (ใน Database จะเก็บไว้เป็นข้อมูล Location ทั้งหมด หรือก็คือ Location History ก็ได้ แต่ว่าใน Cache เอาแค่ Location ล่าสุดก็พอ)

2. Websocker Servers จะส่งข้อมูลตำแหน่งของ User1 ไปไว้ใน Channel ของ User1 ส่วนข้อมูลตำแหน่งของ User5 ก็เอาไปไว้ใน Channel ของ User5 จากนั้น Websocket Servers ก็จะดึงข้อมูลจาก Channel ของ User1 ไปอัพเดทให้ User2 และ User3 และดึงข้อมูลจาก Channel ของ User5 ไปอัพเดทให้ User4 และ User6

### Client Initialization

แล้วเมื่อไหร่ที่ถือว่าเป็นจุดเริ่มต้นกันนะ... คำตอบคือ ตั้งแต่ตอนที่ User เปิดแอปนั่นแหละ โดยที่เมื่อ User เปิดแอป Client ก็จะสร้าง WebSocket Connection ค้างไว้กับ Server ตัวหนึ่ง แล้ว Client ก็จะส่งตำแหน่งของ User ให้กับ Server จากนั้น Server จะทำ 5 อย่างต่อกัน 
1. เก็บตำแหน่งของ Client ลง Location Cache,
2. ดึงข้อมูล Friends ทั้งหมดจาก User DB
3. ยิง Batch Request ไปดึงตำแหน่งเพื่อนทุกคนจาก Cache รวดเดียว
4. คำนวณระยะห่างแล้วส่งเฉพาะเพื่อนที่อยู่ในรัศมีกลับไปแสดง
5. Subscribe เข้า Channel ของเพื่อนทุกคนใน Redis Pub/Sub เพื่อให้ Friends นำไป update ข้อมูล

### API Design

1. เส้นสำหรับเพิ่ม / แก้ไข / ลบเพื่อน
2. เส้นสำหรับแก้ User Profile
3. เส้นสำหรับ subscribe / unsubscribe User คนอื่นๆ ในระบบ โดยใช้ User ID เพื่อบอกว่าจะ subscribe หรือ unsubscribe
4. เส้นสำหรับ initialize Websocket เพื่อใช้ส่งและรับ lat, lng และ timestamp
5. เส้นสำหรับ update location ของ User โดยจะให้ User ส่ง lat, lng ละก็ timestamp ให้ไปกับ request
6. เส้นสำหรับดึงข้อมูล location ของ User ที่มีการ update เข้ามา

ก็จะทำให้ Schema ที่ได้ จะประกอบด้วย Location(user_id, lat, lng, timestamp) นั่นเอง

---

## 3. เจาะลึกส่วนที่สำคัญๆ

หลักๆ ของเรื่องนี้คือเราจะ scale ระบบยังไงให้รองรับการใช้งานเยอะๆ ไหว แล้วจะ scale อะไรได้บ้างนะ...

### Scale API และ WebSocket Servers

-> API Server เป็น Stateless จึง Auto-scale ตาม Load ได้เลย
-> แต่ WebSocket Server เป็น **Stateful** เพราะถือ Connection ค้างไว้ ตอนจะถอด Server ออกจึงถอดดื้อๆ ไม่ได้ ต้องรอให้ Connection ที่ค้างอยู่ปิดหมดก่อน 

เพราะงั้นการจะ scale Websocket Server ได้ คือจะต้อง mark Server นั้นเป็น "draining" ที่ Load Balancer ก่อน เพื่อไม่ให้ Websocket Connection ใหม่วิ่งเข้ามาอีก แล้วทยอยปิด Connection จนหมด จึงจะสามารถถอด Server แล้ว scaling ได้ กระทั่งการ deploy เวอร์ชันใหม่ก็ต้องระวังแบบเดียวกัน

### Scale User and Location DB และ Location Cache

-> **User DB และ Location DB** สามารถทำ Sharding ตาม `user_id` ได้ เพราะข้อมูลของแต่ละ User เป็นอิสระต่อกัน
-> **Location Cache** มีการใช้ Redis เก็บตำแหน่งล่าสุดของทุกคน พร้อมตั้ง TTL ที่ต่ออายุทุกครั้งที่ update ข้อมูลของ Friends ทำให้ในกรณีที่เพื่อนหายไปนานเกิน TTL ตำแหน่งจะหลุดออกจาก Cache เอง ซึ่งก็ตรงกับ requirement รวมถึงปัญหาการที่ต้อง update ราวๆ 334,000 ครั้งต่อวินาที ซึ่งสูงเกินไปสำหรับ Redis ตัวเดียวนั้น ก็แก้ได้ด้วยการ Sharding ตาม `user_id` เพราะข้อมูลตำแหน่งของแต่ละคนเป็นอิสระ กระจายโหลดลงหลายเครื่องได้

### Distributed Pub/Sub: Consistent Hashing

Pub/Sub ที่มีหน้าที่ store ข้อมูลจาก Publisher และส่งข้อมูลให้ Subscriber พอต้องมี Pub/Sub จำนวนมากๆ เข้า แล้วจะกระจาย channel ไปเครื่องไหนยังไง 

ก่อนอื่นเลย ทำไมต้องมี Pub/Sub เยอะด้วยอ่ะ ไม่ใช่ว่ามันเป็น Redis เหรอ มันทำงานบน Memory มันก็น่าจะเร็วมะ ตรงนั้นก็ใช่แหละ ไอ่การเก็บข้อมูลอ่ะไม่ใช่ปัญหาเลย ปัญหาคือการ push ข้อมูลตำแหน่งซึ่งมันต้อง push เยอะมาก ตรงนี้แหละที่ทำให้ต้องมี Pub/Sub เยอะ

ทีนี้ย้อนกลับมาที่คำถามที่ว่า...แล้วจะกระจาย channel ไปเครื่องไหนยังไง?

คำตอบคือใช้ **Consistent Hashing** เช่น Hash Ring โดยการเอาชื่อ channel มา hash แล้วตัดสินว่าอยู่เครื่องไหนบน Hash Ring ทั้งฝั่งที่ Publish และ Subscribe ทำให้ทั้ง Publisher และ Subscriber จะเจอ Channel บนเครื่องเดียวกัน (อ่านต่อได้ใน [[03 Consistent Hashing|Consistent Hashing]])

![[nearby_friends_pubsub_hashring.png]]

>[!Note]
>Pub/Sub Cluster ถือว่าป็น **Stateful** เพราะมีการจด Subscriber เอาไว้ว่า Channel นี้มี Subscriber อะไรบ้าง เพราะงั้นการจะ scale Pub/Sub ก็ต้องระวังเหมือนกับ Websocket Servers เลย เพื่อไม่ให้การ subscribe เสียหาย
>
>โดยการ scale จะมี 2 สถานการณ์
>1. **Resize ทั้ง Cluster** (เพิ่ม/ลดเครื่อง) ซึ่งเสี่ยงมาก เพราะ channel จำนวนมากย้ายเครื่องพร้อมกัน เกิด resubscribe มหาศาลและอาจมี update หลุด จึงควรทำตอน traffic ต่ำสุดของวัน
>2. **แทนเครื่องที่ล่มทีละตัว** จะเสี่ยงต่ำกว่ามาก เพราะกระทบแค่ channel บนเครื่องนั้นเครื่องเดียว แค่อัปเดต hash ring ให้ชี้ไปเครื่องสำรอง แล้ว resubscribe เฉพาะ channel ที่ได้รับผลกระทบ
>
>   ![[nearby_friends_replace_server.png]]


---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Nearby Friends ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---