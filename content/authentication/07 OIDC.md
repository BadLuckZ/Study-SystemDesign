---
title: 07 OIDC
tags:
  - systemdesign
  - authentication
  - oidc
  - jwt
---
ย้อนกลับไปใน [[06 OAuth|OAuth]] คือ OAuth จะมีหน้าที่ในการอนุญาตให้ App นั้นๆ สามารถเข้าถึง Resources ใดได้บ้าง แต่ตัวมันไม่ได้มีหน้าที่ในการ Verify ว่าเป็น User จริงๆ มั้ย เพราะงั้น ในเรื่องการ Authentication จึงตกมาที่ OIDC หรือ OpenID Connect นี่เอง

---

## 1. OIDC คืออะไร

OIDC หรือ OpenID Connect คือตัวเสริมด้าน Authentication ให้กับ OAuth ที่ทำเพียง Authorization ว่า Client สามารถเข้าถึง Resources ใดได้บ้าง โดยสิ่งที่ OIDC ทำ คือการใช้ ID Token ที่เป็น JWT: Json Web Token (ส่วนอะไรคือ JWT อ่านได้ใน [[01 Access Token|Access Token]]) และเซ็น Signature โดย Google เพื่อบอก Application ว่า User ที่ใช้งานคือใคร

![[auth_oidc_tokens.png]]

เพราะงั้น สำหรับระบบนี้ Access Token คือ Authorization นะ ไม่ใช่ Authentication

---

## 2. ข้อมูลใน ID Token

ข้อมูลใน ID Token เราจะเรียกว่า Claims ก็คือเป็นเหมือนกับ Payload ใน Access Token ที่ Server เป็นคนออกให้แหละ มี field อะไรบ้าง มาดูกัน
- `sub`: รหัสประจำตัวของ User ที่ไม่ซ้ำกัน -> Application ควรต้องเช็คว่า User เป็นใครผ่าน field นี้ ไม่ใช่ email
- `email`: อีเมลของ User
- `name`: ชื่อของ User
- `picture`: รูปโปรไฟล์ของ User
- `iss`: Token นี้ออกโดยใคร
- `aud`: Token นี้อนุญาตให้ Application ใดใช้ได้บ้าง

---

## 3. OIDC Usage Flow

ทีนี้เรามาดูกันดีกว่าว่า OIDC มันถูกใช้งานตอนไหน โดยจะเริ่มไล่จาก Flow ที่เป็น OAuth ละกัน
0. User ลงทะเบียน Application กับ Google ทำให้ได้รับ Client ID กับ Client Secret มาใส่ใน .env ของ Application Server
1. User กด Login with Google 
2. User เลือก Email ที่ต้องการจะ Login (ถ้า Email นั้น Session Expired ก็จะได้กรอกรหัสด้วย) โดยที่การ login นี้จะอยู่นอกการมองเห็นของ Application
3. เมื่อ Login สำเร็จ Google ก็จะส่ง Authorization Code กลับมาให้ ผ่าน Browser ของ Application
4. Application Server ส่ง Authorization Code ไปให้ พร้อมแนบ Client Secret ของ Application ไปให้ด้วย
5. Google ตรวจสอบแล้วพบว่าข้อมูลตรงกัน ก็จะส่ง Access Token และ Refresh Token มาให้ เพื่ออนุญาตให้ Application สามารถเข้าใช้งาน Resources ได้

ตรงนี้จะยังเป็นของเดิมอยู่ ทีนี่เราจะต่อ OIDC เข้าไป

5. นอกเหนือจาก Access Token กับ Refresh Token แล้ว Google จะส่ง ID Token ของ Application นั้นกลับมาด้วย
6. Application Server นำ ID Token มาตรวจสอบ Signature ว่าเป็นของ Google จริงหรือไม่ ถ้าใช่ก็จด `sub` ไว้ว่าใครเป็น User ที่ login เข้ามา
7. นำ `sub` ไปเช็คใน Database ของ Application ว่าเคยเข้าใช้งาน Application หรือยัง (มี Record ของ User มั้ย) ถ้ามีก็ถือว่า Login สำเร็จ ถ้าไม่มีก็สร้าง User ใหม่ใน Database ที่ผูกกับ `sub` ที่ได้

---