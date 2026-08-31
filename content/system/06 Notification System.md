---
tags:
  - system
  - systemdesign
  - notification_system
  - push_notification
  - sms
  - email
  - message_queue
  - rate_limiting
  - cac
  - cache
---
Notification System เข้ามามีบทบาทสำคัญขึ้นในหลายๆ Applications ไว้สำหรับแจ้งเตือนข้อมูลต่างๆ ให้ User เช่น ข่าวสาร, โปรโมชันต่างๆ ซึ่งเราจะ implement ยังไง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: จะมี notification แบบไหนบ้างอ่ะ
A1: Push Notification, SMS, ละก็ Email

Q2: ระบบนี้จะเป็น real-time มั้ยนะ
A2: ก็ควรนะ...คือเราอยากให้ user ได้รับ notification เร็วที่สุดเท่าที่จะเป็นไปได้อ่ะ แต่ว่าถ้า load มันเยอะ จะ delay นิดหน่อยก็ได้อยู่ เข้าใจได้

Q3: ระบบนี้จะใช้งานบน devices ไหนได้บ้าง
A3: iOS, Android, Laptop ละก็ Desktop

Q4: User สามารถยกเลิกการได้รับ notification ได้มั้ย
A4: ได้นะ ถ้าไม่ต้องการแล้วก็ยกเลิกได้

Q5: จะมีการส่ง notification มากแค่ไหนใน 1 วัน
A5: 10 ล้าน push, 1 ล้าน SMS, ละก็ 5 ล้าน emails ละกัน

---

## 2. Sketch ภาพระบบคร่าวๆ

#### Third Party Service
ก่อนอื่นเลย ระบบของเรามี notification ถึง 3 แบบ คือ push notification, sms และ email ซึ่งแต่ละแบบจะต้องใช้ Third Party Service สำหรับจัดการ notification แต่ละประเภทที่ต่างกัน เช่น

- iOS Push จะใช้ APNs (Apple Push Notification service)
- Android Push จะใช้ FCM (Firebase Cloud Messenging)
- SMS จะใช้ SMS Service เช่น Twillo, Nexmo etc.
- Email จะใช้ Email Service เช่น Sendgrid, Resend etc.

ซึ่งจะออกมาเป็นตามภาพนี้
![[notification_design_thirdparty_to_user.png]]

---
#### Notification Service
เราจะสร้างข้อมูลเพื่อให้ trigger การส่ง Notification ได้ยังไง...
สิ่งที่ต้องทำคือ การสร้าง system / service ที่รับข้อมูลจาก services ต่างๆ แล้วส่งให้ third party เหล่านี้ส่งเข้า devices ของ user ตาม design นี้
![[notification_design_service_to_user_v1.png]]

แต่ว่าด้วย Design แบบนี้ มันดันมี trade-off อยู่ คือ
- เกิด Single Point of Failure ที่ Notification System
- Scale ได้ยาก เพราะ Notification System ทำหมดทุกอย่าง

การแก้ไขนั้น ทำได้หลายอย่าง เช่น
- ทำ Notification System เป็น Notification Servers หลายๆ ตัว เชื่อมกับ Database ไปด้วย
- ใช้ Message Queue สำหรับเก็บข้อมูลที่ Notification Servers ส่งออกมา โดยที่ใช้ Message Queue แยกตามประเภทของ Notification ละมี Workers หลายๆ ตัวสำหรับส่ง Notification ได้แบบ Parallel และหากมีปัญหาก็เอา Notification ที่เกิดปัญหานั้นมาเก็บใน Message Queue ใหม่
ก็จะออกมาเป็นตาม design นี้
![[notification_design_service_to_user_v2.png]]

ซึ่งพอเราทำเป็น Notification Servers แล้ว หมายความว่า
- มีการทำ API เพื่อส่ง Notification โดยจะอนุญาตให้เฉพาะ verified clients แล้วเท่านั้น และมีการจำกัด rate การส่งไม่ให้มากจนเกินไป เพื่อเลี่ยงการเป็น spam
- มีการทำ Validation เพิ่มเติม เช่น เช็ค email, เช็ค phone number เป็นต้น

ตัวอย่าง API ก็เช่น
```swagger
POST /api/v1/sms/send
	request parameter: {
		to: [
			{"user_id": 123456},
			{...},
			...
		],
		from: {
			email: "example@gmail.com"
		},
		subject: string,
		content: [
			type: "text/plain",
			value: "Hello World!"
		]
	}
```

ละ Flow ของระบบก็จะเป็นแบบนี้
1. Services ต่างๆ มีการเรียกใช้ API จาก Notification Servers แล้วแนบข้อมูลมาให้ เพื่อส่งข้อมูลสำหรับสร้าง Notification
2. Notification Servers ดึงข้อมูลต่างๆ ของ User, ข้อมูล Device ของ User จาก Database และ Cache ตาม parameter ที่ services ต่างๆ ส่งเข้ามา
3. Notification Servers ส่ง Notification เก็บไว้ใน Message Queue ตามข้อมูล Device ของ User
4. Workers ของ Device แต่ละประเภทดึง Notification ออกมาจาก Message Queue
5. Workers ของ Device แต่ละประเภทส่ง Notification ไปให้ Third Party Service
6. Third Party Service ส่ง Notification ไปให้ User Devices

แล้วระบบเราจะรู้ได้ไงว่าจะต้องส่ง notification ให้ใคร และจะจำแนกได้ไงว่า user คนนี้คือ device อะไร เราสามารถทำได้ในตอนที่ user ใช้งาน app ของเราในครั้งแรก เราสามารถเก็บข้อมูลจาก user แล้วเก็บข้อมูลใส่ table ที่ชื่อ device ได้ โดยใน table device นี้ก็สามารถระบุได้เลยว่าเป็น device ประเภทไหน ตามภาพ
![[notification_design_user_input.png]]

---

## 3. เจาะลึกจุดสำคัญๆ

จาก Design ล่าสุดของเรา มี concern ที่จำเป็นต้องคิดเพิ่มเติมด้วย
### 1. Reliability
เราจะทำยังไงให้การส่ง Notification มันส่งครบทุกคน ไม่มีใครบางคนที่ไม่ได้รับ Notification?
คือ Notification สามารถถูก delay หรือสลับ order กันได้ แต่มันจะต้องส่งให้ user ได้รับครบทุกคน ไม่มี notification ของใครที่หายไป

วิธีการนึงที่ทำได้ คือ มี Database ที่เป็น Notification Log แล้วคอยเช็คอยู่เป็นระยะๆ ว่ายังมี Notification ใดบ้างที่ยังไม่ได้ส่ง / ส่งไม่สำเร็จ ตาม design นี้
![[notification_design_reliability.png]]

แต่ว่าในโลกความจริง ก็อาจเกิดเหตุการณ์ที่ Notification เดิมถูกส่งไปหา Device ของ User ซ้ำ ๆ ได้ เช่น **ในกรณีที่ระบบเกิดการ Retry เนื่องจากไม่สามารถยืนยันได้ว่าการส่งครั้งก่อนสำเร็จหรือไม่** 
เพื่อหลีกเลี่ยงการส่ง Notification ซ้ำ จึงควรเพิ่ม Algorithm สำหรับจัดการ Duplicate Notification ด้วย

> หลักการง่ายๆ คือ เมื่อมี Notification เข้ามาใน Message Queue เราจะตรวจสอบก่อนว่า Notification ID นี้เคยถูกประมวลผลแล้วหรือไม่ผ่าน Notification Log หากเคยแล้วจะทิ้ง Notification นั้นไป แต่หากยังไม่เคย เราจะส่ง Notification ออกไป

### 2. Component เพิ่มเติมที่สามารถใส่ได้

#### 1. Notification Template
ตามชื่อเลย กำหนด Template ว่าจะส่งข้อความอะไรให้ User ผ่าน Notification โดยสิ่งที่จะต้องเพิ่มเติมเลยจะมีแค่ข้อมูลบางอย่างที่เฉพาะเจาะจงกับ User นั้นๆ เช่น

> สวัสดี *{{user_name}}* คุณมี *{{notification_count}}* รายการใหม่ที่รอดำเนินการ

#### 2. Notification Setting
User มักจะได้รับ Notification จำนวนมากในแต่ละวัน ทำให้ User อาจจะรำคาญได้ (รำคาญแหละ หลายแอป หลาย Notification อ่ะ) ดังนั้นเราควรเปิดให้ User เลือกได้ว่าต้องการรับ Notification ประเภทไหนได้บ้าง เช่น Push Notification, Email หรือ SMS
![[notification_design_notification_setting_table.png]]

ก่อนที่จะส่ง Notification ให้ User เราจะตรวจสอบก่อนว่า User คนนั้นเปิดรับ Notification ผ่าน Channel นั้นอยู่หรือไม่ หาก User ไม่ได้ Opt-in ก็จะไม่ส่ง Notification นั้นออกไป

#### 3. Rate Limiting
นอกจากการให้ User สามารถปิด Notification ได้แล้ว เราควรมีการจำกัดจำนวน Notification ที่ User สามารถได้รับภายในช่วงเวลาหนึ่งด้วย เพื่อป้องกันไม่ให้ User ได้รับ Notification มากจนเกินไป เพราะถ้าระบบส่ง Notification ให้ User บ่อยเกินไป User อาจรู้สึกรำคาญและเลือกปิด Notification ทั้งหมดแทน

#### 4. Retry Mechanism
หาก Third Party Service ไม่สามารถส่ง Notification ได้ เช่น Service ล่ม หรือเกิด Network Error เราจะต้องไม่ทิ้ง Notification นั้นไป ไม่งั้นจะเกิด Loss Data ได้ แต่จะนำ Notification กลับเข้าไปใน Message Queue เพื่อทำการ Retry อีกครั้ง

เราอาจกำหนดจำนวนครั้งในการ Retry และใช้ Exponential Backoff เพื่อไม่ให้ระบบส่ง Request ไปยัง Third Party Service ที่กำลังมีปัญหาถี่จนเกินไป หาก Retry ครบจำนวนที่กำหนดแล้วยังไม่สำเร็จ เราสามารถส่ง Alert ไปให้ Developer หรือทีมที่เกี่ยวข้องเข้ามาตรวจสอบได้

#### 5. Security in Push Notifications
สำหรับ iOS และ Android เราจะต้องมี Authentication และ Authorization เพื่อป้องกันไม่ให้ Client ที่ไม่ได้รับอนุญาตสามารถเรียกใช้ Notification API ของเราได้

ดังนั้น API สำหรับส่ง Notification ควรอนุญาตเฉพาะ **Authenticated / Verified Clients** เท่านั้น และอาจใช้ `appKey` และ `appSecret` หรือกลไกอื่นที่เหมาะสมในการยืนยันตัวตนของ Client

#### 6. Monitor Queued Notifications
อีก Metric ที่สำคัญที่ควร Monitor คือจำนวน Notification ที่อยู่ใน Message Queue เพราะหากจำนวน Notification ใน Queue เพิ่มขึ้นเรื่อย ๆ แสดงว่า Workers ไม่สามารถประมวลผล Notification ได้เร็วพอกับจำนวน Notification ที่เข้ามา ซึ่งอาจทำให้เกิด Delay ในการส่ง Notification ได้

ดังนั้นเราสามารถใช้ **Queue Depth** เป็นตัวชี้วัด และหากพบว่าจำนวน Notification ใน Queue สูงเกินไป เราสามารถเพิ่มจำนวน Workers เพื่อช่วยประมวลผล Notification แบบ Parallel ได้
![[notification_message_queue_counter.png]]

#### 7. Notification Tracking

สุดท้าย เราสามารถเก็บข้อมูลเกี่ยวกับการใช้งาน Notification เพื่อวิเคราะห์พฤติกรรมของ User ได้ เช่น **Open Rate, Click Rate และ Engagement**

ข้อมูลเหล่านี้สามารถส่งไปยัง Analytics Service เพื่อวิเคราะห์ว่า Notification ประเภทไหนหรือ Channel ไหนมีประสิทธิภาพมากที่สุด และนำข้อมูลเหล่านี้ไปใช้ปรับปรุง Notification Strategy ในอนาคตได้

ตัวอย่าง Event ที่สามารถ Track ได้ เช่น
- Notification Sent
- Notification Delivered
- Notification Opened
- Notification Clicked
- Notification Dismissed

เพราะงั้น Design ล่าสุดก็จะเป็นแบบนี้
![[notification_design_service_to_user_v3.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Notification System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
