---
title: 05 Session-based Authentication
tags:
  - systemdesign
  - authentication
  - session
  - cache
---
ก่อนอื่นเลย Session คืออะไร -> คือแทนที่จะเก็บข้อมูลของ User ไว้ที่ Client เราก็เก็บข้อมูลของ User ไว้ที่ Server เลย แล้วให้ Server เป็นคนบอก Client ว่าข้อมูลของ Client มันอยู่ไหน ผ่านข้อมูลตัวนึง อารมณ์เหมือนกับการที่ User เก็บของตัวเองไว้ใน Locker แล้วถือติดตัวไปแค่กุญแจของ Locker นั้นก็พอ

Locker ในบริบทนี้จะเรียกว่า Session ส่วนกุญแจเนี่ยก็คือ Session ID ตามภาพนี้เลย

![[auth_session_authentication_overview.png]]

---

## 1. เราได้ Session มาจากไหน

User จะได้ Session หลังจากที่ User login สำเร็จ โดย Server จะสร้าง Session เก็บไว้ที่ฝั่ง Server แล้วส่ง Session ID กลับมาทาง `Set-Cookie` เพื่อให้ Client จด Session ID เก็บไว้ใน Cookie เพื่อป้องกัน XSS (อ่านต่อได้ใน [[04 Cookie|Cookie]])

ทีนี้พอ Client ส่ง Request ครั้งถัดๆ ไปก็จะมี Session ID ติดไปด้วย ทำให้ Server รู้ว่าใครเป็นคนเรียก Request นี้มา โดย Server จะต้องเอา Session ID ไปหาใน Session Store เพื่อดูว่าเป็นใคร ซึ่งต่างจากการส่ง JWT เข้าไปใน Header Request เองที่ JWT จะมีข้อมูลของ User อยู่แล้วใน JWT

![[auth_session_login_flow.png]]

ก็จะนำมาสู่ประเด็นถัดมา คือ... Session Store มันอยู่ตรงไหนของระบบ?

---

## 2. Session Store อยู่ไหน

ทางเลือกของ Session Store มีหลายที่ มีผลต่อความเร็วในการหา Session ของ User รวมถึงความยากง่ายในการ Scale ระบบ

1. เก็บไว้ใน Memory ของ Server -> หา Session ของ User ได้เร็ว แต่ว่ามันจะเป็นการผูกข้อมูลไว้กับ Server ตัวเดียว ถ้า Server นั้นมันล่มขึ้นมาก็เจ๊ง (Sticky Session)
2. เก็บไว้ใน Database -> แต่ละ Server สามารถหาข้อมูลได้หมด แต่ว่ากว่าจะหาเจอมันใช้เวลาเยอะ
3. เก็บไว้ใน Cache เช่น Redis -> นิยมสุด เพราะอยู่ใน Memory และเป็น Area ตรงกลางที่ให้ Server หลายๆ ตัวเข้ามาเช็คข้อมูลได้ เกิดเป็น Shared Session Store (อ่านเพิ่มเติมได้ใน [[06 Cache|Cache]])

![[auth_session_shared_session_store.png]]

แต่ว่าใดๆ คือ Session-Based Authentication มันก็สามารถถูกล็อคเป้าโจมตีจาก attacker ได้นะ ผ่านเทคนิค Session Hijacking กับ Session Fixation

---

## 3. Session Hijacking and Session Fixation

Session Hijacking คือการที่ attacker ขโมยเอา Session ID ของ User ไป -> ก็แก้ได้ด้วยลักษณะของ Cookie แหละ ตั้ง `HttpOnly` กับ `Secure` ก็ได้ละ

Session Fixation คือแทนที่ attacker จะพยายามขโมย Session ID ของ User ไป attacker จะยัด Session ID ที่ตนเองรู้ว่าค่าใน Session เป็นยังไงมาให้ User ใช้ Login ละพอ User login สำเร็จ ก็กลายเป็นว่าข้อมูลใน Session ID ดังกล่าวจะเป็นของ User แทน ก็ทำให้ attacker เอาข้อมูลของ User ไปได้ -> การแก้ก็ทำได้ผ่านการกำหนดให้สร้าง Session ใหม่ทุกครั้งหลัง login สำเร็จ ทีนี้ Session ID ของ attacker ก็จะไม่ถูกใช้ละ ข้อมูลของ User ก็ไม่ถูกเอาไป

---

