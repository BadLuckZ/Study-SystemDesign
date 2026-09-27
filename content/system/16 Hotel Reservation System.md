---
title: 16 Hotel Reservation System
tags:
  - system
  - systemdesign
  - hotel_reservation_system
  - database
  - microservices
  - idempotency
  - concurrency
  - locking
  - versioning
  - sharding
  - cache
---
วันนี้เราจะมา Design ระบบของการจองห้องของโรงแรมกัน ก็ตรงตัวแหละ 555 คือจะมี List โรงแรมให้ User เลือก ละก็แต่ละโรงแรมก็จะมี Package ต่างๆ รวมถึงห้องให้ User ได้เลือกใช้บริการ จะเป็นไง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Scale ของระบบนี้นี่ประมาณไหนเหรอครับ
A1: คิดภาพว่าเป็นระบบที่มีโรงแรมอยู่ 5000 โรงแรม ละก็มีห้องรวม 1 ล้านห้องละกัน

Q2: แล้วเราจองห้องสำเร็จจะต้องจ่ายเงินเลยมั้ย หรือว่าให้ไปถึงโรงแรมก่อนค่อยจ่าย
A2: เอาเป็นว่า...จ่ายเงินตั้งแต่ตอนจองเลยละกัน

Q3: เราสามารถยกเลิกการจองห้องได้มั้ยครับ
A3: ยกเลิกได้นะ

Q4: แล้ว...มีเรื่องอะไรที่ต้องกังวลอีกมั้ยครับ
A4: อ่า... ก็มีอีกนะ
1. เรื่องการจองห้องเกินละกัน แบบว่า...ระบบควรจะอนุญาตให้โรงแรมสามารถขายห้องเกินกว่าจำนวนห้องที่โรงแรมมีได้ หรือก็คือมีลำดับสำรองนั่นแหละ เผื่อว่าจะมี User บางคนยกเลิกการจองของตัวเองไป โดยสำรองไว้ได้ซัก 10% ของห้องที่โรงแรมมีละกัน
2. ราคาแต่ละห้องของโรงแรมสามารถเปลี่ยนแปลงไปได้ในแต่ละวันด้วย

### Feature Requirement

- ระบบสามารถแสดงรายละเอียดของโรงแรมได้
- ระบบสามารถแสดงรายละเอียดของห้องต่างๆ ที่โรงแรมมีได้
- User สามารถใช้ระบบเพื่อจองห้องได้
- Hotel Staff ของโรงแรมสามารถเพิ่ม / แก้ไข / ลบโรงแรม รวมถึงห้องที่โรงแรมนั้นมีได้
- ระบบสามารถเปิดให้มีการจองเกินกว่าที่โรงแรมสามารถรับได้ได้

### Non-Functional Requirement

- High Concurrency: รองรับการใช้งานในช่วงที่มีคนใช้งานเยอะๆ ได้ เช่นในช่วงงานเทศกาลต่างๆ
- Moderate Latency: สามารถใช้เวลาไปประมาณนึงเพื่อจองห้องของโรงแรมได้ แต่ถ้าแทบไม่ใช้เวลาได้เลยก็ดี

---

## 2. Sketch ภาพระบบคร่าวๆ

### API Design

API ก็จะแบ่งเป็น 3 ส่วนหลักๆ คือส่วนของโรงแรม, ห้องของโรงแรม, ละก็การจองห้อง
#### 1. Hotel-related APIs
| API                       | Detail                                                     |
| ------------------------- | ---------------------------------------------------------- |
| **GET** /v1/hotels/:id    | แสดงรายละเอียดของโรงแรม                                    |
| **POST** /v1/hotels       | สร้างข้อมูลของโรงแรมใหม่ขึ้นมา สำหรับ Hotel Staff เท่านั้น |
| **PUT** /v1/hotels/:id    | แก้ไขข้อมูลของโรงแรม สำหรับ Hotel Staff เท่านั้น           |
| **DELETE** /v1/hotels/:id | ลบข้อมูลของโรงแรม สำหรับ Hotel Staff เท่านั้น              |
#### 2. Room-related APIs
| API                                            | Detail                                                           |
| ---------------------------------------------- | ---------------------------------------------------------------- |
| **GET** /v1/hotels/:hotel_id/rooms/:room_id    | แสดงรายละเอียดของห้องในโรงแรม                                    |
| **POST** /v1/hotels/:hotel_id/rooms            | สร้างข้อมูลของห้องใหม่ในโรงแรมขึ้นมา สำหรับ Hotel Staff เท่านั้น |
| **PUT** /v1/hotels/:hotel_id/rooms/:room_id    | แก้ไขข้อมูลของห้องใหม่ในโรงแรมขึ้นมา สำหรับ Hotel Staff เท่านั้น |
| **DELETE** /v1/hotels/:hotel_id/rooms/:room_id | ลบข้อมูลของห้องใหม่ในโรงแรมขึ้นมา สำหรับ Hotel Staff เท่านั้น    |
#### 3. Reservation-related APIs
| API                             | Detail                               |
| ------------------------------- | ------------------------------------ |
| **GET** /v1/reservations        | แสดงประวัติการจองห้องทั้งหมดของ User |
| **GET** /v1/reservations/:id    | แสดงรายละเอียดของการจองห้องของ User  |
| **POST** /v1/reservations       | สร้างการจองห้องใหม่                  |
| **DELETE** /v1/reservations/:id | ยกเลิกการจองห้อง                     |
Note: ใน Request ของการจองห้อง จะมีหน้าตาประมาณนี้

```swagger
{
	startDate: 2021-04-08
	endDate: 2021-04-30
	hotelID: 123
	roomID: U12312
	reservationID: 123556435
}
```

โดยจะมี reservationID ติดไปด้วย เพื่อทำหน้าที่เป็น Idempotency Key ป้องกันการจองห้องซ้ำ 2 รอบ

### Data Model

Schema ก็ค่อนข้างตรงตัวนะ โดยที่เราจะเลือกใช้ Relational Database เพราะ Relationshipp มีความชัดเจน รวมถึงการมี ACID Properties สำหรับจัดการปัญหาต่างๆ เช่น เงินติดลบ, การจองหลายรอบ เป็นต้น (รายละเอียดเพิ่มเติมอ่านได้ใน [[01 Database|Database]])

![[hotel_reservation_schema_v1.png]]

เราจะทำระบบของเราแบบ Microservices เพื่อรองรับการ scale ได้เวลามีการใช้งานในช่วง Peak
- Hotel Service ทำหน้าที่เกี่ยวกับโรงแรม รวมถึงห้องของโรงแรม
- Rate Service ทำหน้าที่เกี่ยวกับราคา ละก็การชำระเงิน
- Reservation Service ทำหน้าที่เกี่ยวกับการจองห้องในโรงแรม
- Guest Service ทำหน้าที่เกี่ยวกับข้อมูลของ User

เพิ่มเติมคือ... Status ของ Reservation จะประกอบด้วย... PENDING, PAID, REFUNDED, CANCELED, REJECTED

แต่ว่า Data Model แบบนี้จะยังมีปัญหากับห้องที่สามารถเพิ่ม Add on ได้ เช่น ขอเพิ่มเตียง หรือขอเตียงที่ขนาดใหญ่ขึ้น หรือ Option อื่นๆ ที่สามารถเพิ่มเติมได้ ซึ่งจะไปพูดต่อใน Section ถัดๆ ไป

### High-Level Design

หน้าตา Design จริงๆ ก็ตรงตัวนะ ตาม Schema แหละ

![[hotel_reservation_high-level_design.png]]

ซึ่งมันจะมี Hotel Management Service เพิ่มเติมขึ้นมาสำหรับให้ Admin / Hotel Staff จัดการข้อมูลของโรงแรม รวมถึงเรื่องต่างๆ ที่เกี่ยวข้อง เพราะงั้นถ้ามองในแง่ Microservice มันก็จะมีความสัมพันธ์กันด้วย ตามภาพนี้เลย

![[้hotel_reservation_microservices.png]]

เผื่อว่างงๆ ว่า Microservice คืออะไร อ่านต่อได้ใน [[10 Microservices|Microservices]]

---

## 3. เจาะลึกส่วนที่สำคัญๆ

ก่อนหน้านี้เรามีปัญหาเรื่องการ Add On เนอะ แล้วเราจะแก้อะไรยังไงได้บ้าง มาดูกัน
### Improved Data Model

จากเดิมที่ **POST** /v1/reservations ที่มีหน้าตาประมาณนี้

```swagger
{
	startDate: 2021-04-08
	endDate: 2021-04-30
	hotelID: 123
	roomID: U12312
	reservationID: 123556435
}
```

เราจะแก้จาก roomID เป็น roomTypeID ละก็เพิ่ม roomCount เข้ามาด้วย โดย roomTypeID ก็คือเป็นตัวบ่งบอกประเภทของห้องๆ นั้น เช่น Standard, Suite, Luxury หรือประเภทอื่นๆ ตามแต่ที่จะมี เลยทำให้ Request ใหม่มีหน้าตาประมาณนี้

จากเดิมที่ **POST** /v1/reservations ที่มีหน้าตาประมาณนี้

```swagger
{
	startDate: 2021-04-08
	endDate: 2021-04-30
	hotelID: 123
	roomTypeID: 1232342
	roomCount: 1
	reservationID: 123556435
}
```

ทีนี้เราก็ต้องบอกเพิ่มเติมด้วยว่าแต่ละห้องเป็นห้องประเภทไหน ทำให้ Schema ใหม่ก็จะมีหน้าตาประมาณนี้

![[้hotel_reservation_schema_v2.png]]

โดยจะเพิ่ม room_type_inventory เข้าไปใน reservation เพื่อบ่งบอกว่าห้องแต่ละประเภทของแต่ละโรงแรมและในแต่ละวัน มีอยู่กี่ห้อง และถูกจองไปแล้วเท่าไหร่ เพื่อเป็น info สำหรับจัดการเรื่องการจองนั่นเอง

![[hotel_reservation_room-type-inventory_mock_data.png]]

การมี room_type_inventory จะช่วยทำให้ระบบสามารถเช็คได้ว่าในช่วงเวลาที่ User ต้องการใช้ห้องประเภทนี้ของโรงแรมนี้ จะมีห้องเหลืออยู่หรือไม่ ผ่าน 2 จังหวะ
1. ใช้ Query นี้เพื่อดึง row ใน Range ที่ User ต้องการจะใช้ห้อง
```sql
	SELECT date, total_inventory, total_reserved
	FROM room_type_inventory
	WHERE room_type_id = ${roomTypeID} 
	AND hotel_id = ${hotelID}
	AND date >= ${startDate}
	AND date <= ${endDate}
```

![[hotel_reservation_query_result.png]]

2. เช็คว่าในทุกๆ row ที่เข้าข่ายนั้น...
```sql
	total_reserved + ${roomCount} <= 1.1 * total_inventory
```

หรือไม่ ถ้าใช่ในทุกๆ row แปลว่า User สามารถจองห้องประเภทนั้นของโรงแรมนั้นๆ ได้นั่นเอง

### Concurrency Issues

Concurrency Issues ในที่นี้จะแบ่งเป็น 2 อย่าง คือ...
#### 1. User กดจองห้องไปหลายครั้ง

ปัญหานี้สามารถแก้ได้ผ่าน 2 อย่าง
1. การแก้ UI ให้ disable ปุ่มจองห้องระหว่างที่ระบบกำลังจองห้องให้ User
2. การใช้ Idempotent Key อย่าง reservation ID โดยส่งเข้าไปใน Request ด้วย เพื่อเช็คว่าได้ทำการจองครั้งนี้ไปหรือยัง

![[hotel_reservation_idempotency.png]]

การส่ง reservationID ไปด้วยมันจะทำให้ตอนใส่ข้อมูลเข้า Schema Reservation มันจะโดน Schema ดักไม่ให้ใส่ข้อมูลได้ เพราะ reservationID นี้มันมีไปแล้ว ทำให้สามารถเลี่ยงการกดจองห้องติดหลายๆ ครั้งได้

#### 2. มี Users หลายคนจองห้องเดียวกันไป

ประมาณว่า มี User 2 คนเห็นว่าห้องนี้ว่าง ละจะกดจองห้องนี้ มันควรจะมีแค่คนเดียวที่ทำได้ แต่บังเอิญว่า User ทั้ง 2 คนกดจองมาในเวลาที่ไล่เลี่ยกัน แล้วก่อนที่จะ Commit การจองของ User 1 เข้าสู่ระบบ ดันมี Request 2 จาก User 2 เข้ามาพอดีแล้วกลายเป็นว่าของ User 2 ก็ดันเช็คละพบว่าจองได้ เพราะการจองของ User 1 ยังไม่เสร็จ ก็สามารถ Commit ได้อีกเหมือนกัน เลยจบที่ทั้ง 2 User จองห้องเดียวกันได้นั่นเอง

เป็นผลมาจาก Isolation ของ Relational Database ที่ Transaction 2 ตัวจะไม่ก้าวก่ายกันและกัน (อ่านต่อได้ใน [[01 Database#ACID Properties|Database]])

![[hotel_reservation_isolation_booking.png]]

การแก้เหตุการณ์นี้ทำได้ 2 วิธี

##### 1. Pessimistic Locking

บังคับไปเลยว่าระหว่างที่มี Transaction นึงทำงานอยู่ จนกว่าจะ Commit สำเร็จ จะล็อค row ที่กำลังอ่านไว้เลย แล้ว Transaction อื่นที่จะแตะ row เดียวกันต้องรอจนกว่าตัวแรกจะ Commit เสร็จ 

ทำให้พอ Transaction ของ User1 ทำงานอยู่ Transaction ของ User2 ต้องรอจน Transaction ของ User1 commit เสร็จก่อนถึงจะเริ่ม Transaction ของ User2 ได้ ซึ่งตอนนั้น reserved เป็น 100 แล้ว ทำให้ User2 จองไม่ได้นั่นเอง

![[hotel_reservation_pessimistic_locking.png]]

**ข้อดี** คือมันเข้าใจง่าย ป้องกัน Case นี้ได้ชัวร์ๆ 
**ข้อเสีย** คือเสี่ยง Deadlock (Transaction ต่างรอ Transaction อีกตัวทำงานก่อน รอกันไปรอกันมาจนระบบล่ม) และ **ไม่ scalable** เพราะ transaction ที่ล็อคนานจะบล็อคตัวอื่นหมด กระทบ performance

วิธีนี้เลย**ไม่แนะนำสำหรับ Hotel Reservation**

##### 2. Optimistic Locking

เพิ่ม Column `version` เข้าไปใน Database เพื่อเช็คตอนเขียนข้อมูลว่าเป็น version ปัจจุบันหรือยัง 
หลักการคือ จะอ่านเลข version มาก่อน พอตอนที่จะเขียนข้อมูลใน Database ก็เช็คก่อนว่า version ยังตรงกับตอนอ่านไหม ถ้ามีคนแก้ไปก่อน (version ไม่ตรง) transaction จะ fail แล้วให้ retry ใหม่ ถ้าตรงอยู่ก็เพิ่มเลข version ขึ้นไปอีก 1 -> หรือก็คือเป็นการทำ Versioning นั่นแหละ

![[hotel_reservation_optimistic_locking.png]]

**ข้อดี** คือทำงานได้เร็วกว่า Pessimistic เพราะไม่ล็อค DB  
**ข้อเสีย** คือตอน concurrency สูง performance จะตก เพราะทุกคนอ่าน version เดียวกัน แต่เขียนสำเร็จแค่คนเดียว ที่เหลือ fail แล้วต้อง retry วนไป

วิธีนี้เลย**เหมาะสำหรับ Hotel Reservation มากกว่า** เพราะว่า conflict เกิดไม่บ่อย QPS หรือ Query Per Second ไม่ได้มีเยอะมากนั่นเอง

> [!tip] สามารถใช้ Database Constraint เสริมได้
> นอกจากเช็ค version แล้ว ยังเพิ่ม Constraint ให้ Database ช่วยกันอีกชั้นได้
> 
> ```sql
> CONSTRAINT check_room_count CHECK (total_reserved <= 1.1 * total_inventory)
> ```
> 
> ถ้า Transaction ไหนพยายามทำให้  `total_reserved` เกิน `total_inventory` Database จะ reject แล้ว rollback ให้เองทันที 

### Scalability

Service ทั้งหมดใน Hotel Reservation System เป็น Stateless จึงสามารถเพิ่ม Server ได้ง่าย 
แต่ **Database เป็นตัวที่ scale ยาก** เพราะเก็บ state ทั้งหมดเอาไว้ 

พอเป็นงี้ จึงมี 2 เทคนิคที่สามารถทำได้

1. **Sharding** เพราะ query ส่วนใหญ่ filter ด้วย `hotel_id` เลย shard ตาม `hotel_id` ได้เลย เช่นแบ่ง 16 shard ด้วย `hotel_id % 16` เพื่อแบ่ง Database ให้เก็บข้อมูลอย่างพอๆ กัน และทำให้การ Query ทำได้รวดเร็วขึ้น

	![[hotel_reservation_sharding.png]]

2. **Caching** ที่ใช้ได้เพราะเราสนใจแค่ **วันปัจจุบันกับอนาคตเท่านั้น** (จองย้อนหลังไม่ได้) จึงสามารถใช้ Cache อย่าง Redis ที่ตั้ง TTL ให้ข้อมูลเก่าหมดอายุเองได้ โดยย้าย logic เช็ค/จองห้องมาที่ Cache layer ทำให้ request ส่วนใหญ่ถูกกรองที่ Cache ไม่ต้องแตะ DB -> แต่ก็ต้องมีการจัดการข้อมูลระหว่าง Cache กับ Database ดีๆ ด้วย เพราะข้อมูล Inventory จริงๆ อยู่ใน Database ไม่ใช่ Cache

	![[hotel_reservation_caching.png]]

### Data Consistency Among Services

ถ้าเป็น Monolith แบบปกติ มันก็ไม่มีปัญหาอะไร เพราะเราใช้ Database ก้อนเดียวเพื่อจัดการทุก Service แต่อันนี้เราเป็น Microservice ซึ่ง Database จะแยกกัน 

![[hotel_reservation_microservice_databases.png]]

รวมถึงใน Transaction หนึ่งอันมันอาจจะต้องแตกเป็น 2 Transaction ด้วย อาทิเช่น ในการจองห้อง มันก็มี Transaction แรกที่ไปแก้ไขข้อมูลใน `room_type_inventory` schema และ Transaction ที่สองที่เข้าไปเพิ่มข้อมูลใน `reservation` schema ซึ่งทำงานบนคนละ Service กัน

![[hotel_reservation_microservice_transactions.png]]

ซึ่งการจะถือว่าจองสำเร็จได้เนี่ย ต้องสำเร็จทั้ง 2 Transaction ถ้า fail ตั้งแต่ Transaction แรก หรือไป fail ตอน Transaction ที่สอง ก็ต้องมีการ rollback กลับไปให้เป็นเหมือนก่อนจะทำทั้งสอง Transaction ด้วยนั่นเอง โดยอาจจะใช้ 2-Phase Commit หรือว่า Saga เพื่อเข้ามา handle multiple transactions ด้วยก็ได้

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Hotel Reservation System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---