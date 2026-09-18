---
title: 07 CDN (Content Delivery Network)
tags:
  - database_optimization
  - systemdesign
  - cdn
  - fundamental
---
![[content_delivery_network.png|689]]

**CDN (Content Delivery Network)** คือเครือข่ายของ Server ที่กระจายตัวอยู่ตามพื้นที่ต่างๆ ทั่วโลก (เรียกแต่ละจุดว่า **Edge Server**) ทำหน้าที่เก็บสำเนาของ Content ต่างๆ เช่น รูปภาพ, วิดีโอ, ไฟล์, หน้าเว็บทั้งหน้า) ไว้ใกล้ๆ กับตำแหน่งของ User เพื่อให้ User ดึงข้อมูลจาก Edge Server ที่อยู่ใกล้ตัวเองที่สุด แทนที่จะต้องวิ่งไปหา **Origin Server** (Server ต้นทางจริงที่เก็บข้อมูลตัวจริง) ที่อาจอยู่ไกลคนละทวีป

> [!note] มุมมองง่ายๆ CDN คือ [[04 Reverse Proxy|Reverse Proxy]] ที่กระจายอยู่ทั่วโลก โดยเน้นเก็บ Cache ของ Content ไว้ใกล้ User เพื่อลดระยะทางและเวลาที่ข้อมูลต้องเดินทาง

---

## 1. หน้าที่หลัก

- เก็บสำเนาไฟล์ Static (รูป, วิดีโอ, JS, CSS) ไว้ที่ Edge Server ใกล้ User ที่สุด
- CDN หลายเจ้า (เช่น Cloudflare) มีระบบป้องกัน **DDoS** (การโจมตีด้วยการยิง Traffic จำนวนมหาศาลเข้าระบบพร้อมกันจนล่ม) และ **WAF** (Web Application Firewall) ในตัว เพราะ Traffic ทั้งหมดวิ่งผ่าน CDN ก่อนถึง Origin เป็น Security อีกชั้นให้ระบบ

---

## 2. ข้อดี

1. **ลด Latency ได้** เพราะ User ดึงข้อมูลจาก Edge Server ที่ใกล้ที่สุดแทนที่จะวิ่งไป Server ไกลๆ
2. **ลดภาระ Origin Server** เพราะ Request ส่วนใหญ่จะเข้าถึง และถูกตอบโดย Edge Server แทน
3. **เพิ่ม Availability / Reliability** เพราะถ้า Origin Server มีปัญหาชั่วคราว ก็จะยังใช้ Edge Server บางเจ้าส่ง Content แทนได้อยู่
4. **รองรับ Traffic Spike ได้ดีขึ้น** เพราะ Load กระจายไปตาม Edge Server หลายจุดทั่วโลก ไม่กระจุกที่ Origin เดียว
5. **ป้องกันการโจมตีได้ในตัว** เพราะมี DDoS Protection, WAF, Rate Limiting ที่ระดับ Edge ก่อนถึง Origin จริง

---

## 3. ข้อเสีย

1. **Data Staleness (ข้อมูลที่ Edge Server เก่ากว่า Origin)** เพราะถ้า Content ที่ Origin เปลี่ยนไปแล้ว แต่ Edge Server ยังไม่ได้อัปเดต Cache ผู้ใช้บางคนอาจเห็นข้อมูลเก่าอยู่
2. **มีค่าใช้จ่ายเพิ่ม** จากค่าบริการ CDN Provider ตามปริมาณ Traffic ที่ใช้
3. **เพิ่มความซับซ้อนของระบบ** เพราะต้อง Config เรื่อง Cache Rule, TTL เพิ่มอีกชั้น
4. **Debug ยากเวลามีปัญหา** เช่น User บางคนเห็นข้อมูลผิด) ต้องตรวจสอบว่าปัญหาอยู่ที่ Origin หรือ Edge Server ตัวไหน

---

## 4. วิธีป้องกันปัญหาของ CDN

### 1. ป้องกัน Data Staleness

- ตั้ง **TTL** ให้เหมาะกับความถี่ที่ Content เปลี่ยน เช่น ไฟล์ที่แทบไม่เปลี่ยน (Font, Library) ตั้ง TTL ยาวได้ ส่วนไฟล์ที่เปลี่ยนบ่อยควรตั้ง TTL สั้น
- อัพเดท Cache อัตโนมัติ ทุกครั้งที่ Deploy Content เวอร์ชันใหม่

### 2. ป้องกันปัญหาด้าน Cost ที่บานปลาย

- Monitor การใช้ Traffic ผ่าน CDN เป็นระยะ 
- ตั้ง Budget Alert ล่วงหน้า
- Cache เฉพาะ Content ที่คุ้มค่าจริงๆ (Static Content ที่ถูกเรียกซ้ำบ่อย) ไม่ Cache Content ที่ไม่ได้ประโยชน์ เพื่อไม่ให้เสีย Cost โดยไม่จำเป็น

### 3. ป้องกัน Debug ยาก

- ตั้ง Logging ที่ Edge Server (ถ้า CDN Provider รองรับ) เพื่อแยกได้ว่า Request ไหนตอบจาก Cache (Cache Hit) หรือวิ่งไป Origin จริง (Cache Miss) ช่วยตามปัญหาได้ง่ายขึ้น
- ทำ Documentation เกี่ยวกับ Cache Rule ไว้ล่วงหน้า ให้รู้ว่า Path ไหน Cache ยังไง TTL เท่าไหร่

### 4. ป้องกันปัญหากับ Dynamic / Personalized Content

- แยก Content ให้ชัดเจนตั้งแต่ Design ว่าส่วนไหน Static (ให้ CDN Cache เต็มที่) ส่วนไหน Dynamic (ให้ผ่านตรงไป Origin หรือ Cache แบบสั้นมากๆ)
- ถ้าจำเป็นต้อง Cache Content ที่ Personalize บางส่วน ให้พิจารณาใช้เทคนิค **Edge Computing** (รัน Logic บางส่วนที่ Edge Server เอง เช่น Cloudflare Workers) แทนการวิ่งกลับไป Origin ทุกครั้ง

---
