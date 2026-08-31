---
title: 06 Cache
tags:
  - database
  - database_optimization
  - systemdesign
  - cache
---
![[cache.png|697]]

**Cache** คือ**พื้นที่เก็บข้อมูลชั่วคราว**ที่เก็บสำเนาของข้อมูลที่ถูกเรียกใช้บ่อยๆ ไว้ในที่ที่เข้าถึงได้เร็วกว่าต้นทางเดิม (เช่น Database หรือ External API) เพื่อให้ครั้งต่อไปที่มีการขอข้อมูลเดิม ระบบสามารถดึงจาก Cache ได้เลยโดยไม่ต้องไปคำนวณหรือ Query ข้อมูลจากต้นทางซ้ำ

---

## 1. นิยามศัพท์

- **Cache Hit** กรณีที่ขอข้อมูลแล้วเจอใน Cache เลย ไม่ต้องไปที่ Database (เร็ว)
- **Cache Miss** กรณีที่ขอข้อมูลแล้วไม่มีใน Cache เลยต้องไปดึงจาก Database แล้วค่อยเก็บผลลัพธ์ลง Cache ไว้ใช้ครั้งถัดไป
- **Hit Rate** สัดส่วนของ Request ที่เป็น Cache Hit เทียบกับ Request ทั้งหมด (ยิ่งสูงยิ่งดี แปลว่า Cache ทำงานคุ้มค่า)
- **TTL (Time To Live)** ระยะเวลาที่ข้อมูลใน Cache จะยังถือว่า "ใช้ได้" ก่อนจะถือว่าหมดอายุ แล้วถูกลบออก หรือว่าดึงมาเก็บใน Cache ใหม่

---

## 2. ตัวอย่าง

- **Client-side Cache** คือ Cache ที่เก็บที่ฝั่ง Browser/Client (เช่น Browser Cache, Local Storage)
- **CDN Cache** คือ Cache ที่เก็บที่ Server ที่กระจายอยู่ทั่วโลก (Content Delivery Network) ใกล้ User เพื่อลด Latency ของไฟล์ Static เช่นรูปภาพ, JS, CSS
- **Application-level Cache** คือ Cache ที่เก็บในหน่วยความจำของ Application เอง หรือใช้ระบบแยกต่างหากอย่าง **Redis**
- **Database Cache** คือ Cache ใน Database เอง สำหรับ Query ที่ถูกเรียกซ้ำๆ (เช่น Query Cache, Buffer Pool)

---

## 3. ข้อดี

1. **ลด Latency** เพราะการดึงข้อมูลจาก Cache เร็วกว่าไป Query จาก Database มาก เพราะ Cache มักอยู่ใน RAM ซึ่งเร็วกว่าการดึงข้อมูลจาก Disk มาก
2. **ลดภาระของ Database / Backend** เพราะ Request ที่ตอบได้จาก Cache ไม่ต้องไปถึง Database เลย ลดจำนวน Query และลดโอกาส Database Overload
3. **เพิ่ม Throughput** เพราะเมื่อ Backend มีภาระน้อยลง ระบบก็รองรับ Traffic โดยรวมได้มากขึ้น
4. **ลด Network Traffic** ได้ โดยเฉพาะ CDN Cache ที่เก็บไฟล์ไว้ใกล้ User ไม่ต้องส่งข้อมูลข้ามประเทศทุกครั้ง

---

## 4. ข้อเสีย

1. **ข้อมูลเก่าไม่ตรงกับปัจจุบัน** เพราะถ้าข้อมูลต้นทางเปลี่ยนแล้วแต่ Cache ยังไม่ได้อัปเดต ระบบอาจส่งข้อมูลเก่าให้ User โดยไม่รู้ตัว (Cache Invalidation)
2. **เพิ่มความซับซ้อนของระบบ** เพราะต้องออกแบบว่าจะ Cache อะไร, TTL เท่าไหร่, จะ Invalidate เมื่อไหร่ เป็นต้น
3. **มีค่าใช้จ่ายเพิ่มขึ้น** เพราะ Cache ต้องใช้พื้นที่เก็บข้อมูล ทำให้มีต้นทุนเพิ่มขึ้น
4. **Cache Stampede** กรณีที่ข้อมูลใน Cache หมดอายุพร้อมกัน Request จำนวนมากจะพุ่งไปที่ Database พร้อมกันในจังหวะเดียว จนอาจทำให้ Database รับไม่ไหว
5. **Consistency ยากขึ้นในระบบที่มีหลาย Server** เพราะต้องทำให้ Cache ทุกจุด sync กันพร้อมกัน

---

## 5. วิธีป้องกันปัญหาของ Cache

### 1. ป้องกัน Data Staleness / Cache Invalidation ผิดพลาด

- ตั้ง **TTL** ให้เหมาะสมกับลักษณะข้อมูลตั้งแต่ Design (ข้อมูลที่เปลี่ยนบ่อยควรมี TTL สั้น ข้อมูลที่นิ่งเก็บ TTL ยาวได้)
- ทำ **Write-through** (เขียนข้อมูลลง Cache พร้อมกับ Database ทุกครั้งที่มีการ Write) หรือ **Cache-Aside with Explicit Invalidation** (อัปเดต Cache ทันทีเมื่อข้อมูลต้นทางถูกแก้ไข) 

### 2. ป้องกัน Cache Stampede

- ใช้เทคนิค **Cache Warming** (โหลดข้อมูลลง Cache ล่วงหน้าก่อนที่ TTL จะหมดอายุจริง) เพื่อไม่ให้ Cache ว่างพร้อมกันหลายตัว
- Setup ให้แค่ Request แรกไป Query Database ตอน Cache Miss ส่วน Request อื่นๆ ที่มาพร้อมกันรอผลลัพธ์แทนที่จะยิง Database ซ้ำ
- ให้ TTL ให้ไม่ตรงกัน โดยเพิ่มค่าสุ่มเล็กน้อยให้กับ TTL ของแต่ละ Key ทำให้ TTL มีค่าไม่เท่ากัน

### 3. ป้องกันปัญหา Memory เต็ม

- ตั้ง **Eviction Policy** ที่เหมาะสมตั้งแต่แรก (เช่น LRU: Least Recently Used เอาข้อมูลที่ไม่ใช้นานที่สุดออกไป) พร้อมกำหนด Memory Limit ชัดเจน ไม่ให้ Cache กินทรัพยากรจนกระทบระบบอื่น
- Monitor Hit Rate และขนาดการใช้ Memory ของ Cache เป็นระยะ เพื่อปรับขนาด/นโยบายให้เหมาะกับ Traffic จริง

### 4. ป้องกันปัญหา Consistency ระหว่าง Cache หลายจุด

- ถ้าเป็นไปได้ กำหนด Cache กลางที่ทุก Server เรียกใช้ร่วมกัน เช่น Redis Cluster แทนการให้แต่ละ Server มี Local Cache แยกกัน เพื่อลดปัญหาข้อมูลไม่ตรงกันระหว่าง Server
- ถ้าจำเป็นต้องใช้ Local Cache ต่อ Server ให้วางระบบ **Pub/Sub Invalidation** คือมีการแจ้งเตือนทุก Server ให้ล้าง Cache พร้อมกันเมื่อข้อมูลเปลี่ยนไว้ตั้งแต่ตอนกำหนด Design

---
