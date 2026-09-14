---
title: 02 Refresh Token
tags:
  - refresh_token
  - system
  - authentication
---
ย้อนความใน Access Token ที่มีหน้าที่ในการ identify ว่า User ใดเป็นคนส่ง Request มา เรา pause ไว้ที่ปัญหาของ Access Token อย่างนึงว่าเราควรจะกำหนดอายุของ Access Token ไว้แค่ไหนดี ตามไปอ่านได้ใน [[01 Access Token|Access Token]]

---

## 1. Refresh Token เข้ามาแก้ปัญหานี้ยังไง

เราสามารถกำหนด 2 Token ที่ทำหน้าที่แยกกัน โดยให้ **Access Token** ทำหน้าที่เดิม คือแนบไปกับทุก Request แต่... <ตั้งอายุให้สั้นเข้าไว้> เช่น 15 นาที เพื่อจำกัดความเสียหาย

แต่เพื่อลดความน่ารำคาญที่อาจเกิดขึ้นจากการที่ต้อง login บ่อย ก็เป็น Token ใหม่ที่เข้ามาจัดการ คือ **Refresh Token** ที่จะเป็น Token ใหม่ที่... <มีอายุยาว> เช่น 30 วัน โดยไว้ใช้สำหรับการขอ Access Token ใหม่จาก Server ที่ทำหน้าที่ออก Access Token ใหม่ ไว้ใช้แทน Access Token เดิมที่หมดอายุ

---

## 2. Refresh Token Flow

เมื่อ Access Token หมดอายุ Client จะไม่พา User กลับไปหน้า Login แต่จะจัดการต่ออายุให้เองเบื้องหลัง

![[auth_refresh_token_flow.png]]

1. Client เรียก API โดยแนบ Access Token ไปใน HTTP Header
2. Server ตรวจแล้วพบว่าหมดอายุ จึงตอบกลับ 401 Unauthorized แล้วจบ Request นั้น
3. เมื่อ Client ได้รับ 401 ระบบจะยิง Request ใหม่ ไปที่ Auth Server พร้อม Refresh Token ที่ Client เก็บไว้ เพื่อขอ Access Token ใหม่
4. เมื่อ Client ได้ Access Token ใหม่แล้ว ระบบจะยิง Request เดิมจากข้อ 1 ซ้ำอีกครั้ง พร้อมใช้ Access Token ใหม่ที่ได้ไป

User จะถูกบังคับให้ Login ใหม่จริงๆ ก็ต่อเมื่อ Refresh Token เองหมดอายุแล้วเท่านั้น ซึ่งหมายถึงไม่ได้เข้าใช้งานเลยเป็นเวลานาน

---

## 3. Database Schema

ปัญหาของ Access Token คือ Revoke ไม่ได้ แบบว่า... ถ้า Server ออก Token นี้ไปแล้ว ต้องรอให้ Token มันหมดอายุไปเอง พอมี Refresh Token แล้วมันช่วยแก้ยังไงกันนะ...

![[auth_database_schema.png]]

จะเห็นว่า...
1. ไม่มีตารางสำหรับเก็บ Access Token เพราะเราต้องการให้ระบบตรวจสอบ Access Token ได้โดยไม่ต้องแตะ Database ในทุกๆ Request 
2. Refresh Token ที่มีอายุยาว จึงจำเป็นต้องควบคุมได้ว่าจะยกเลิกเมื่อไหร่ เช่นตอน User กด Logout จากทุกอุปกรณ์ หรือตอนพบว่าบัญชีถูกบุกรุก การเก็บลง Database คือสิ่งที่ทำให้ควบคุมแบบนี้ได้

เหตุผลของแต่ละ Column มีดังนี้
- **`token_hash` ไม่ใช่ `token`** ใช้หลักการเดียวกับ Password คือเก็บแค่ผลลัพธ์จากการ Hash ถ้า Database หลุด จะได้ไปแค่ Hash ซึ่งย้อนกลับไปเป็น Token ตัวจริงไม่ได้
- **`issued_at`** ไว้บอกว่า Refresh Token นี้ถูกสร้างขึ้นมาเมื่อไหร่
- **`expired_at`** ไว้บอกว่า Refresh Token นี้จะหมดอายุเมื่อไหร่
- **`revoked_at`** ถ้าไม่ใช่ค่าว่าง แปลว่า Token นี้ถูกยกเลิกไปแล้วเมื่อเวลา t แม้ `expires_at` จะยังไม่ถึงกำหนดก็ตาม ตอนตรวจสอบ Server จึงต้องเช็คทั้งสองเงื่อนไขควบคู่กัน

> [!tip] Refresh Token ไม่จำเป็นต้องเป็น JWT
> เมื่อ Refresh Token ต้อง Query Database ทุกครั้งอยู่แล้ว มันจึงไม่ได้ประโยชน์จากการเป็น Self-contained เลย ระบบจริงส่วนใหญ่ใช้เป็น Random String ยาวๆ ธรรมดา เพราะข้อมูลทั้งหมดที่ต้องรู้อยู่ใน Database Row อยู่แล้ว ไม่ต้องซ้อนไว้ในตัว Token อีกชั้น

---

## 4. Refresh Token Rotation

ยังเหลือช่องโหว่หนึ่งที่ยังไม่ได้แก้ ก็เหมือน Access Token ที่ถ้า Refresh Token ถูกขโมยไป ระบบจะรู้ได้ยังไง ในเมื่อทั้งเจ้าของ และคนร้ายต่างก็ถือ Token ที่ถูกต้องเหมือนกัน

Refresh Token Rotation แก้ปัญหานี้ด้วยการอนุญาตให้ Refresh Token ใช้ได้ครั้งเดียวเท่านั้น คือ...

> ทุกครั้งที่ขอ Access Token ใหม่ ระบบจะออก Refresh Token ใบใหม่มาแทน แล้วยกเลิก Refresh Token เก่าทิ้งไปทันที

เป็นไปตาม Flow ในภาพนี้

![[auth_refresh_token_rotation.png]]

เพราะงั้นการที่มีคนใช้ Refresh Token ที่ถูก revoked ไปแล้ว จึงเป็นสัญญาณที่ชัดเจนว่าไม่ใช่ User ตัวจริง เพราะ User ตัวจริงจะได้ Refresh Token ที่ไม่ถูก revoked ไปเสมอ

เมื่อตรวจพบ ระบบจะสามารถไล่ revoke Refresh Token ทั้งหมดผ่านการเช็ค user id ที่เป็นเจ้าของ Refresh Token ทั้งหมดนั้น บังคับให้ User ถูก logout ใหม่ แต่แลกมากับการที่คนร้ายก็ใช้งานต่อไม่ได้ด้วย

---

## 5. เก็บ Token ไว้ที่ไหนดี

การออกแบบฝั่ง Server ครบแล้ว แต่ยังเหลือคำถามสำคัญว่า Client ควรเก็บ Token ไว้ที่ไหนดี มี 2 ทางเลือกคือ localStorage กับ httpOnly Cookie
1. JS เข้าถึง httpOnly Cookie ไม่ได้ เลยปลอดภัยจาก XSS แต่ localStorage จะถูก JS เข้าถึงได้ เลยเสี่ยง XSS มากกว่า
2. httpOnly Cookie จะมีการให้ Browser แนบไปกับ Request อัตโนมัติ เลยจะเสี่ยงต่อ CSRF แต่ localStorage จะต้องเขียน code แนบเอง เลยปลอดภัยจาก CSRF มากกว่า

>[!Note] XSS and CSRF
> **XSS (Cross-Site Scripting)** คือการฝังโค้ด JavaScript เข้ามารันในหน้าเว็บของเรา ถ้า Token อยู่ใน localStorage โค้ดนั้นจะอ่านออกไปได้ทันทีด้วยคำสั่งบรรทัดเดียว แต่ถ้าอยู่ใน httpOnly Cookie จะอ่านไม่ได้เลย เพราะ Browser ปิดกั้นไม่ให้ JavaScript แตะต้อง
> 
> ---
> 
> **CSRF (Cross-Site Request Forgery)** คือการที่เว็บอื่นหลอกให้ Browser ยิง Request มาที่เว็บเรา ซึ่งเป็นปัญหาเฉพาะของ Cookie เพราะ Browser จะแนบ Cookie ไปให้อัตโนมัติโดยไม่สนว่า Request มาจากไหน 
> 
> ทางแก้คือตั้ง Attribute `SameSite=Strict` หรือ `Lax` เพื่อบอก Browser ไม่ให้แนบ Cookie ไปกับ Request ที่มาจากเว็บอื่น

แนวทางที่นิยมที่สุดในปัจจุบันคือผสม โดยเก็บ Access Token ไว้ใน Memory ของ JavaScript / เก็บไว้ใน State ของ React ซึ่งหายไปเมื่อ Refresh หน้าเว็บ และเก็บ Refresh Token ไว้ใน httpOnly Cookie พร้อม `SameSite` 

พอเปิดหน้าเว็บใหม่ Client จะใช้ Refresh Token ที่อยู่ใน Cookie ขอ Access Token ใบใหม่มาเก็บใน Memory ทันที วิธีนี้ทำให้ Token ที่มีอายุยาวไม่เคยถูก JavaScript แตะเลย ส่วน Access Token ที่อยู่ใน Memory ก็เสี่ยงน้อยเพราะอายุสั้นและไม่ถูกเก็บถาวร
  
---

## 6. Logout

Logout หมายถึงการลบ Refresh Token ออกจากฝั่ง Client และตั้งค่า `revoked_at` ใน Database ส่วน Access Token ที่ยังอยู่ในมือจะใช้งานต่อได้จนกว่าจะหมดอายุ ซึ่งเป็นข้อจำกัดที่ยอมรับได้เพราะอายุสั้นมากอยู่แล้ว ส่วนถ้าจะ logout จากทุกอุปกรณ์ ก็ทำได้ด้วยการยกเลิก Refresh Token ทุกอันที่ผูกกับ `user_id` นั้น

---