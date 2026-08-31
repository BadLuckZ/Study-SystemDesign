---
tags:
  - systemdesign
  - api
  - grpc
---
![[api_grpc.png]]

ทั้ง API และ gRPC คือวิธีที่ระบบ/Service ต่างๆ **สื่อสารกัน** ผ่าน Network แต่ต่างกันที่รูปแบบ Protocol และ Data Format:

- **REST API** สื่อสารผ่าน **HTTP** ทั่วไป ใช้ Verb มาตรฐาน (GET, POST, PUT, DELETE) ส่งข้อมูลส่วนใหญ่เป็น **JSON** (รูปแบบข้อมูลที่มนุษย์อ่านง่าย เป็น Text)
- **gRPC (Google Remote Procedure Call)** สื่อสารผ่าน **HTTP/2** ส่งข้อมูลเป็น **Protocol Buffers (Protobuf)** ซึ่งเป็นรูปแบบข้อมูลแบบ Binary ที่มีขนาดเล็กกว่าและแปลงข้อมูลเร็วกว่า JSON มาก

---

## 1.1. ข้อดีของ REST API

1. **เข้าใจง่าย/Debug ง่าย** เพราะข้อมูลเป็น JSON อ่านด้วยตาเปล่าได้เลย ทดสอบผ่าน Browser หรือเครื่องมืออย่าง Postman ได้ทันที
2. **รองรับโดย Browser โดยตรง** เพราะ Frontend เรียกใช้ได้เลยไม่ต้องผ่านตัวแปลงเพิ่ม
3. **Ecosystem/เครื่องมือเยอะ** มีมานาน มี Library, Documentation, Tool รองรับจำนวนมาก
4. **เหมาะกับ Public API** ที่ต้องให้ Developer ภายนอกเข้าใจและใช้งานได้ง่าย

## 1.2. ข้อเสียของ REST API

1. **Payload ใหญ่กว่า** JSON เป็น Text ทำให้ขนาดข้อมูลใหญ่กว่า Binary Format
2. **Performance ต่ำกว่าในงานที่ต้องการความเร็วสูง** เพราะ Parse ข้อมูลช้ากว่า Protobuf
3. **ไม่มี Contract บังคับ** อาจจะมีโครงสร้างข้อมูลที่ไม่ตรงกันระหว่าง Client-Server ถ้าไม่ทำ Documentation ดีพอ (แก้ด้วย OpenAPI / Swagger แต่ก็ไม่ได้บังคับตั้งแต่ระดับ Protocol)
4. **Streaming ทำได้จำกัด** เพราะต้องพึ่ง WebSocket หรือ Server-Sent Events เพิ่มเติมสำหรับ Real-time Communication

---

## 2.1. ข้อดีของ gRPC

1. **Performance สูง** เพราะ Protobuf มีขนาดเล็กกว่า JSON มาก และ Parse เร็วกว่า เหมาะกับระบบที่ต้องสื่อสารกันบ่อยและเร็ว เช่นการคุยกันระหว่าง [[10 Microservices|Microservices]]
2. **รองรับ Streaming ในตัว** สามารถส่งข้อมูลต่อเนื่องได้ทั้งสองทาง (Bidirectional Streaming) เหมาะกับ Real-time Data
3. **มี Contract บังคับผ่าน .proto file** ทำให้ Client-Server ต้องตรงกันตาม Schema ที่กำหนด ลดโอกาส Bug จากโครงสร้างข้อมูลไม่ตรงกัน
4. **Generate Code อัตโนมัติ** จาก .proto file สามารถ Generate Client/Server Code ในหลายภาษาได้ทันที ลดงาน Boilerplate (ลด code พื้นฐานที่ต้องเขียนซ้ำๆ ได้)

## 2.2. ข้อเสียของ gRPC

1. **Debug ยากกว่า** เพราะข้อมูลเป็น Binary อ่านด้วยตาเปล่าไม่ได้ ต้องใช้เครื่องมือเฉพาะ (เช่น grpcurl) ในการทดสอบ
2. **Browser เรียกตรงไม่ได้** ต้องผ่านตัวแปลงเพิ่มเช่น gRPC-Web ทำให้ Frontend ใช้งานยุ่งยากกว่า REST
3. **Ecosystem เล็กกว่า REST** มีเครื่องมือ/Documentation ยังน้อย
4. **ไม่เหมาะกับ Public API ทั่วไป** เพราะ Developer ภายนอกส่วนใหญ่คุ้นกับ REST มากกว่า และต้องมี .proto file แชร์กันก่อนถึงจะใช้งานได้

---

## 3. แนวทางเลือกใช้

### เมื่อไหร่ควรใช้ REST API

- Public API ที่ต้องให้ Developer ภายนอกเข้าใจง่าย
- ระบบที่ Frontend (Browser) ต้องเรียกใช้โดยตรง
- ระบบที่ Performance ไม่ใช่ปัจจัยหลัก และต้องการ Debug/Maintain ง่าย

### เมื่อไหร่ควรใช้ gRPC

- การสื่อสารภายในระหว่าง Service ต่อ Service (Internal Communication) ใน [[10 Microservices|Microservices]] ที่ต้องการ Performance สูง
- ระบบที่ต้องการ Streaming Data แบบ Real-time
- ระบบที่ต้องการ Contract ที่ชัดเจนระหว่างทีมต่างๆ ผ่าน .proto file

### แนวทางผสม

- หลายระบบเลือกใช้ **gRPC สำหรับการสื่อสารภายใน** (Service-to-Service) และใช้ **REST API สำหรับ Public-facing** (Client ภายนอก/Browser) เพื่อได้ประโยชน์ทั้งสองแบบพร้อมกัน

---
