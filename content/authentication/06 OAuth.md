---
title: 06 OAuth
tags:
  - systemdesign
  - authentication
  - oauth
  - access_token
  - refresh_token
  - pkce
---
น่าจะเคยได้ยินคำว่า OAuth มาบ้างแหละ เช่น Google OAuth แต่เคยสงสัยมั้ยว่า OAuth คืออะไร? ถึงไม่สงสัยก็ไม่เป็นไร จะแพล่มอยู่ดี 

คือก่อนจะมี OAuth เนี่ย ปัญหาที่เกิดขึ้น คือ สมมติว่าเราจะเข้าใช้งานแอปหนึ่งที่ต้องการเข้าถึงรูปใน Google Photo สิ่งที่ต้องทำคือการเอา Email + Password ที่ใช้กับ Google มาใช้ในแอปนี้ด้วย อ่า พอเก็ตภาพยัง มันแปลว่าแอปนี้สามารถเก็บ Email + Password เราไปได้เต็มๆ ถ้าเราอยากถอดสิทธิ์จากแอปนี้ก็ต้องเปลี่ยน Password แล้วมันก็ไปกระทบแอปอื่นที่ใช้ Google Photo เหมือนกันงี้

OAuth ก็มาเพื่อการนี้แหละ เช่น Google OAuth ก็อนุญาตให้แอปสามารถเข้าถึง Google Photo ได้ โดยไม่ต้องให้ User มากรอก Email + Password ให้แอปเห็น และถ้า User อยากจะยกเลิกการเข้าถึงก็สามารถทำได้โดยไม่กระทบกับแอปอื่นๆ

---

## 1. OAuth Flow

![[auth_oauth_flow.png]]

Flow จะเป็นประมาณนี้ (เผื่อนึกภาพไม่ออกว่าเราใช้ตอนไหน มันคือไอ่หน้านี้อ่ะ... ก่อนจะเข้าหน้านี้ เราก็ต้องเลือก Email ที่จะใช้ก่อน)

![[auth_oauth_screen.png]]

ผมจะสมมติว่าแอปของเราชื่อ Rancher ละก็ Rancher นี้ต้องการเข้าถึง Google Photo ละกัน 
- **User** = Resource Owner (เจ้าของรูป)
- **Google Photos** = Resource Server (ที่เก็บรูป)
- **Google Login/Auth** = Authorization Server (คนออก Token)
- **Rancher** = Client (คนอยากได้รูป)

เพราะงั้น Flow ก็จะเป็นประมาณนี้
1. Rancher พา User ไปหน้า Login ของ Google ที่เชื่อมกับ Auth Server
2. User กดเลือก Email ที่จะ Login จากนั้นก็จะขึ้นหน้าข้อมูลว่า Rancher จะขอเข้าใช้งานข้อมูลของ Google อย่างไรบ้าง -> สมมติว่า User อนุญาตเลยละกัน Rancher Server ก็จะส่ง Client Secret ติดไปด้วยเพื่อบอกว่า Rancher Server เป็นคนขอ Code นี้
3. Auth Server ส่ง Authorization Code กลับมาให้ Rancher
4. Rancher Server นำ Authorization Code ไปแลก Access Token กับ Refresh Token กับ Auth Server โดยมี Client Secret ติดไปด้วย (กำหนดไว้ใน .env ของ Backend) เพื่อใช้ยืนยันตัวเอง
5. Auth Server คืน Access Token และ Refresh Token สำหรับให้ Rancher Server เข้าถึงข้อมูลใน Google Photo ได้ -> แล้ว Access Token กับ Refresh Token คืออะไร? อ่านต่อใน [[01 Access Token|Access Token]] กับ [[02 Refresh Token|Refresh Token]]
6. Rancher Server นำ Access Token ที่ได้ไปดึงข้อมูลจาก Google Photo ได้

>[!Note] แล้วทำไมต้องเป็น Code ก่อน ไม่คืน Access Token มาเลยอ่ะ
>คือใน Flow ที่เกิดการแลก Code อ่ะ มันอยู่บน Browser ของ Rancher ซึ่งมันอาจจะถูกเก็บเป็น Log แล้ว attacker เห็นแล้วเอาไปใช้ได้ แต่การเป็น Code ถึงดักไปก็เอาไปทำอะไรต่อไม่ได้
>
>พอได้เป็น Code มาแล้ว Connection หลังจากนี้มันจะเป็น Server to Server ไม่มี Browser แล้ว ทำให้ไม่สามารถ track ได้ ทำให้การแลก Code เป็น Access Token + Refresh Token ตรงนี้มีความปลอดภัย รวมถึงจะต้องมี Client Secret ที่ออกโดย Google ด้วยตอน Developer สร้าง App แล้วเก็บไว้ใน .env ของ Backend ทำให้ attacker จะสวมรอย Code ก็ไม่ได้

---

## 2. Mobile App No Backend Server

ปัญหาถัดมา คือ Mobile App มันไม่มี Backend Server เหมือน Web App เพราะงั้นมันจะไม่มี Area สำหรับเก็บ Client Secret จาก Google ไว้ แล้ว Mobile App มันทำยังไงให้ Google รู้ได้ว่าคือตนเอง

คำตอบคือการใช้ PKCE ของ Mobile App สร้าง Client Secret ขึ้นมาใหม่ทุกครั้งแทนการเก็บ Client Secret โดยจะมี String หนึ่งที่เรียกว่า `code_verifier` ติดมาด้วย จากนั้น เอา Client Secret ที่ Hash แล้วส่งไปให้ Auth Server เพื่อบอกว่า Mobile App นี้เป็นคนขอ Code

ทีนี้ พอจังหวะที่จะเอา Code มาแลก Token ก็ใช้ `code_verifier` กับ Code ที่ได้รับมา เพื่อให้ Auth Server รู้ได้ว่าเป็น Mobile App จริงๆ แล้วทำให้ Auth Server สามารถส่ง Access Token กับ Refresh Token กลับมาได้

---

## 3. OAuth ไม่ใช่ Authentication แต่เป็น Authorization

ตามนั้นเลย OAuth เป็นการให้ Consent กับ Application นั้นๆ ว่าสามารถเข้าใช้งาน Resource ได้ แต่เรื่องการยืนยันว่าเป็น Application จริงๆ มั้ยนั้นไม่ได้ทำตรงๆ 

ดูจากการกด "Login with Google" อ่ะ มันไม่ได้มี Process การเช็คว่าคนที่ใช้ Email ดังกล่าวคือเจ้าของ Email จริงๆ มั้ย เพราะใครจะมี Email ดังกล่าวก็ได้หมด attacker ก็มีได้ เพราะงั้น OAuth มันเลยมีหน้าที่แค่ Authorization ให้ App ใช้ Resource นั้นๆ ได้

แล้ว Authentication ทำยังไงดี? - คำตอบคือ OIDC (OpenID Connect) ที่จะทำงานบน OAuth อีกชั้นนึง เพื่อ authorize ว่าเป็น User จริงๆ -> ทำให้ Login with Google เกิดขึ้นมาได้จริงๆ

รายละเอียดเพิ่มเติมจะอยู่ใน [[07 OIDC|OIDC]] สามารถตามไปอ่านต่อได้

---



