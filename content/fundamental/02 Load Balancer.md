---
title: 02 Load Balancer
tags:
  - database
  - systemdesign
  - database_optimization
  - load_balancer
  - fundamental
---
![[load_balancer.png]]

**Load Balancer (LB) ตามชื่อเลย คือตัวกระจายโหลด** 
ทำหน้าที่รับ Request จาก User แล้วกระจายไปยัง Server หลายๆ ตัว เพื่อแจกจ่ายภาระ**ให้เหมาะสมกับความพร้อมของ Server แต่ละตัว

> [!note] Server แต่ละตัว**ไม่จำเป็น**ต้องได้รับ Request ในจำนวนที่เท่ากัน ขึ้นอยู่กับ Algorithm ที่ LB ใช้ และสภาพความพร้อม (capacity, load ปัจจุบัน) ของแต่ละ Server ณ ขณะนั้น

---

## 1. หน้าที่หลัก

- กระจาย Request ไปยัง Server ที่เหมาะสม
- ตรวจสอบสถานะของแต่ละ Server (Health Check) และหยุดส่ง Request ไปให้กับ Server ที่ล่ม หรือไม่พร้อมใช้งาน

## 2. Algorithm ที่ใช้กระจาย Request

- **Round Robin** คือการวนส่งไปทีละ Server ตามลำดับ เช่น Request 1, 3, 5 ให้ Server A ส่วน Request 2, 4, 6, ให้ Server B แล้ววนซ้ำเรื่อยๆ
- **Weighted Round Robin** คือการทำ Round Robin แต่ให้น้ำหนักตาม Spec ของ Server (Server แรงกว่า รับ Request มากกว่า) เช่น Request 1, 2, 3 ให้ Server A ส่วน Request 4 ให้ Server B แล้ววนซ้ำเรื่อยๆ
- **IP Hash** คือการใช้ IP ของ User มาคำนวณว่าต้องส่งไปที่ Server ไหน (ช่วยให้ User เดิมไปที่ Server เดิมเสมอ)

---

## 3. ข้อดีของ Load Balancer

1. **เพิ่ม Availability / Reliability** คือถ้า Server ตัวใดตัวหนึ่งล่ม LB จะไม่ส่ง Request ไปยัง Server นั้น ทำให้ระบบยังทำงานต่อได้
2. **ทำ Scalability ได้** เพราะเราสามารถเพิ่ม/ลด Server ได้โดยไม่กระทบ User
3. **ลด Overload** เพราะมีการกระจายงานไม่ให้ Server ใดตัวหนึ่งรับ Request หนักเกินไป
4. **เพิ่ม Performance** เพราะมีการส่ง Request ไปยัง Server ที่พร้อมที่สุด ทำให้ Response Time โดยรวมดีขึ้น
5. **Flexibility ในการ Maintenance** เพราะเราสามารถถอด Server ออกมาซ่อมบำรุงได้โดยระบบไม่เกิด Downtime

---

## 4. ข้อเสียของ Load Balancer

1. **Single Point of Failure** เพราะถ้า LB เองล่ม ระบบทั้งหมดอาจใช้งานไม่ได้
2. **เพิ่มความซับซ้อนของระบบ** เพราะต้อง setup config, monitor และ maintain LB
3. **มี Latency เพิ่มขึ้นเล็กน้อย** เพราะ Request ต้องผ่าน LB ก่อนถึง Server จริง
4. **มีค่าใช้จ่ายเพิ่ม** ค่า LB นี่แหละ รวมถึงค่าบำรุงด้วย
5. **ต้องจัดการ Session ดีๆ** เพราะถ้า Request ของ User เดิมไปคนละ Server ก็จะต้องจัดการเรื่อง Session/State ร่วมกันด้วย
6. **ต้องตั้งค่า Algorithm ให้เหมาะสมกับการใช้งาน** เลือกผิดระบบพัง

---
## 5. วิธีป้องกันปัญหาของ Load Balancer

### 1. ป้องกัน Single Point of Failure

- ใช้ **Cloud Load Balancer** (เช่น AWS ELB/ALB, GCP Load Balancing) ที่จัดการ Redundancy ให้
- มี Setup ที่ route ไปยัง LB ตัวอื่นได้ทันทีถ้าตัวหลักไม่ตอบสนอง

### 2. ป้องกัน Overload

- ออกแบบให้ Scale Out ได้ตั้งแต่แรก (เพิ่มจำนวน LB ได้ง่ายผ่าน DNS Round Robin)
- เลือก Algorithm ให้เหมาะกับ Traffic Pattern ตั้งแต่ต้น
- มี Load Testing เพื่อรู้ Threshold ก่อนจะถึงจุด Overload จริง

### 3. ป้องกัน Health Check ผิดพลาด

- ออกแบบ Health Check ให้เช็ค Endpoint มากพอที่จะบอกว่า Server พร้อม

### 4. ป้องกันปัญหา Session/State

- ให้ **Stateless** ตั้งแต่แรก หรือใช้ **Shared Session Store** (เช่น Redis) แทนการเก็บ Session ไว้ที่ Server ตัวเดียว หรือถ้าต้องผูก State กับ Server จริงๆ ก็ต้องคิดเรื่อง Server ล่มด้วยว่าจะเก็บ Session เหล่านั้นไว้ที่ไหนแทน

---
