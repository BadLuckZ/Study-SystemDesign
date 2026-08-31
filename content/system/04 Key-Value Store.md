---
tags:
  - system
  - systemdesign
  - key_value
  - database
  - hash_table
  - cap_theorem
  - database_optimization
  - database_partition
  - database_replication
  - database_consistency
  - versioning
  - hash_ring
  - virtual_nodes
  - cache
---
รอบนี้เรามาลอง implement Database รูปแบบนึงที่สามารถพบเห็นได้ในการทำงาน เช่น Redis อย่าง Key-Value Store ที่เป็นรูปแบบการเก็บข้อมูลแบบเป็นคู่ๆ คือ Key กับ Value นั่นเอง

---

## Problem Statement

เงื่อนไขที่อยากได้จาก Database นี้ได้แก่
1. Size ของ Key-Value ควรเล็กกว่า 10KB
2. สามารถเก็บค่าที่มีขนาดใหย๋ได้
3. มี Availability - ต่อให้ระบบจะล่ม ก็ยังสามารถดึงค่าออกมาได้อย่างรวดเร็ว
4. มี Scalability - ระบบจะต้อง scaled ให้รองรับ Data จำนวนมากได้
5. Auto Scale ได้ ตามปริมาณข้อมูลที่มี
6. Latency ต่ำ - เวลาที่ใช้ในการดึง Value ต่ำ

กรณีที่เราใช้ Server ตัวเดียวมันก็ไม่ใช้เรื่องยากอ่ะนะ ก็ทำเป็น Hash Table ก็จบละ ละก็อาจจะมี Option เพิ่มเติมอย่างการเก็บ Data ที่มีการเรียกบ่อยไว้ใน Memory เพื่อเพิ่มความรวดเร็ว นอกนั้นก็เก็บไว้ใน Disk ตามปกติอ่ะ

![[hash_table.png]]

แต่กับ Distributed System ที่มีหลาย Server อ่ะ มันก็คงไม่ง่ายอยู่ละเนอะ...

---

## Theory

สิ่งสำคัญที่เราต้องคำนึงในการสร้าง Database คือ [[01 Database#4. CAP Theorem|CAP Theorem]] ซึ่งโดยสรุปก็คือ
- `Consistency (C)` - User จะเห็นข้อมูลเหมือนเดิมตลอดไม่ว่าจะเปิดดูข้อมูลเมื่อไหร่ก็ตาม หรืออ่านข้อมูลจาก Node / Server ใดก็ตาม
- `Availability (A)` - Client ที่มีการส่ง Request ไปจะได้รับ Response ตลอด แม้ว่าจะมีบาง Node / Server พัง หรือไม่สามารถ connect กับ Network ได้ก็ตาม
- `Partition Tolerance (P)` - ระบบสามารถทำงานต่อได้ แม้จะมีบาง Node / Server พัง หรือไม่สามารถ connect กับ Network ได้ก็ตาม

ทำให้จะได้ Database ออกมา 3 ระบบ คือ `CP` `AP` และ `CA

ซึ่งในโลกความเป็นจริง ไม่มีทางที่จะเลี่ยงการที่มี Server ตัวนึงในระบบล่มไปได้ เพราะงั้นจะต้องทำให้ Key-Store Value มี Partition Tolerance อยู่แล้ว ทำให้เลือกได้อีก 1 อย่าง ระหว่าง Consistency กับ Availability

A) ถ้าเราเลือก Consistency สิ่งที่เราต้อง concern คือ การ Write ข้อมูลลงใน Server ที่จะต้องทำให้ข้อมูลในทุก Server เหมือนกัน เหมือนกับระบบของธนาคาร

B) ถ้าเราเลือก Availability สิ่งที่เราต้อง concern คือการ Read ได้ตลอด แม้ว่า Server บางตัวจะล่มไป

---

## System Components

มีหลาย Concepts เลยที่ต้องทราบในการสร้าง Key-Value Store เช่น
- Data Partition
- Data Replication
- Consistency
- Inconsistency Resolution
- Handling Failures
- System Architecture Diagram
- Write Path
- Read Path
### Data Partition

![[database_partition_hash_ring.png]]

การจะทำ Data Partition ได้นั้น ไอเดียนึงที่ทำได้คือการทำเป็น Hash Ring โดยนำ Server และ Key มาเรียงกันเป็นวงกลม แล้วเมื่อไล่แต่ละ Key ตามเข็มนาฬิกาไปเจอกับ Server ใดที่อยู่ใกล้ที่สุดก็หมายความว่า Key นั้นจะอยู่ใน Server ดังกล่าว

วิธีนี้จะรองรับทั้งการทำ Auto Scaling และแต่ละ Server ก็จะเพิ่มโอกาสให้เก็บปริมาณ Key ได้เหมาะสมกับความสามารถของ Server
### Data Replication

![[database_replication_hash_ring.png]]

ด้วย Concept ของ Hash Ring เราสามารถกำหนดเพิ่มเติมได้ว่าแต่ละข้อมูลจะเก็บไว้ในกี่ Server สมมติว่าผมอยากกำหนดให้มี 1 Server เป็น Server หลัก และมี 2 Server เป็นตัวรอง (นั่นคือใช้ 3 Server) ก็สามารถใช้การไล่จากแต่ละ Key แบบตามเข็มนาฬิกา แล้วได้ว่า Server ตัวแรกที่เจอก็เป็น Server หลัก Server ที่สองและสามที่เจอก็เป็น Server รองที่เก็บ Key เหล่านั้นไว้

วิธีนี้ก็ทำให้เกิด High Availability ไปด้วย

ส่วนข้อดี และ Concept เพิ่มเติมอ่านได้ใน [[05 Database Replication|Database Replication]] 

### Consistency

![[consistency_hash_ring.png]]

ด้วยความที่เราใช้ Concept Hash Ring เลยเลี่ยงไม่ได้ที่จะต้องคิดถึงเรื่องการอ่าน และเขียนข้อมูลลงในแต่ละ Nodes อ่ะนะ โดยจะมีตัวแปรที่เกี่ยวข้องอยู่ 3 ตัว
- `N` คือจำนวน Replicas หรือจำนวน Nodes ที่เก็บ Key นึงไว้
- `W` คือจำนวน Replicas ที่เราเขียนข้อมูล Key เข้าไปสำเร็จ โดยเมื่อมีจำนวน Replicas ถึงค่า W เราถือว่าการเขียนข้อมูลนี้สำเร็จแล้ว
- `R` คือจำนวน Replicas ที่เราใช้เพื่ออ่านข้อมูล Key โดยเมื่อเราอ่านครบ R Replicas จะถือว่าการอ่านข้อมูลนี้สำเร็จ

โดยถ้า W + R > N มันจะการันตีได้เลยว่ามีอย่างน้อย 1 Server ที่มีข้อมูลในปัจจุบัน ตามหลัก Pigeon Hole Principle

แต่ว่าเราก็สามารถ Optimize ค่าเพิ่มเติมเพื่อให้เหมาะกับการใช้งานได้ เช่น
- R = 1, W = N -> เน้นการอ่านที่รวดเร็ว เพราะมันแปลว่าเราขอข้อมูลจาก Server เดียวพอ ถ้าได้มาแล้วก็ช่างแม่ง Response จาก Server อื่นได้เลย ทำให้การเขียนต้องมั่นใจจริงๆ ว่าเขียนครบทุกตัว
- W = 1, R = N -> เน้นการเขียนที่รวดเร็ว เพราะมันแปลว่าเราเขียนข้อมูลสำเร็จบน Server เดียวก็พอ แต่ว่าพอเราจะดึงข้อมูล มันไม่มีอะไรการันตีเลยว่าข้อมูลใน Server ไหนมันล่าสุด เลยต้องเช็คจากทุก Server อ่ะนะ

### Versioning

![[versioning.png]]

จบไปเรื่อง Hash Ring มาที่ประเด็นถัดไปอย่าง Versioning

มันก็มีโอกาสเป็นไปได้สูงที่จะเกิดเหตุการณ์ที่มีหลาย User แก้ไขข้อมูลพร้อมๆ กันแล้วระบบมันเลือกผลลัพธ์ไม่ถูกต้อง เช่น User A แก้ชื่อเป็น johnSanFrancisco ส่วน User B แก้ชื่อเป็น johnNewyork พอมันเป็นการแก้ที่ไปบนคนละ Server เลยกลายเป็นว่าถ้าไม่มีการเช็คก่อนว่าใครมาก่อนหลัง มันจะทำให้่มีฝั่งใดฝั่งนึงที่ได้ UX แย่ไป (ก็กูแก้เป็นชื่อนี้หนิ ทำไมเป็นชื่อนี้ไปได้วะ...)

เพราะงั้นเลยใช้ Concept ของ Vector Clock หรือการสร้างนาฬิกาขึ้นมานี่แหละ
โดยที่จะอยู่ในรูปแบบ (Si, vi) เมื่อ Si คือ Server ที่ i ส่วน vi คือ Version Counter : บ่งบอกว่ามีการแก้ไขข้อมูลบน Server i สำเร็จมาแล้วกี่ครั้ง

จะเป็นตาม Diagram นี้

![[versioning_vector_clock.png]]

ในกรณีที่เราคุยกันจะเป็นการที่เกิด Step 3 กับ Step 4 พร้อมกัน เราสามารถเขียน Algorithm ยังไงก็ได้ให้สามารถตัดสินได้ว่าจะเอาข้อมูลการแก้ของใครมาก่อน มาหลัง พอเราตัดสินได้แล้ว เราก็เอาแค่ข้อมูลสุดท้ายที่ solve conflicts ทั้งหมดแล้วเขียนเข้าไปใน Server หลักก็จบ (ตรงนี้ก็ถือเป็นจุดที่ต้องตัดสินใจดีๆ เหมือนกันว่าจะทำยังไงดี เช่น Last Write Win, Server Priority เป็นต้น)

### Failure Detection

อีกจุดนึงที่ต้องสนใจคือ เราจะรู้ได้ไงว่ามี Server / Nodes ที่พัง หรือ disconnect ไป วิธีนีงที่ทำได้คือให้แต่ละ Node ping หา Nodes อื่นๆ ที่เหลือทั้งหมด ก็ทำได้แหละ แต่ยิ่งพอมี Nodes เยอะเข้า มันจะเผาทรัพยากรแบบไม่จำเป็นอ่ะดิ

วิธีที่ดีกว่าคือ Gossip Protocal โดยจะทำงานประมาณนี้
- แต่ละ Node จะมี **Membership List** ซึ่งเก็บข้อมูลของ Nodes อื่น ๆ เช่น Node ID และ Heartbeat Counter
- แต่ละ Node จะเพิ่มค่า **Heartbeat Counter** ของตัวเองเป็นระยะ ๆ เพื่อบอกว่า “กูยังทำงานอยู่นะ”
- จากนั้นแต่ละ Node จะสุ่มเลือก Nodes จำนวนหนึ่ง แล้วส่งข้อมูล Heartbeat หรือ Membership Information ไปให้
- เมื่อ Node ได้รับข้อมูล ก็จะเอาไป Update Membership List ของตัวเอง และช่วยกระจายข้อมูลนี้ต่อไปยัง Nodes อื่น ๆ
- ถ้า Node ไหนไม่มี Heartbeat ใหม่เข้ามาภายในระยะเวลาที่กำหนด ระบบก็จะเริ่ม **Suspect** ว่า Node นั้นอาจจะมีปัญหา
- ถ้าผ่าน Timeout ไปแล้วก็ยังไม่มี Heartbeat ใหม่เข้ามาอีก จึงค่อย Mark Node นั้นว่า **Failed / Dead**

ยกตัวอย่างเช่น สมมติว่ามี Node A, B, C, D และ E
```
A → B, C
B → D
C → E
D → A
E → B
```

ถ้า Node A มี Heartbeat Counter เป็น `10` แล้วมันทำงานต่อและเพิ่มเป็น `11` ข้อมูลนี้ก็จะถูกส่งออกไปผ่าน Gossip เพื่อให้ Nodes B, C เรียนรู้ แล้วให้ B, C ไปส่งต่อให้ Nodes อื่นๆ ว่า Node A มี Counter = 11 ไรงี้

ทีนี้ถ้า Node A พังไป ก็จะไม่มี Heartbeat ใหม่จาก A เข้ามาอีก ระบบก็จะเริ่มสงสัยว่า Node A น่าจะมีปัญหา และหลังจากผ่าน Timeout ที่กำหนดไป แล้ว Node A ยังไม่มีการอัพเดท Counter ก็สามารถ Mark Node A ว่า Failed ได้

ข้อดีของวิธีนี้คือ **เราไม่จำเป็นต้องให้ทุก Node Ping หากันเองทั้งหมด** แต่ใช้วิธีให้ข้อมูลค่อย ๆ กระจายไปทั่ว Cluster ผ่าน Gossip ทำให้ลด Network Overhead ลง และสามารถรองรับระบบที่มี Nodes จำนวนมากได้ดีกว่า

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Key-Value Store ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
