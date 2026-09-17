---
title: 04 Cookie
tags:
  - systemdesign
  - authentication
  - cookie
---
ย้อนกลับไปในเรื่อง Token คือปัญหาของ HTTP คือมันไม่มี Memory มันไม่สามารถจำได้ว่า Request นี้มาจากใคร เพราะงั้นเราเลยมี Concept ที่ทำให้ HTTP รู้ได้ว่าใครเป็นส่ง Request นี้มา 

สิ่งนั้นเราเรียกว่า Cookie 

Cookie ไม่ใช่ชื่อขนม แต่ว่าเป็น Area ของ Browser ที่มีหน้าที่เก็บข้อมูลของ User โดยความพิเศษของ Cookie คือการส่ง Request ทุกครั้งจะมีการส่งข้อมูลใน Cookie ไปด้วยเสมอ ทำให้ Developer ไม่ต้องจัดการเรื่องการส่งข้อมูลจาก Client ไปหา Server เพราะ Browser จัดการให้แล้ว

![[auth_cookie_flow.png]]

---

## 1. Set-Cookie

วงจร Cookie มันเริ่มจากเมื่อ User login สำเร็จแล้ว Server ส่ง Response กลับมา มันจะมี Header ตัวนึงที่ชื่อว่า `Set-Cookie` มาด้วย

```http
HTTP/1.1 200 OK
Set-Cookie: session_id=abcdefg; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
```

พอ Browser เห็น `Set-Cookie` แล้ว ก็จะเก็บค่าไว้ใน Cookie ของ Browser จากนั้นพอ User มีการส่ง Request ต่อหลังจากนี้ มันก็จะแนบ Cookie ไปใน Request Header แบบอัตโนมัติ

```http
GET /api/profile
Cookie: session_id=abcdefg
```

Server ก็จะรู้ว่า อ๋อ Request นี้มาจาก Session นี้นะ ก็เอาข้อมูล Session ไปประมวลผลแล้วส่ง Response กลับมาได้อย่างเหมาะสม

แล้ว... Set-Cookie มันมีข้อมูลอะไรบ้างอ่ะ?

1. `HttpOnly`: Attribute ที่ป้องกันไม่ให้ JavaScript อ่าน Cookie ได้ ทำให้ป้องกัน XSS ที่เป็นการยัด script เข้ามาในหน้า Web เพื่อขโมยข้อมูล ก็จะขโมยสิ่งที่อยู่ใน Cookie ไปไม่ได้
2. `Secure`: บังคับให้ Browser ส่ง Cookie ผ่าน HTTPS เท่านั้น
3. `SameSite`: จะยอมให้แนบ Cookie นี้ไปกับ Request ที่มาจาก Web อื่นหรือไม่
	- `Strict` - ไม่ให้แนบ
	- `Lax` - ยอมในบางกรณี
	- `None` - แนบหมดทุกกรณี (ต้องมี Secure ไปด้วย)
4. `Domain` กับ `Path`: กำหนดขอบเขตว่า Cookie จะแนบไปกับ Domain และ Path ไหนได้บ้าง
5. `Max-Age` หรือ `Expires`: กำหนดอายุของ Cookie ถ้าไม่ตั้งไว้ก็จะหายไปเมื่อ User ปิด Browser

แต่ว่าๆ ไอ่การส่ง Cookie ไปกับ Browser แบบอัตโนมัติได้เนี่ย มันก็เป็นทั้งข้อดี ที่ Developer ไม่ต้องจัดการเอง และข้อเสียที่เสี่ยงโดน CSRF อ่ะนะ

---

## 3. XSS กับ CSRF

ก่อนหน้านี้เคยมีการพูดถึง XSS กับ CSRF ไปแล้ว แต่เดะจะ recap อีกซักรอบละกัน

XSS (Criss-Site Scripting) คือการที่ attacker ฝัง JavaScript Code เข้ามาในหน้าเว็บของเรา เพื่ออ่านข้อมูลใน Cookie -> ทางแก้ก็คือการกำหนด `HttpOnly` ใน Cookie เพื่อไม่ให้ JavaScript อ่านได้นี่แหละ

CSRF (Cross-Site Request Forgery) คือการใช้ลักษณะของ Browser ที่จะแนบ Cookie เข้าไปใน Request ทุกครั้ง ผ่านการหลอกให้ Browser ยิง Request ไปหา Web ปลอมของตน เพื่อที่จะได้ Cookie มา -> ทางแก้ก็คือ `SameSite` นี่แหละ ที่เป็นการบอก Browser ว่าไม่ต้องส่ง Cookie นี้ให้ Web อื่น

---