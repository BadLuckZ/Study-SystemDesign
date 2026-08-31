---
tags:
  - system
  - systemdesign
  - chat_system
  - polling
  - websocket
  - stateful
  - stateless
  - fanout
  - heartbeat
  - key_value
---
วันนี้จะเป็น Chat System ละกัน เป็น 1 ใน Product ที่เป็นส่วนนึงในชีวิตประจำวัน ละก็สามารถสอดแทรกเข้าไปในระบบต่างๆ ได้ด้วย มาดูกันว่าจะเจอกับอะไรบ้าง

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Chat อันนี้จะ focus การคุยแบบไหนอ่ะครับ? 1 ต่อ 1 หรือคุยแบบเป็นกลุ่ม
A1: ควรมี และเน้นทั้งคู่นะ

Q2: ใช้งานบน Mobile, Web หรือว่าทั้งคู่เหรอครับ?
A2: ทั้งคู่ๆ

Q3: จะมีคนใช้งานเยอะขนาดไหนเหรอครับ?
A3: ราวๆ 50 Million DAU (Daily Active Users) ได้แหละ

Q4: Group Chat หรือคุยเป็นกลุ่มเนี่ย อยากจะจำกัดจำนวนคนในกลุ่มนั้นๆ มั้ยครับ
A4: จำกัดหน่อยละกัน สูงสุด 100 คนต่อกลุ่ม

Q5: จะมี Feature อะไรในการคุยกันบ้างเหรอครับ?
A5: ก็...มีการส่ง Message กัน ทั้งแบบเห็นทั้งกลุ่ม ละก็เห็น 1-1 อ่ะนะ ละก็มีสถานะบอกว่า User นี้ Online หรือ Offline แต่ว่าไม่ต้องส่ง File แนบละกัน มีแค่ Text Message ก็พอ

Q6: จะจำกัดจำนวนอักษรในแต่ละ Message มั้ยครับ?
A6: มีนะๆ เอาซัก 100000 ตัวอักษรละกัน

Q7: จะเก็บ Chat ไว้นานแค่ไหนเหรอครับ?
A7: เก็บไว้ตลอด ไม่มีทางลบออก มีแต่เพิ่ม

---

## 2. Sketch ภาพระบบคร่าวๆ

ก่อนอื่นเลย พื้นฐานของระบบนี้มี Feature อะไรบ้าง
1. User จะต้องรับ Message จาก User คนอื่นๆ ได้
2. User สามารถส่ง Message ได้ โดยที่ระบบจะต้องเก็บ Message เหล่านั้นไว้ให้ รอให้ User อีกฝั่งมาเห็น Message ที่ส่งมาได้เมื่อ User อีกฝั่ง online (ถ้า User อีกฝั่ง offline อยู่ก็ยังต้องมีการเก็บ Message นั้นไว้ให้จนกว่า User อีกฝั่งจะ online)

โดยการทำ Feature ตรงนี้ขึ้นมา technique หนึ่งที่ทำได้ คือการทำ Polling นั่นเอง
### Polling
![[chat_system_polling.png]]

Polling คือการที่ Client คอยเช็คถาม Server อย่างสม่ำเสมอว่ามี Message อะไรที่ Client นี้ต้องได้รับจาก Server บ้าง โดยจะเป็นการถามอัตโนมัติ ไม่ต้องให้ User มาเปิดเช็ค Application เอง ซึ่งข้อเสียก็ค่อนข้างจะชัดเจนอยู่แล้วนะ ก็คือ...
- Take Cost เยอะ โดยหมดไปกับการที่ต้องคอยเช็คอยู่เรื่อยๆ ทั้งๆ ที่การใช้งานจริงมันไม่จำเป็นต้องเรียกเช็คก็ได้ สำหรับ Chat ที่ไม่ค่อย active
- ต่อให้จะเพิ่ม delay การ polling เพื่อให้เพิ่มโอกาสเจอ Message แต่ว่า polling ก็ยังมาตายในเรื่องการเช็คว่า User online หรือ offline

เพราะงั้น เราก็จะขยับไปยังตัวที่นิยมกว่า อย่าง Websocket นั่นเอง
### Websocket
![[chat_system_websocket.png]]

Websocket เป็น Solution ยอดนิยมสำหรับปัญหา Asynchronous Update ระหว่าง Client กับ Server เลย โดยหลักการก็คือ WebSocket จะสร้าง **การเชื่อมต่อแบบสองทาง (เรียก Port ที่ไว้รับข้อมูลของแต่ละฝั่ง หรือจะบอกว่าปากท่อทั้งสองฝั่งก็ได้แหละ ว่า Socket) ระหว่าง Client และ Server ขึ้นมา** ทำให้ทั้งสองฝั่งสามารถส่งข้อมูลหากันได้ตลอดเวลาโดยไม่จำเป็นต้องรอ Request จากอีกฝั่ง

ถ้าจะขยายความ โดยปกติการสื่อสารผ่าน HTTP จะเป็นลักษณะ **Request-Response** คือ Client ต้องส่ง Request ไปก่อน แล้ว Server จึงตอบกลับมา แต่ WebSocket เมื่อสร้าง Connection สำเร็จแล้ว Connection จะถูกเปิดค้างไว้ ทำให้ **Server สามารถส่งข้อมูลไปยัง Client ได้ทันทีเมื่อมีข้อมูลใหม่ โดยไม่ต้องรอ Request จาก Client** จึงเหมาะกับระบบที่ต้องการ Real-time Update เช่น Chat และ Notification

เพราะงั้นเราก็จะมาใช้ Websocket กัน

### Architecture Design ในเบื้องต้น

นอกเหนือจาก Chat System แล้ว ส่วนอื่นๆ ก็ไม่จำเป็นต้องเป็น Websocket ก็ได้นะ เช่น Authentication, Group Management, Profile ไรงี้ เพราะว่ามันเป็น Stateless อ่ะ มีแค่ Chat นี่แหละที่ Stateful ละก็อาจจะมีการทำ Notification แบบ Push เพิ่มเติมให้ด้วยก็ได้ ด้วยการใช้ Third Party ตาม Platform ของ Client (อ่านเพิ่มเติมได้ใน [[06 Notification System|Notification System]])

>[!Note] Stateful VS Stateless
> - **Stateful** คือ Server ต้องจดจำสถานะหรือข้อมูลของ Client ระหว่างการติดต่อกัน จึงจะสื่อสารกันได้
> - **Stateless** คือ Server ไม่จำเป็นต้องจดจำสถานะของ Client ก็ได้ แต่ละ Request สามารถประมวลผลได้อย่างอิสระ
> 
> สามารถอ่านเพิ่มเติมได้ใน [[08 Stateful VS Stateless|Stateful VS Stateless]] 

เพราะงั้น ภาพระบบใหญ่ของเราจะเป็นประมาณนี้
![[chat_system_design_v1.png]]
 
โดยที่...
- Chat Server จะจัดการเรื่องการรับ และส่ง Messages
- Presence Server จะจัดการเรื่อง Online และ Offline Status
- API Server ก็รวม Services ต่างๆ ที่ระบบนี้มี เช่น Signup / Login, Profile, สร้าง Group เป็นต้น
- Notification Server จัดการเรื่องการส่ง Notification ในรูปแบบต่างๆ ให้เหมาะสมกับ Devices
- KV Store หรือ Key-Value Store ทำหน้าที่เป็น Database (ส่วนที่ใช้เป็น Key-Value Store เพราะว่า **ส่วนใหญ่เป็นข้อมูลแบบ Key-Value และต้องการการ Read/Write ที่รวดเร็ว** เช่น `userId → socketId` (User คนนี้ใช้ Socket ID อะไร), `userId → online/offline` (User คนนี้ online หรือ offline อยู่) เป็นต้น)

### Schema ในเบื้องต้น

หลักๆ เลยก็จะเป็น Message Schema เนอะ ที่จะแบ่งออกเป็น 2 ชุด คือ Message สำหรับคุยส่วนตัว (Message) กับ Message สำหรับคุยเป็นกลุ่ม (Message_Group) คร่าวๆ ของทั้ง 2 Schema ก็จะออกมาเป็นประมาณนี้

![[chat_system_message_schema.png]]

ที่น่าสนใจคือ เราจะจัดการ message_id ยังไงให้มัน unique กันนะ...

วิธีการนึงที่ทำได้ คือการใช้ Snowflake (เนื้อหาใน [[02 Unique ID Generator|Unique ID Generator]]) ซึ่งจะได้ ID ที่ไม่ซ้ำกัน ละก็สามารถเรียงข้อมูลตามเวลาที่เกิดการสร้าง Message ได้

แต่แบบนั้นก็อาจจะยังยากไป และอาจจะยังไม่พอกับ case นี้ ในเมื่อเรามี `channel_id` มาแล้ว เราก็ใช้ `channel_id` ให้เป็นประโยชน์ คือทำให้ Primary Key ของ Message Group เป็น `(channel_id, message_id)` ซึ่งจะทำให้แม้ว่า message_id จะซ้ำกัน แต่ว่าถ้าอยู่คนละ channel_id ก็ถือว่ายังใช้ได้นั่นเอง ช่วยให้การสร้าง id ให้ message_id ทำได้ง่ายขึ้นเยอะ

---

## 3. เจาะลึกจุดสำคัญๆ

จาก Architecture Design ด้านบน มีอยู่ 3 จุดที่น่าเจาะลึกเป็นพิเศษ คือ **Service Discovery**, **Message Flow** และ **Online/Offline Indicator**
### Service Discovery

หน้าที่หลักของ Service Discovery คือการแนะนำว่า Client คนนี้ควรต่อกับ Chat Server ตัวไหนดี ถึงจะทำให้การใช้งานเป็นไปอย่างมีประสิทธิภาพ โดยพิจารณาจาก Criteria ต่างๆ เช่น ตำแหน่งของ Client, Capacity ที่เหลือของแต่ละ Chat Server ตอนนั้น เป็นต้น

เครื่องมือยอดนิยมสำหรับงานนี้คือ **Apache Zookeeper** ที่คอยเก็บ List ของ Chat Server ที่พร้อมใช้งานทั้งหมดไว้ แล้วเลือก Server ที่เหมาะสมที่สุดให้ Client แต่ละคนตาม Criteria ที่ตั้งไว้

Flow คร่าวๆ ก็จะเป็นแบบนี้:
1. User A พยายาม Login เข้าแอป
2. Load Balancer ส่ง Request Login ไปให้ API Server
3. หลังจาก authenticate User สำเร็จแล้ว Service Discovery จะหา Chat Server ที่ดีที่สุดให้ User A (สมมติในตัวอย่างนี้เลือก Server 2) แล้วส่งข้อมูล Server กลับไปให้ User A
4. User A เชื่อมต่อไปยัง Chat Server 2 ผ่าน WebSocket

![[chat_system_service_discovery.png]]

### Message Flow

Flow ของ Message จะแบ่งออกเป็น 3 Case คือ 1-1, การ Sync Message ข้ามหลาย Device และ Group Chat
#### 1-1 Chat Flow
![[chat_system_1-1_flow.png]]

เวลา User A ส่ง Message หา User B จะมีขั้นตอนประมาณนี้:
1. User A ส่ง Message ไปที่ Chat Server 1 (ตัวที่ตัวเองเชื่อมต่ออยู่)
2. Chat Server 1 ไปขอ Message ID ที่ Unique จาก **ID Generator** (เช่นใช้ Snowflake ตามที่พูดถึงไปก่อนหน้า)
3. Chat Server 1 ส่ง Message เข้า **Message Sync Queue**
4. Message ถูกเก็บลง **KV Store** เพื่อเก็บถาวร
5. แยกเป็น 2 เคสตามสถานะของ User B:
    - **5.a) ถ้า User B Online** - Message ถูกส่งต่อไปยัง Chat Server ที่ User B เชื่อมต่ออยู่ (เช่น Chat Server 2) ทันที
    - **5.b) ถ้า User B Offline** - ระบบจะยิง Push Notification ผ่าน **PN (Push Notification) Server** แทน เพื่อแจ้งเตือน User B แม้จะไม่ได้เปิดแอปอยู่
6. Chat Server 2 ส่ง Message ต่อไปให้ User B ผ่าน WebSocket Connection ที่เปิดค้างไว้อยู่แล้ว

#### Message Sync ข้าม Device
มี User หลายคนที่ใช้มากกว่า 1 Device (เช่นทั้งมือถือและ Laptop) คนเขียนก็ด้วยเหมือนกัน XD แล้วเราจะ Sync Message ให้ตรงกันทุก Device ได้ยังไง?
![[chat_system_multidevices.png]]

ข้อมูลที่มี คือแต่ละ Device จะมีตัวแปรที่เรียกว่า **`cur_max_message_id`** ที่ไว้จำว่า Message ล่าสุดที่ Device นั้นเห็นคือ ID อะไร แล้ว Message ที่จะถือว่าเป็น "Message ใหม่" ที่ต้อง Sync ให้ Device นั้นเห็น ต้องเข้าเงื่อนไข 2 ข้อพร้อมกัน
- Recipient ID ตรงกับ User ที่ Login อยู่บน Device นั้น
- Message ID ใน KV Store มีค่า**มากกว่า** `cur_max_message_id` ของ Device นั้น

เพราะแต่ละ Device เก็บค่า `cur_max_message_id` แยกกันเอง การ Sync เลยทำได้ง่ายมาก ก็แค่ให้แต่ละ Device ไปขอ Message ใหม่ๆ จาก KV Store ที่มี ID มากกว่าค่าที่ตัวเองจำไว้เท่านั้นเอง ไม่ต้องมาคอยเทียบกันข้าม Device ให้ซับซ้อน
#### Small Group Chat Flow
เราจะแบ่ง Flow Message ใน Group เป็น 2 ช่วง คือช่วงการส่ง Message และช่วงการรับ Message
##### Sending Messages
![[chat_system_send_message.png]]

Flow การส่ง คือ Message จาก User A จะถูก **Copy** ไปไว้ใน Message Sync Queue ของสมาชิกทุกคนในกลุ่ม ในที่นี้ก็คือ Message Sync Queue ของ User B และ User C

> กรณีที่มีสมาชิกในกลุ่มเยอะ สามารถเก็บ Message แค่ชุดเดียวต่อกลุ่ม โดยผูกไว้กับ `channel_id` ของกลุ่มนั้นแทนที่จะแจกให้ User ทุกคน แล้วให้ Client แต่ละคน**ไป Pull ข้อความจาก Channel กลางนั้นเอาเอง** โดยอิง `cur_max_message_id` ของ Device ของ User มาเช็คว่าอ่านมาถึงไหนแล้ว วิธีนี้จะคล้ายกับแนวคิด **Fanout Read** ที่เคยพูดถึงใน [[07 News Feed System|News Feed System]] เลย ความหมายคือถ้าสมาชิกเยอะ ก็สร้างกองกลางให้สมาชิกแต่ละคนดึงไปเช็คเอาเอง ถึงแต่ละคนจะใช้เวลาเพิ่มนิดหน่อย แต่ในระบบใหญ่ก็เร็วกว่าการแจกจ่ายให้ถึงมือทุกคนที่มีจำนวนเยอะมาก
##### Receive Messages
![[chat_system_receive_message.png]]

ด้วยการออกแบบในลักษณะนี้ Flow ของการรับ Message เลยเป็นแค่การเปิด Message Sync Queue ของตัวเองดู เพื่อดึงข้อความที่ใหม่กว่าที่ Device มีในปัจจุบันมาทั้งหมดแทน

### Online/Offline Indicator

มันคึอไอ่จุดเขียวๆ ข้างชื่อ/รูปโปรไฟล์ที่บอกว่า User online อยู่หรือเปล่า โดย **Presence Server** จะเป็นตัวที่รับผิดชอบเรื่องนี้ คอยจัดการสถานะ Online/Offline และสื่อสารกับ Client ผ่าน WebSocket
มีเหตุการณ์หลักๆ ที่ทำให้ Status เปลี่ยนอยู่ 4 แบบ:

![[chat_system_presence_login.png]]

**1. User Login** - พอ WebSocket Connection ถูกสร้างสำเร็จ (เกิดตอนที่ User เข้าใช้งาน Application สำเร็จ) สถานะ Online ของ User พร้อม Timestamp ล่าสุด (`last_active_at`) จะถูกบันทึกลง KV Store แล้ว Indicator ก็จะโชว์ว่า User คนนั้น Online

![[chat_system_presence_logout.png]]

**2. User Logout** - User กด Logout -> Request วิ่งผ่าน API Server → API Server ส่งข้อมูลให้ Presence Server → update สถานะของ User A ใน KV Store เป็น Offline

![[chat_system_heartbeat.png]]

**3. User Disconnect** จะด้วยเน็ตหลุดดี หรือปัญหาจากตัวเครื่องดี ถ้าระบบเรามันดีเกินว่า detect การเปลี่ยนแปลงได้ไวมากๆ ก็อาจจะได้เห็นการ online / offline รัวๆ เกิดเป็น Status ที่กระพริบไปมาเลย

วิธีแก้คือ **Heartbeat Mechanism** - Client ที่ Online จะส่ง Event ไปหา Presence Server เป็นระยะๆ (เช่นทุก 5 วินาที) ถ้า Presence Server ได้รับ Heartbeat ภายในเวลาที่กำหนด (เช่น 30 วินาที) ก็ยังถือว่า User Online อยู่ แต่ถ้าไม่ได้รับ Heartbeat เลยเกินเวลาที่กำหนด ถึงจะเปลี่ยนสถานะเป็น Offline

แปลไทยง่ายๆ ว่าทุกๆ 30 วินาทีจะมา update Status แหละ และจะมีเวลาส่งข้อมูลว่า online อยู่นะ 5 วินาที ถ้าส่งทันก็นับใหม่ 30 วินาที วนงี้ไปเรื่อยๆ จนถ้าส่งไม่ทัน / ไม่ส่งแล้วก็ถือว่า offline

![[chat_system_update_user_status.png]]

**4. แจ้ง Online Status ให้เพื่อนรู้** ผ่านการทำให้ Presence Server ใช้โมเดล **Publish-Subscribe** โดยแต่ละคู่เพื่อนจะมี Channel ของตัวเอง เช่น Channel A-B, A-C, A-D พอสถานะของ User A เปลี่ยน ระบบจะ Publish Event เข้าไปทั้ง 3 Channel นี้ แล้วเพื่อนแต่ละคน (B, C, D) ที่ Subscribe Channel ของตัวเองไว้ ก็จะได้รับ Update ผ่าน WebSocket ทันที

ซึ่งก็จะเห็นว่าภาพนี้จะคล้ายๆ Fanout Write ที่แจกจ่ายข้อมูลให้ครบทุกคน มันก็เหมาะกับคนที่มี Friend List น้อยๆ หรือ Group Chat ขนาดเล็กแหละ (Concept เหมือนที่เคยพูดถึงใน [[07 News Feed System|News Feed System]])

แต่พอ Friend List เยอะๆ หรือ Group Chat ขนาดใหญ่ก็จะกลายเป็นภาพ Fanout Read แทน คือไม่ Push Status ให้ User ทุกคนแล้ว แต่ให้ User แต่ละคน**ดึง Status เมื่อ User เข้ากลุ่มหรือกด Refresh Friend List เอง** แลกกับ Real-time ที่ลดลงบ้าง แต่ก็ยังทันใช้งานจริง

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Chat System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---