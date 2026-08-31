---
title: 08 Stateful VS Stateless
tags:
  - systemdesign
  - stateful
  - stateless
---
![[stateful_stateless.png]]

**State** คือข้อมูลที่ระบบต้องจำไว้ระหว่าง Request หนึ่งกับ Request ถัดไป (เช่น ข้อมูล Login, ตะกร้าสินค้า, ขั้นตอนที่ User ทำถึงไหนแล้ว) การออกแบบระบบแบ่งได้เป็น 2 แนวคิดหลัก:

- **Stateful**: Server **จำ State** ของ Client ไว้ (เช่น เก็บ Session ไว้ใน Memory ของ Server ตัวนั้น) แปลว่า Request ครั้งถัดไปของ Client คนเดิม**ต้อง**ไปที่ Server ตัวเดิม
- **Stateless**: Server **ไม่จำ**อะไรเกี่ยวกับ Client เลย แต่ละ Request ต้องส่งข้อมูลที่จำเป็นทั้งหมดมาด้วยตัวเอง (เช่น แนบ Token ยืนยันตัวตนมาทุกครั้ง) ทำให้ Request ไปที่ Server ตัวไหนก็ได้ในกลุ่ม

---

## 1.1. ข้อดีของ Stateless

1. **Scale ง่าย** เพราะสามารถเพิ่ม/ลด Server ได้อิสระ เพราะทุก Server เหมือนกันหมด ไม่มี Server ตัวไหนรู้อะไรมากกว่าตัวอื่น
2. **ทำงานกับ Load Balancer ได้ดี** เพราะสามารถ set LB ให้เลือก Algorithm แบบ Round Robin ธรรมดาได้
3. **Fault Tolerance สูง** เพราะถ้า Server ตัวหนึ่งล่ม Request ไปตัวอื่นได้ทันทีโดยไม่มีข้อมูลหาย เพราะไม่มีอะไรเก็บไว้ที่ Server อยู่แล้ว

## 1.2. ข้อเสียของ Stateless

1. **ต้องส่งข้อมูลซ้ำๆ ทุก Request** เป็นการเพิ่มขนาดของแต่ละ Request (เช่นต้องแนบ Token ทุกครั้ง)
2. **ต้องพึ่ง External Storage สำหรับ State ที่จำเป็นจริงๆ** เช่นต้องมี Database หรือ Redis เก็บข้อมูล User แทนที่จะเก็บใน Memory ของ Server เอง ซึ่งเพิ่ม Latency เล็กน้อย
3. **ไม่เหมาะกับการทำ Real-Time Application** เพราะจำเป็นต้องเก็บ State ของผู้ใช้งานอย่างต่อเนื่อง

---

## 2.1. ข้อดีของ Stateful

1. **Performance ดีกว่าในบาง Use Case** เพราะข้อมูลอยู่ใน Memory ของ Server เอง ไม่ต้องไป Query จาก External Storage ทุกครั้ง
2. **เหมาะกับ Real-time / Interactive Application** เช่นเกมออนไลน์ หรือ WebSocket Connection ที่ต้องคงการเชื่อมต่อและ State ต่อเนื่อง

## 2.2. ข้อเสียของ Stateful

1. **Scale ยากกว่า** เพราะต้องใช้ Sticky Session ผูก Client กับ Server ตัวเดิมเสมอ ทำให้กระจายโหลดไม่สม่ำเสมอ
2. **Fault Tolerance ต่ำกว่า** เพราะถ้า Server ที่เก็บ State ล่ม ข้อมูล State นั้นอาจหายไปเลย (เว้นแต่จะทำ Replication ของ State ไว้)

---

