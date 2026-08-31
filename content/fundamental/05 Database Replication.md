---
tags:
  - systemdesign
  - database
  - database_optimization
  - database_replication
---
![[database_replication.png]]

**Database Replication** คือการทำสำเนา (copy) ข้อมูลจาก Database หลักไปยัง Database อีกชุดหนึ่งหรือหลายชุด โดยมีหลักการทำงานคือ:
- เวลาข้อมูลมีการแก้ไข (Write/Update/Delete) จะแก้ไขที่ **Database หลัก** (เรียกว่า **Primary** หรือ **Master**) เท่านั้น
- หลังแก้ไขเสร็จ ข้อมูลจาก Primary จะถูกส่ง (**replicate**) ไปยัง Database สำเนาทั้งหมด (เรียกว่า **Replica** หรือ **Slave**)
- เวลาระบบต้องการ**อ่านข้อมูล** (Read) จะอ่านจาก Replica แทน ซึ่งมีได้หลายตัว

> [!note] รูปแบบนี้เรียกว่า **Primary-Replica Replication** (บางที่เรียก Master-Slave Replication) คือการ Write ทำที่จุดเดียว (Primary) แต่ Read กระจายไปหลายจุด (Replica) เพื่อลดภาระการอ่านข้อมูลที่ Primary ตัวเดียว

---

## 1. ข้อดี

1. **High Availability (ความพร้อมใช้งานสูง)** เพราะมีข้อมูลสำรองหลายชุด ถ้า Database ตัวใดตัวหนึ่งล่ม (ไม่ว่าจะ Primary หรือ Replica) ระบบยังเข้าถึงข้อมูลได้จากตัวที่เหลือ
2. **รองรับ Read แบบ Parallel ได้** เพราะสามารถเพิ่มจำนวน Replica ได้เรื่อยๆ เพื่อรองรับ Read Request ที่มากขึ้น (**Horizontal Scaling** ฝั่ง Read)
3. **ลด Latency สำหรับ User ที่อยู่ต่างพื้นที่** เพราะถ้าวาง Replica ไว้ใกล้ User ในแต่ละภูมิภาค (**Geo-distributed Replica**) การอ่านข้อมูลจะเร็วขึ้นเพราะไม่ต้องข้ามไปอ่านจาก Database ที่อยู่ไกล
4. **แยกภาระงานได้ชัดเจน (Separation of Concerns)** เช่น ให้ Replica บางตัวรับหน้าที่ทำ Reporting/Analytics (งานที่กิน Resource เยอะ) แยกจาก Replica ที่ให้ Application ใช้งานจริง เพื่อไม่ให้กระทบ Performance ของระบบหลัก

---

## 2. ข้อเสีย

1. **Replication Lag (ความหน่วงของการ Sync ข้อมูล)** เพราะต้องใช้เวลาในการ sync ข้อมูลให้ตรงกันระหว่าง DB Primary และ DB Replica ทำให้จะมีช่วงเวลาที่ข้อมูล 2 ที่นี้ไม่ตรงกัน เรียกปัญหานี้ว่า **Eventual Consistency** (ข้อมูลจะตรงกันในที่สุด แต่ไม่ใช่ทันที)
2. **เพิ่มความซับซ้อนของระบบ** เพราะต้องมีการจัดการเรื่องการ Sync ข้อมูลระหว่าง Primary-Replica เพิ่มอีกชั้น
3. **Primary เป็น Single Point of Failure สำหรับ Write** เพราะถ้า Primary ล่ม แม้ Read จะยังทำได้จาก Replica แต่ **Write ทำไม่ได้** จนกว่าจะมีการเปลี่ยนให้ Replica ตัวใดตัวหนึ่งเป็น Primary ตัวใหม่
4. **มีค่าใช้จ่ายเพิ่ม** จากค่า Server สำหรับ Replica แต่ละตัว
5. **Replication Conflict** เช่นถ้าใช้รูปแบบที่ Write บน DB ได้หลายที่ อาจเกิด **Write Conflict** ได้ จากการที่ข้อมูลชนกันเพราะถูกแก้จากหลายที่พร้อมกัน ทำให้ต้องมีวิธีจัดการ Conflict เพิ่ม

---

## 3. วิธีป้องกันปัญหาของ Database Replication

### 1. ป้องกันปัญหา Replication Lag

- เลือกใช้ **Synchronous Replication** (Primary รอ Replica ยืนยันว่าได้รับข้อมูลก่อนถึงจะถือว่า Write สำเร็จ) สำหรับข้อมูลที่ต้อง Consistency สูง แม้จะแลกกับ Latency ที่เพิ่มขึ้น
- ถ้าใช้ **Asynchronous Replication** (ไม่รอ Replica) ให้ Monitor ค่า Replication Lag อย่างสม่ำเสมอ และตั้ง Alert ถ้า Lag สูงเกิน Threshold ที่ยอมรับได้
- ออกแบบ Application ให้รู้ว่า Read บาง Use Case (เช่นข้อมูลที่เพิ่ง Write ไปหมาดๆ) ควรอ่านจาก Primary โดยตรงแทน Replica เพื่อเลี่ยงปัญหาอ่านข้อมูลเก่า

### 2. ป้องกัน Primary เป็น Single Point of Failure

- ตั้งระบบ **Automatic Failover** (การเลื่อน Replica ขึ้นเป็น Primary ใหม่อัตโนมัติเมื่อ Primary เดิมล่ม) ไว้ล่วงหน้า เช่นใช้เครื่องมืออย่าง Patroni, Orchestrator (สำหรับ MySQL/PostgreSQL) แทนการทำ Manual Failover ที่ช้าและเสี่ยง Human Error
- เลือก Replica ที่มี Lag น้อยที่สุดให้เป็นตัวสำรองที่จะถูกเลื่อนขึ้นก่อนเสมอ เพื่อลดโอกาสข้อมูลหาย

### 3. ป้องกันปัญหาจากการ Scale จำนวน Replica มากเกินไป

- Monitor ภาระที่ Primary ต้องส่งข้อมูลไปยัง Replica ทุกตัว เพราะยิ่งมี Replica มาก Primary ยิ่งต้องส่งข้อมูลมากขึ้นตาม อาจกลายเป็นคอขวดใหม่ได้
- พิจารณาทำ **Chained Replication** (ให้ Replica บางตัวไป Replicate ต่อให้ Replica ตัวอื่นอีกที แทนที่ Primary จะส่งตรงให้ทุกตัว) เพื่อลดภาระที่ Primary

### 4. ป้องกัน Write Conflict (กรณีใช้ Multi-Primary)

- ถ้าจำเป็นต้องใช้ Multi-Primary Replication ให้วางกลไกจัดการ Conflict ไว้ตั้งแต่ Design เช่น **Last-Write-Wins** (ยึดข้อมูลที่แก้ล่าสุดเป็นหลัก) หรือใช้ Timestamp/Version ควบคุมลำดับการ Merge ข้อมูล
- ถ้าไม่จำเป็นจริงๆ ควรเลือกใช้ Single-Primary Replication แทน เพราะจัดการง่ายกว่าและมีความเสี่ยงน้อยกว่า

---
