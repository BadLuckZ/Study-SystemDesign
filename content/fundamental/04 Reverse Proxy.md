---
tags:
  - systemdesign
  - proxy
  - load_balancer
  - reverse_proxy
---
![[reverse_proxy.png]]

**Reverse Proxy** คืออุปกรณ์ตัวกลาง ที่อยู่ฝั่ง **Server** ทำหน้าที่รับ Request จาก Client ทั้งหมดก่อน แล้วส่งต่อไปยัง Server จริงที่อยู่ข้างหลังมัน ทำให้ Client จะไม่รู้ว่า Server จริงที่ตอบกลับมาคือตัวไหน

> [!note] มุมมองง่ายๆ Reverse Proxy = "ตัวแทนของฝั่ง Server" ทำหน้าที่ปกปิด/จัดการ Server ที่อยู่ข้างหลังมัน (ตรงข้ามกับ [[03 Forward Proxy|Forward Proxy]])

---

## 1. หน้าที่หลัก

- **ปกปิด IP ของ Server จริง** เพราะ Client ไม่รู้ว่ามี Server กี่ตัว หรือ Server ตัวไหนตอบ
- **SSL Termination** — Encrypt และ Decrypt รหัส HTTPS แทน Server จริง
- เป็น Load Balancer ไปในตัว เช่น Nginx
- **Caching** โดยเก็บ Response ที่ Server เคยตอบไว้ เมื่อเจอว่า Request เป็นตัวเดิมในช่วงเวลาที่กำหนด ก็จะคืน Response นี้กลับไปเลย ลดภาระ Server จริง
- **Security** โดยการเป็นด่านหน้า กันการโจมตี Server จริง (เช่น Rate Limiting)

---

## 2. ตัวอย่างการใช้งานจริง

- Nginx / HAProxy / Traefik หน้า Web Server
- CDN: Content Delivery Network (เช่น Cloudflare)
- API Gateway ในสถาปัตยกรรม Microservices

---

## 3. ข้อดี

1. **Security เพิ่มขึ้น** เพราะมีการซ่อน Structure และ IP ของ Server จริงจากภายนอก
2. **Scalability** เพราะเราสามารถเพิ่ม/ลด Server ข้างหลัง Proxy ได้โดย Client ไม่รู้ตัว
3. **SSL/TLS จัดการที่จุดเดียว** เพียงแค่ทำ Encryption Decryption ที่ Proxy ตัวเดียวก็พอ ไม่ต้อง setup ในทุก Server
4. **เพิ่ม Performance** ผ่าน Caching

---

## 4. ข้อเสีย

1. **Single Point of Failure** เพราะ ถ้า Reverse Proxy ล่ม Client เข้าถึง Server ไม่ได้
2. **เพิ่มความซับซ้อนของระบบ** เพราะต้อง Config, Monitor Proxy เพิ่ม
3. **เพิ่ม Latency เล็กน้อย** เพราะผ่านอีกชั้นก่อนถึง Server จริง

---

## 5. วิธีป้องกันปัญหาของ Reverse Proxy

### 1. ป้องกัน Single Point of Failure

- ใช้ **Cloud-managed Reverse Proxy/CDN** (บริการที่ผู้ให้บริการ Cloud ดูแล Redundancy ให้เอง เช่น Cloudflare) ที่มี Redundancy ระดับ Global อยู่แล้วตั้งแต่ต้น

### 2. ป้องกัน Overload

- ออกแบบให้ Scale Out ได้ตั้งแต่แรก (เพิ่มจำนวน Reverse Proxy แล้วกระจายด้วย DNS Round Robin)
- เปิด Caching, Compression ตั้งแต่ต้นเพื่อลดภาระที่ต้องส่งไป Server จริง
- ทำ Load Testing

### 3. ป้องกัน Routing/Config ผิดพลาด

- Review Config การ Route แบบ **path-based** หรือ **host-based** ทุกครั้งก่อน Deploy โดยเฉพาะหลัง Deploy ใหม่ หรือใช้ Automated Config Testing

### 4. ป้องกันปัญหา SSL/TLS Certificate

- ตั้ง Monitoring แจ้งเตือนล่วงหน้าก่อน Certificate หมดอายุ และใช้ **Auto-renew** (ระบบต่ออายุ Certificate อัตโนมัติ) ตั้งแต่ต้นเพื่อไม่ให้ลืมต่ออายุ

### 5. ป้องกันการตกเป็นเป้าโจมตี (DDoS/Bot)

- เปิดใช้ **Rate Limiting** (จำกัดจำนวน Request ต่อช่วงเวลาต่อ Client)

---

