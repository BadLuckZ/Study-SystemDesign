---
tags:
  - systemdesign
  - message_queue
---
![[message_queue_point2point.png]]
![[message_queue_pubsub.png]]

**Message Queue (MQ)** คือระบบตัวกลางที่ทำหน้าที่รับ **Message** (ข้อมูล/งานที่ต้องการให้ประมวลผล) จากฝั่งที่ส่ง (**Producer**) แล้วเก็บไว้ใน **Queue** รอให้ฝั่งที่รับ (**Consumer**) มาดึงไปประมวลผลทีหลัง แทนที่ Producer จะเรียก Consumer โดยตรงแบบ Synchronous (ต้องรอผลลัพธ์ก่อนถึงจะทำงานต่อ)

> [!note] แนวคิดหลัก Message Queue ทำให้ Producer กับ Consumer **"ไม่ต้องคุยกันโดยตรง"** และ**ไม่ต้องพร้อมทำงานพร้อมกัน** (Asynchronous Communication) - Producer ส่ง Message แล้วทำงานต่อได้เลยโดยไม่ต้องรอ Consumer ประมวลผลเสร็จ

---

## 1. รูปแบบการทำงาน

- **Point-to-Point (Queue)**: Message หนึ่งชิ้นถูกประมวลผลโดย Consumer เพียงตัวเดียว (ถึงจะมี Consumer หลายตัว แต่ Message หนึ่งชิ้นไปหาแค่ตัวใดตัวหนึ่ง) เหมาะกับงานที่ต้องทำแค่ครั้งเดียว เช่น ประมวลผล Order
- **Publish/Subscribe (Pub/Sub)**: Message หนึ่งชิ้นถูกส่งไปให้ Consumer **ทุกตัว** ที่ Subscribe (สมัครรับ) Topic นั้นไว้ เหมาะกับงานที่ต้องแจ้งเตือนหลายระบบพร้อมกัน เช่น เมื่อมี Order ใหม่ ให้ทั้งระบบ Email, ระบบ Inventory, ระบบ Analytics รับรู้พร้อมกัน

---

## 2. ตัวอย่างเครื่องมือ

- **RabbitMQ** - เน้น Message Queue แบบ Traditional รองรับทั้ง Point-to-Point และ Pub/Sub
- **Apache Kafka** - เน้น Event Streaming (การส่ง Event จำนวนมากแบบต่อเนื่อง) รองรับ Throughput สูงมาก นิยมใช้ในระบบ Log/Analytics ขนาดใหญ่
- **AWS SQS (Simple Queue Service)** - Managed Message Queue บน Cloud ที่ไม่ต้องดูแล Infrastructure เอง

---

## 3. ข้อดีของ Message Queue

1. **Decoupling (แยกส่วนของระบบออกจากกัน)** เพราะ Producer กับ Consumer ไม่ต้องรู้จักกันโดยตรง สามารถเปลี่ยน/เพิ่ม Consumer ได้โดยไม่กระทบ Producer
2. **Asynchronous Processing** เพราะ Producer ส่ง Message แล้วทำงานต่อได้ทันที ไม่ต้องรอ Consumer ประมวลผลเสร็จ ทำให้ Response Time ของ User เร็วขึ้น (เช่น กด Checkout แล้วได้รับคำตอบทันที ส่วนการส่ง Email ยืนยันทำเบื้องหลัง)
3. **รองรับ Scale ของ Consumer ได้อิสระ** เพราะสามารถเพิ่มจำนวน Consumer ได้ตามปริมาณงานใน Queue โดยไม่กระทบ Producer
4. **Load Leveling / Buffering** (การรองรับ Traffic ที่มาไม่สม่ำเสมอ) เพราะถ้า Message เข้ามาเยอะพร้อมๆ กันก็จะถูกเก็บไว้ใน Queue แล้วรอให้ Consumer ค่อยๆ ดึงไปประมวลผลได้
5. **Retry ได้ง่าย** เพราะถ้า Consumer ประมวลผล Message ไม่สำเร็จ สามารถส่ง Message กลับเข้า Queue ให้ลองใหม่ได้ โดยไม่ต้องให้ Producer ส่งซ้ำ

---

## 4. ข้อเสียของ Message Queue

1. **เพิ่มความซับซ้อนของระบบ** เพราะต้อง Deploy, Config, Monitor ระบบ Message Queue เพิ่ม
2. **Eventual Consistency** (ข้อมูลจะตรงกัน แต่ไม่ตรงกันในทันที) เพราะเป็น Asynchronous ผลลัพธ์ของงานอาจไม่เสร็จทันทีที่ Producer ส่ง Message
3. **Message Ordering ซับซ้อน** เพราะบางระบบต้องการให้ Message ถูกประมวลผลตามลำดับที่ส่งมา (เช่น Event ของ Order เดียวกันต้องเรียงลำดับ) ซึ่งต้อง Config เพิ่มเติม
4. **Duplicate Message** เพราะบาง MQ รับประกันแค่การส่งถึง Consumer อย่างน้อยหนึ่งครั้ง แปลว่าอาจเกิดการส่ง Message เดียวกันซ้ำได้
5. **Message Queue as Single Point of Failure** เพราะถ้า Queue ล่มหรือเต็ม อาจกระทบทั้งระบบที่พึ่งพา Asynchronous Processing ได้

---

## 5. วิธีป้องกันปัญหาของ Message Queue

### 1. ป้องกัน Message Queue เป็น Single Point of Failure

- ใช้ Message Queue ที่รองรับ **Clustering/Replication** ในตัว (เช่น Kafka ที่มี Replication ของแต่ละ Partition) หรือใช้ **Managed Service** (เช่น AWS SQS) ที่ Provider ดูแล Redundancy ให้

### 2. ป้องกันปัญหา Duplicate Message

- ออกแบบ Consumer ให้เป็น **Idempotent** ตั้งแต่แรก เช่น เช็ค Message ID ที่เคยประมวลผลไปแล้วก่อนทำงานซ้ำ

### 3. ป้องกันปัญหา Message Ordering

- ถ้าต้องการลำดับที่แน่นอน ให้ใช้ฟีเจอร์ที่ MQ รองรับ (เช่น Kafka Partition Key ที่การันตีลำดับภายใน Partition เดียวกัน) และออกแบบ Key ให้ Message ที่เกี่ยวข้องกันไปอยู่ Partition เดียวกันเสมอ

### 4. ป้องกันปัญหา Queue ล้น (Backlog สะสม)

- Monitor ความยาวของ Queue (จำนวน Message ที่รอประมวลผล) เป็นระยะ ตั้ง Alert ถ้า Backlog สูงเกิน Threshold
- ออกแบบให้ Consumer Scale Out ได้ง่าย (เช่น เพิ่มจำนวน Consumer อัตโนมัติตามความยาวของ Queue)

### 5. ป้องกัน Message ที่ประมวลผลไม่สำเร็จค้างอยู่ตลอด

- ตั้ง **Dead Letter Queue (DLQ)**: Queue พิเศษที่เก็บ Message ที่ล้มเหลวซ้ำๆ เกิน Limit ที่กำหนด (เช่น Retry ครบ 5 ครั้งแล้วยังไม่สำเร็จ) แยกออกมาตรวจสอบภายหลัง แทนที่จะปล่อยให้วนซ้ำไม่จบ

---
