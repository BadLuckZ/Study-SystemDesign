---
title: 03 Forward Proxy
tags:
  - systemdesign
  - proxy
  - forward_proxy
---
![[forward_proxy.png]]

**Forward Proxy** คืออุปกรณ์ตัวกลาง ที่อยู่ในฝั่ง **Client** และรับ Request จาก Client แล้วส่งต่อไปยัง Server ปลายทางแทน Client อีกที ทำให้ Server ปลายทางจะไม่รู้ว่า Client ตัวจริงคือใคร

> [!note] Forward Proxy = "ตัวแทนของฝั่ง Client" ทำหน้าที่ปกปิด/จัดการ Client ที่อยู่ข้างหลังมัน

---

## 1. หน้าที่หลัก

- **ปกปิด IP ของ Client** เพราะ Server ปลายทางเห็นแค่ IP ของ Proxy ไม่เห็น Client จริง
- **Caching** โดยการเก็บ Response ที่เคยขอจาก Request นั้นไว้ เวลาที่ User ขอ Request เดิมในช่วงนั้นก็คืน Response ที่เก็บไว้เลย
- **Anonymity Restriction** ใช้ซ่อนตัวตนหรือเข้าถึงเนื้อหาที่ถูกจำกัดตามภูมิภาค

---

## 2. ตัวอย่างการใช้งานจริง

- Proxy ขององค์กร/โรงเรียน ที่กรอง Content หรือบล็อกเว็บบางประเภท
- VPN บางเจ้า
- Proxy สำหรับ Web Scraping เพื่อสลับ IP ไปยัง Domain อื่น ไม่ให้ User โดน Block

---

## 3. ข้อดี

1. **ความเป็นส่วนตัว** เพราะมีการซ่อน IP จริงของ Client จาก Server ปลายทาง
2. **ควบคุม/กรองการเข้าถึง** เพราะเราสามารถกำหนดการเข้าถึง Content ต่างๆ ของ IP ที่เชื่อมกับ Proxy นี้ได้ (ดีต่อ IT ขององค์กร)
3. **ลด Bandwidth** ด้วย Caching เนื้อหาที่ถูกเรียกซ้ำๆ

## 4. ข้อเสีย

1. **Single Point of Failure** เพราะ ถ้า Proxy ล่ม Client ทุกตัวที่เชื่อมกับ Proxy จะไม่เชื่อม Internet อีก
2. **เพิ่ม Latency** เพราะ Request ต้องผ่านอีกชั้นหนึ่ง คือ Proxy นี่แหละ
3. **Privacy ขึ้นอยู่กับผู้ดูแล Proxy** เพราะผู้ดูแล Proxy สามารถเห็นพฤติกรรมของ Client ได้ทั้งหมด
4. **ต้อง Config ที่ฝั่ง Client** คือต้องมีการระบุตัวตนให้ Proxy รู้จักก่อนอ่ะ จึงจะเชื่อมกับ Proxy ได้

---

## 5. วิธีป้องกันปัญหาของ Forward Proxy

### 1. ป้องกัน Single Point of Failure

- ตั้งกลุ่ม Proxy หลายตัวที่ทำงานร่วมกัน แล้วระบุให้ Client รู้ว่าใช้ Proxy ตัวไหน ถ้า Proxy ตัวนี้พัง ให้ไปใช้ Proxy ตัวไหนเลือกใช้ผ่าน Load Balancer

### 2. ป้องกัน Overload

- ออกแบบให้ Scale Out ได้ง่าย (เพิ่ม Proxy Server แล้วกระจาย Client ไปแต่ละตัวได้ทันที)
- เปิดใช้ Caching ตั้งแต่แรกเพื่อลด Traffic ที่ต้องออกไป Internet จริง

### 3. ป้องกัน IP โดน Block/Blacklist

- วางแผนการสลับเปลี่ยน IP ที่ใช้ใน Internet เป็นระยะ แทนที่จะใช้ IP เดิมตลอด
- แยก Proxy ตามกลุ่มการใช้งาน เพื่อไม่ให้พฤติกรรมของ Client กลุ่มหนึ่ง เช่น จะทำ Action A ให้ Client ไปเชื่อม Proxy A นะ ไรงี้

### 4. ป้องกัน Filtering Rule บล็อกผิดที่ (False Positive)

- Review Whitelist/Blacklist เป็นระยะ และเช็คว่ากฎที่ตั้งเหมาะสมจริงๆ ก่อน Deploy จริงทุกครั้ง เพื่อลดโอกาสบล็อก Traffic ที่ควรผ่านได้

### 5. ป้องกันปัญหา Privacy/Compliance

- กำหนดนโยบายการเก็บ Log ให้ชัดเจนตั้งแต่ต้น (เก็บนานแค่ไหน ใครเข้าถึงได้) เพื่อไม่ให้ผิดกฎหมาย/นโยบายข้อมูลส่วนบุคคล (เช่น PDPA) ตั้งแต่ Design

---
