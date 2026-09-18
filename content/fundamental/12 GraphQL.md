---
title: 12 GraphQL
tags:
  - systemdesign
  - graphql
  - fundamental
---
![[graphql.png]]

**GraphQL** คือ Query Language และ Runtime สำหรับ API ที่ Client เป็นคนกำหนดเองว่าต้องการข้อมูลอะไรบ้าง (Field ไหนบ้าง) ใน Request เดียว แทนที่จะต้องพึ่ง Endpoint ตายตัวแบบ [[11 API vs gRPC|REST API]] ที่ Server เป็นคนกำหนดว่าแต่ละ Endpoint จะ Return อะไรกลับมา

---

## 1. องค์ประกอบหลัก

- **Schema** - นิยามโครงสร้างข้อมูลและความสัมพันธ์ทั้งหมดในระบบ (Type, Field, Relationship) เขียนด้วย **SDL (Schema Definition Language)** เป็น Contract กลางระหว่าง Client-Server
- **Query** - การขออ่านข้อมูล (เทียบเท่า GET ใน REST) โดยระบุ Field ที่ต้องการได้ละเอียดถึงระดับ Nested Object
- **Mutation** - การขอแก้ไขข้อมูล (เทียบเท่า POST/PUT/DELETE ใน REST)
- **Subscription** - การขอรับข้อมูลแบบ Real-time ผ่าน WebSocket เมื่อข้อมูลมีการเปลี่ยนแปลง
- **Resolver** - Function ฝั่ง Server ที่ทำหน้าที่ไปดึงข้อมูลจริงมาตอบแต่ละ Field ใน Query (อาจไปดึงจาก Database, Service อื่น, หรือ API ภายนอกก็ได้)

---

## 2. ข้อดี

1. **ขอข้อมูลได้ตรงตามที่ต้องการ (No Over-fetching)** เพราะ Client ระบุ Field เองได้ ไม่ต้องรับข้อมูลส่วนเกินที่ไม่ได้ใช้เหมือน REST ที่มักจะ Return ทั้ง Object
2. **ลดจำนวน Request (No Under-fetching)** เพราะ Query เดียวสามารถดึงข้อมูลจากหลาย Resource ที่มีความสัมพันธ์กันได้พร้อมกัน (เช่น ดึง User พร้อม Order และ Product ในครั้งเดียว) ต่างจาก REST ที่อาจต้องยิงหลาย Endpoint ต่อกัน
3. **มี Schema/Contract ชัดเจน** เพราะ Type ทุกตัวถูกนิยามไว้ล่วงหน้า ทำให้ Client รู้ล่วงหน้าว่ามี Field อะไรบ้าง ลดปัญหา Documentation ไม่ตรงกับของจริง
4. **Version API ได้ง่ายกว่า** เพราะสามารถเพิ่ม Field ใหม่เข้าไปใน Schema ได้โดยไม่กระทบ Client เดิมที่ไม่ได้เรียกใช้ Field นั้น ลดความจำเป็นในการทำ /v1, /v2 แบบ REST
5. **เหมาะกับ Frontend ที่หลากหลาย** เพราะ Mobile App กับ Web App ที่ต้องการข้อมูลไม่เท่ากัน สามารถ Query ต่างกันได้จาก Schema เดียวกัน โดยไม่ต้องสร้าง Endpoint แยกให้แต่ละฝั่ง

---

## 3. ข้อเสีย

1. **Caching ทำได้ยากกว่า REST** เพราะ REST ใช้ URL เป็น Key ในการ Cache ได้ตรงไปตรงมา (เช่นผ่าน [[07 CDN (Content Delivery Network)|CDN]]) แต่ GraphQL ยิง Request ไปที่ Endpoint เดียวเสมอ (มักเป็น POST) ทำให้ Cache ระดับ HTTP/CDN ทำได้ยาก ต้องพึ่งเครื่องมือเฉพาะทาง
2. **เสี่ยงต่อ N+1 Query Problem** เพราะถ้า Resolver แต่ละ Field ไปดึงข้อมูลจาก Database แยกกันโดยไม่ระวัง อาจเกิดการ Query ซ้ำจำนวนมากเกินจำเป็น (เช่น Query User 1 ครั้ง แต่ดึง Order ของ User แต่ละคนแยกทีละ Query)
3. **Client ควบคุม Load ของ Server ได้มากเกินไป** เพราะ Client สามารถเขียน Query ที่ซับซ้อนมาก (Nested ลึกๆ) จนสร้างภาระให้ Server หนักเกินคาด ถ้าไม่มีการจำกัดไว้
4. **Learning Curve สูงกว่า REST** เพราะต้องเรียนรู้ Schema, Resolver, และวิธีเขียน Query/Mutation ใหม่ ต่างจาก REST ที่ใช้ HTTP Verb พื้นฐานที่คุ้นเคยอยู่แล้ว
5. **Error Handling ไม่ตรงไปตรงมาเท่า REST** เพราะ GraphQL มักจะ Return HTTP Status 200 เสมอแม้ Query จะ Error บางส่วน ทำให้ต้องเข้าไปดูใน Body ของ Response เพื่อเช็ค Error จริงๆ

---

## 4. วิธีป้องกันปัญหาของ GraphQL

### 1. ป้องกัน N+1 Query Problem

- ใช้ **DataLoader** (Library ที่ช่วย Batch และ Cache การเรียก Resolver ในรอบเดียวกัน) เพื่อรวม Query ที่ซ้ำกันให้เหลือรอบเดียวแทนที่จะยิงแยกทีละ Record

### 2. ป้องกัน Client ยิง Query หนักเกินไป

- ตั้ง **Query Depth Limiting** (จำกัดความลึกของ Nested Query ที่ยอมรับได้)
- ตั้ง **Query Complexity Analysis** (คำนวณ "คะแนน" ความหนักของแต่ละ Query ก่อนรัน แล้วปฏิเสธถ้าหนักเกิน Threshold ที่กำหนด)
- ทำ **Rate Limiting** ที่ระดับ Query ไม่ใช่แค่ระดับ Request เหมือน REST

### 3. ป้องกันปัญหา Caching ยาก

- ใช้ Client Library ที่มี Cache ในตัวอย่าง **Apollo Client** หรือ **Relay** ซึ่งจัดการ Cache แบบ Normalized (แยกเก็บตาม ID ของแต่ละ Object) แทนการพึ่ง HTTP Cache
- พิจารณาใช้ **Persisted Queries** (เก็บ Query ที่ใช้บ่อยไว้เป็น ID สั้นๆ ฝั่ง Server ล่วงหน้า) เพื่อให้ CDN/Proxy สามารถ Cache ได้ง่ายขึ้นในบางกรณี

### 4. ป้องกันปัญหา Error Handling ไม่ชัดเจน

- Return Error Detail ที่ครบถ้วนใน `errors` field ของ Response พร้อม Error Code ที่ Custom ไว้ชัดเจน (ไม่ใช่แค่ Message ตรงๆ)
- ทำ Logging/Monitoring แยกตาม Field ที่เกิด Error เพื่อตามปัญหาได้ง่ายขึ้นเวลามีแค่บาง Field ใน Query ที่ Fail

### 5. ป้องกันความซับซ้อนที่ Introduce เข้ามาโดยไม่จำเป็น

- พิจารณาใช้ REST ต่อไปถ้า API ยังไม่ซับซ้อน หรือ Client มีแค่ประเภทเดียวที่ต้องการข้อมูลเหมือนกันหมด (ไม่ได้ประโยชน์จาก Flexibility ของ GraphQL มากพอจะคุ้มกับ Learning Curve และ Infrastructure ที่เพิ่มขึ้น)

---

## 5. แนวทางเลือกใช้ (เทียบกับ REST / gRPC)

- ใช้ **GraphQL** เมื่อมี Client หลากหลายชนิด (Web, Mobile, Third-party) ที่ต้องการข้อมูลไม่เท่ากัน และต้องการลด Over/Under-fetching
- ใช้ **[[11 API vs gRPC|REST API]]** เมื่อต้องการความเรียบง่าย, Cache ง่ายผ่าน CDN, และ Public API ที่ Developer ภายนอกคุ้นเคยอยู่แล้ว
- ใช้ **[[11 API vs gRPC|gRPC]]** สำหรับการสื่อสารภายในระหว่าง [[10 Microservices|Microservices]] ที่ต้องการ Performance สูงและ Streaming
---

## เพิ่มเติม

N+1 Query Problem คือปัญหาที่เกิดตอนดึงข้อมูลแบบมี Relationship กัน แล้ว Code ไปยิง Query แยกทีละ Record แทนที่จะยิงรวมทีเดียว ทำให้จำนวน Query ที่ยิงจริงเยอะกว่าที่ควรจะเป็นมาก

สมมติมี Query แบบนี้:
```graphql
{
  users {
    name
    orders {
      product
    }
  }
}
```

ถ้า Resolver เขียนแบบไม่ระวัง จะเกิด Query ดังนี้

1. `SELECT * FROM users` → 1 Query (ดึง User มา 10 คน)
2. สำหรับ User แต่ละคน วนไปดึง Order ของตัวเองแยกกัน
	1. `SELECT * FROM orders WHERE user_id = 1`
	2. `SELECT * FROM orders WHERE user_id = 2`
	3. `SELECT * FROM orders WHERE user_id = 3`
	4. `SELECT * FROM orders WHERE user_id = 4`
	5. `SELECT * FROM orders WHERE user_id = 5`
	6. `SELECT * FROM orders WHERE user_id = 6
	7. `SELECT * FROM orders WHERE user_id = 7`
	8. `SELECT * FROM orders WHERE user_id = 8`
	9. `SELECT * FROM orders WHERE user_id = 9`
	10. `SELECT * FROM orders WHERE user_id = 10`

→ 10 Query (N = 10)

รวมแล้วคือ 1 + 10 = 11 Query ทั้งที่จริงๆ ควรจะดึง Order ของ User ทั้งหมดได้ในทีเดียวด้วย Query เดียว เช่น `SELECT * FROM orders WHERE user_id IN (1,2,3,...,10)` ซึ่งจะใช้รวม 2 Query เท่านั้น

---
