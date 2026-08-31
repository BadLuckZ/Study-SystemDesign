---
tags:
  - systemdesign
  - microservices
  - api
  - grpc
  - message_queue
---
![[microservices.png]]

**Microservices** คือแนวทางออกแบบสถาปัตยกรรมระบบที่แยกระบบใหญ่หนึ่งระบบออกเป็น **Service ย่อยๆ หลายตัว** โดยแต่ละ Service จะเป็นแบบนี้...

- ทำงานเฉพาะด้าน (เช่น Service จัดการ User, Service จัดการ Order, Service จัดการ Payment) แบ่งตาม **Business Capability**
- มี Database เป็นของตัวเอง แยกจาก Service อื่น (**Database per Service**)
- Deploy และ Scale ได้อิสระจาก Service อื่น
- สื่อสารกันผ่าน Network เช่น [[11 API vs gRPC|REST API หรือ gRPC]] หรือผ่าน [[09 Message Queue|Message Queue]]

ตรงข้ามกับแนวทางดั้งเดิมที่เรียกว่า **Monolith** (สถาปัตยกรรมแบบก้อนเดียว) ที่รวม Logic ทุกอย่างไว้ใน Codebase เดียว, Deploy พร้อมกันทั้งหมด, ใช้ Database ร่วมกันตัวเดียว

---

## 1. องค์ประกอบใน Microservices

- **API Gateway** - จุดเดียวที่ Client เรียกเข้ามา แล้วกระจาย Request ไปยัง Service ที่ถูกต้อง (คล้าย [[04 Reverse Proxy|Reverse Proxy]] ที่รู้จัก Service ต่างๆ ในระบบ)
- **Service Discovery** - กลไกที่ทำให้ Service รู้ว่า Service อื่นอยู่ที่ไหน (เพราะ Service อาจถูก Scale/ย้ายตำแหน่งตลอดเวลา)
- **Container/Orchestration** - มักใช้ **Docker** (เทคโนโลยี Container ที่แพ็ค Service พร้อม Dependency ไว้ด้วยกัน) และ **Kubernetes** (เครื่องมือจัดการ Container จำนวนมากให้ Deploy/Scale/Monitor อัตโนมัติ)

---

## 2. ข้อดีของ Microservices

1. **Scale แยกส่วนได้อิสระ** เพราะ Service ไหนที่ Traffic เยอะ (เช่น Order Service ช่วง Sale) Scale เฉพาะตัวนั้นได้ ไม่ต้อง Scale ทั้งระบบ
2. **Deploy อิสระ (Independent Deployment)** เพราะสามารถแก้/เพิ่ม Feature ใน Service หนึ่ง Deploy ได้โดยไม่กระทบ Service อื่น ลด Risk และเพิ่มความถี่ในการ Release
3. **เลือกเทคโนโลยีต่างกันได้ตาม Service (Polyglot)** เพราะแต่ละ Service เลือกภาษา/Database ที่เหมาะกับงานตัวเองได้ ไม่ต้องใช้ Stack เดียวกันทั้งระบบ
4. **แยกทีมพัฒนาได้ชัดเจน** โดยแต่ละทีมดูแล Service ของตัวเอง ทำงานคู่ขนานกันได้โดยไม่ต้องรอกัน (**Team Autonomy**)
5. **Fault Isolation (จำกัดผลกระทบเมื่อเกิดปัญหา)** เพราะถ้า Service หนึ่งมีปัญหา ระบบส่วนอื่นยังทำงานต่อได้ (ถ้าออกแบบดี) ต่างจาก Monolith ที่ Bug จุดเดียวอาจทำให้ทั้งระบบล่ม

---

## 3. ข้อเสียของ Microservices

1. **เพิ่มความซับซ้อนของ Infrastructure มาก** เพราะต้องดูแล Service จำนวนมาก, Network Communication ระหว่าง Service, API Gateway, Service Discovery เป็นต้น
2. **Debug/Trace ปัญหายาก** ในกรณีที่ Request หนึ่งวิ่งผ่านหลาย Service กว่าจะจบ 
3. **Data Consistency ข้าม Service ยาก** เพราะแต่ละ Service มี Database แยกกัน การทำ Transaction ที่ครอบคลุมหลาย Service (เช่น หัก Stock พร้อม Confirm Payment) ทำได้ยากกว่า Monolith ที่ใช้ Database เดียวและพึ่ง Transaction ปกติได้เลย
4. **Network Latency เพิ่มขึ้น** เพราะ Service ต้องคุยกันผ่าน Network แทนที่จะเรียก Function ภายใน Codebase เดียวกันแบบ Monolith

---

## 4. วิธีป้องกันปัญหาของ Microservices

### 1. ป้องกันปัญหา Data Consistency ข้าม Service

- ใช้แนวคิด **Saga Pattern** (การแบ่ง Transaction ใหญ่เป็นหลายขั้นตอนย่อย พร้อมกำหนดขั้นตอน "ย้อนกลับ" หรือ Compensating Action ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว) แทนการพึ่ง Transaction แบบ Monolith
- ยอมรับแนวคิด **Eventual Consistency** (จะมีการ update ข้อมูลให้ up-to-date แต่จะไม่ได้เกิดในทันที) ในหลายจุด แล้วออกแบบ UX ให้รองรับความล่าช้าที่อาจเกิดขึ้น

### 2. ป้องกันปัญหา Debug/Trace ยาก

- ตั้ง **Distributed Tracing** (เช่น Jaeger, Zipkin) และ **Centralized Logging** (รวม Log จากทุก Service ไว้ที่เดียว) ไว้ตั้งแต่ Design เพื่อตามปัญหาข้าม Service ได้ง่ายขึ้น
- ใส่ **Correlation ID** (รหัสเดียวกันที่แนบไปกับ Request ตลอดทางที่มันวิ่งผ่านหลาย Service) เพื่อเชื่อมโยง Log จากทุก Service ที่เกี่ยวข้องกับ Request เดียวกัน

### 3. ป้องกันปัญหา Service ล่มแล้วลามไป Service อื่น (Cascading Failure)

- ใช้ **Circuit Breaker Pattern** (กลไกที่ "ตัดวงจร" การเรียก Service ที่กำลังมีปัญหาชั่วคราว แทนที่จะปล่อยให้ Request รอจนหมดเวลาซ้ำๆ) เพื่อป้องกันไม่ให้ปัญหาของ Service หนึ่งลามไปกระทบ Service อื่น
- ตั้ง **Timeout** และ **Retry with Backoff** (การลองใหม่โดยเว้นระยะเวลาเพิ่มขึ้นเรื่อยๆ แทนที่จะยิงซ้ำถี่ๆ) ให้เหมาะสมทุกจุดที่มีการเรียกข้าม Service

### 4. ป้องกันความซับซ้อนของ Infrastructure บานปลาย

- เริ่มจาก Monolith ก่อนถ้าระบบยังเล็ก แล้วค่อยแตกเป็น Microservices เมื่อระบบโตขึ้นจริงและมีเหตุผลชัดเจน (หลีกเลี่ยงการทำ Microservices ตั้งแต่ Day 1 โดยไม่จำเป็น ซึ่งเป็นปัญหาที่พบบ่อย)
- ใช้เครื่องมือ Orchestration อย่าง Kubernetes ที่มี Community/Documentation รองรับดี แทนการทำระบบจัดการ Container เอง

---

