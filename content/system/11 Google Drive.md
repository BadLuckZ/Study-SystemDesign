---
title: 11 Google Drive
tags:
  - system
  - systemdesign
  - google_drive
  - database_replication
  - sharding
  - block_server
  - notification_system
  - polling
  - amazon_s3
---
วันนี้เราจะมา discuss กันในระบบ Google Drive ซึ่งเป็นระบบสำหรับ upload file ไปเก็บไว้บน Cloud แล้ว share file เหล่านั้นได้อย่างสะดวกสบาย จะมีส่วนสำคัญอะไรยังไงบ้าง มาดูกัน

![[googledrive_intro.png]]

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: ระบบนี้จะ focus ที่ feature อะไรบ้างเหรอครับ?
A1: User ควรที่จะ upload และ download file ได้ ละก็เห็น file ที่ตรงกัน (พร้อมเก็บประวัติการแก้ไข file) รวมถึงมี noti in-app คอยแจ้ง user ว่า upload สำเร็จหรือไม่ 

Q2: แล้ว focus ที่ platform ไหนบ้าง? mobile, web หรือทั้งคู่เหรอครับ?
A2: ทั้งคู่ๆ

Q3: User สามารถ upload file แบบไหนได้บ้างเหรอครับ?
A3: file สกุลไหนก็ได้ ระบบเราควรจะเก็บไว้ได้หมด

Q4: แล้วใน storage จริงๆ ที่เก็บ file ของ user ทุกคนนี่ต้อง encrypt มั้ยครับง
A4: ต้องนะ

Q5: แล้ว...จะมีการกำหนดขนาดไฟล์มั้ยนะ?
A5: มีๆ เอาไม่เกิน 10 GB ละกัน

Q6: ละระบบนี้มีผู้ใช้งานเยอะขนาดไหนเหรอครับ?
A6: ก็ราวๆ 10 ล้าน DAU (Daily Active Users)

---

## 2. Sketch ภาพระบบคร่าวๆ

รอบนี้เราอาจจะมาในแนวทางที่แปลกไปจากเดิมหน่อยนะ เพราะเราจะเริ่มผ่านการออกแบบด้วย server ตัวเดียว จากนั้นจึงค่อย scale เพื่อให้รองรับ user เยอะๆ ได้ละกัน

### API Examples

ก่อนอื่นเลย API จะมีหน้าตาประมาณไหนนะ?
หลักๆ ก็คงมี 3 เส้น คือ การ upload file, download file ละก็ดูประวัติการแก้ไข files

>[!Caution] ระบบนี้ไม่ได้ลงลึกเรื่องการแก้ไข File ได้แบบ Real Time เช่นการแก้ไข text ใน Google Doc นะครับ

#### 1. Upload File API

การ upload file จะมี 2 แบบ
1. upload แบบปกติ - ใช้สำหรับ upload file ที่มีขนาดเล็ก
2. upload แบบ resume ได้ - ใช้สำหรับ upload file ที่มีขนาดใหญ่ซึ่งเสี่ยงกว่าที่จะ upload file ไปเก็บไว้ไม่สำเร็จ

หน้าตา API ก็เป็นประมาณว่า 
```swagger
https://api.example.com/files/upload
```

โดยจะมี params เพิ่มเข้าไป 2 ตัว
1. uploadType: simple / resumable
2. data: local path ของ file ที่จะ upload

#### 2. Download File API

หน้าตา API ก็ประมาณว่า
```swagger
https://api.example.com/files/download
```

โดยจะมี params เพิ่มเข้าไป 1 ตัว
1. path: path ใน google drive ของ file ที่จะ download

#### 3. File Revision API

หน้าตา API ก็ประมาณว่า
```swagger
https://api.example.com/files/list_revisions
```

โดยจะมี params เพิ่มเข้าไปอีก 2 ตัว
1. path: path ใน google drive ของ file ที่จะดูประวัติการแก้ไข
2. limit: จำนวนครั้งการแก้ไขสูงสุด นับจากการแก้ไขครั้งล่าสุด

### Scalability

แน่นอนว่าถ้าเราปล่อยระบบทิ้งไว้เรื่อยๆ มันก็จะถึงจุดที่ Server ของเราเนื้อที่เต็มอ่ะนะ มันก็ต้องมาคำนึงต่อละว่าจะ scale ระบบนี้ยังไงดี

![[googledrive_highlevel_design.png]]

ปัญหาหลักๆ ของระบบนี้จะอยู่ที่การทำยังไงให้เก็บไฟล์ของ user ได้อย่างมีประสิทธิภาพ ท่าที่ง่ายสุดคือการใช้ Amazon S3 เป็น File Storage ละก็มี Database แยก ที่เก็บ Metadata ของ File ที่ user upload ขึ้นมา เพื่อรอให้ user เรียกใช้งานตอนจะ download หรือดูประวัติการแก้ไข

แล้ว...Amazon S3 มันช่วยยังไงนะ - Amazon S3 จะมีการทำ Replication หรือการสำรอง File โดยจะมีทั้ง Bucket ที่อยู่ใน Region เดียวกันจำนวนหลายถังเพื่อเก็บ File นี้ รวมถึง Bucket ที่อยู่คนละ Region อีกหลายถังเพื่อเก็บ File ไว้ เพื่อป้องกัน Data Loss ละก็ทำให้ระบบมี High Availability

![[googledrive_replication.png]]

แต่ว่าก็ยังมีปัญหาอยู่ คือในการเก็บข้อมูลหลักอ่ะ สุดท้ายมันก็จะไปกระจุกอยู่ที่ Database หลักตัวเดียว แม้จะมี Replication แต่ Replication เหล่านั้นก็จะเก็บ File เหล่านั้นไว้เต็มเหมือน Database หลักด้วย

วิธีการคือ เราอาจจะต้องทำ Sharding คือมี Bucket หลักหลายๆ อันที่ทำหน้าที่เก็บข้อมูลหลัก เช่น
- user_id % 4 = 1 ให้เก็บใส่ Storage 1
- user_id % 4 = 2 ให้เก็บใส่ Storage 2
- user_id % 4 = 3 ให้เก็บใส่ Storage 3
- user_id % 4 = 0 ให้เก็บใส่ Storage 0

ทีนี้แต่ละ Storage ก็จะเก็บไฟล์น้อยลง มีเนื้อที่เหลือเยอะขึ้น ละก็ให้ Replication ก๊อปข้อมูลตรงนี้ไปสำรองข้อมูล อีกทั้งการ query ข้อมูลก็ทำได้เร็วขึ้นด้วย

---

## 3. เจาะลึกจุดสำคัญๆ

### Block Servers

อย่างที่เราได้พูดไปก่อนหน้านี้ใน API ว่า การ upload file จะแบ่งเป็น 2 แบบ และที่เป็นปัญหาอยู่คือแบบที่ user upload file ที่มีขนาดใหญ่เข้ามา เราจะจัดการ file ตรงนั้นยังไงก่อนที่จะเอาเก็บใน storage

![[googledrive_blocking_server.png]]

วิธีการนึงที่ทำได้คือ การใช้ Block Server ที่จะ split file ออกเป็น block เล็กๆ จากนั้นก็ compress แล้ว encrypt block files เหล่านั้นแล้วเก็บไว้ใน storage

ฟังดูไม่ได้มีความแตกต่างจากการเก็บ file แบบปกติเลยป่ะ ก็ในมุมปกติก็ใช่แหละ แต่ว่าจะมี feature นึงที่มันดีขึ้นมามากกว่าการเก็บแบบปกติเลย คือเมื่อ User แก้ไฟล์แล้ว save

ปกติแล้ว User ก็แก้แค่รายละเอียดบางส่วนของ File เนอะ รายงานมี 10 หน้าก็อาจจะแก้แค่ 1-2 หน้า ไรงี้ การทำเป็น Block Server นี้จะช่วยให้ระบบสามารถบันทึกการเปลี่ยนแปลงแค่เฉพาะส่วนที่มีการเปลี่ยนแปลงจริงๆ ได้ ส่วนไหนที่ไม่ได้เปลี่ยนก็ไม่ต้องทำอะไร เทียบกับการเก็บทั้ง File คือถ้ามีหน้านึงเปลี่ยนก็คือมี Change ใน File นั้น ทำให้ระบบต้องบันทึกใหม่หมด ไรงี้ 

### Metadata Database

![[googledrive_metadata.png]]

Metadata Database ที่จะใช้ คือมีประมาณนี้...
- User เก็บข้อมูล user ที่ upload ไฟล์นี้เข้ามา
- Workspace เก็บข้อมูล workspace ที่ใช้สำหรับเก็บไฟล์ที่ user upload เข้ามา เช่น my_workspace ไรงี้ ฟีลๆ Folder / Directory ใหญ่
- File เก็บข้อมูลไฟล์ที่ user upload เข้ามา
- File Version เก็บประวัติการแก้ไข file
- Block เก็บข้อมูลแต่ละส่วนของ File ของ Version นั้นๆ
- Device เก็บข้อมูลของ device ที่ user ใช้ upload ไฟล์เข้ามา

### File Upload Flow

![[googledrive_upload_file_flow.png]]

ข้อมูลที่ User upload เข้ามาจะถูกแบ่งออกเป็น 2 ส่วน คือ Metadata กับตัว File จริงๆ
1. Part ของ Metadata ก็จะผ่านเข้า API Servers แล้วเก็บไว้ใน Metadata DB ด้วยสถานะ Pending จนกว่าการ upload file จะเสร็จสิ้น
2. Part ของ File ก็จะผ่าน Block Server แล้วเก็บ Blocks เหล่านั้นไว้บน Cloud
3. ระบบจะกลับมาแก้สถานะ Pending ของ Metadata นี้ให้เป็น Success / Fail จากนั้นก็แจ้งเตือน User ว่า Upload สำเร็จ หรือว่าล้มเหลว ผ่าน Notification Service

### File Download Flow

![[googledrive_download_file_flow.png]]

เมื่อ User จะ download file ก็จะมี noti บอกอยู่ว่า download อยู่ จากนั้นก็เช็ค metadata database เพื่อดึง metadata ล่าสุดของ file ที่กำลัง download มาให้ User จากนั้นก็ดึง blocks ตาม metadata ที่ได้รับมา แล้วประกอบร่างกันใน Block Servers เป็น File เต็ม คืนเป็นผลลัพธ์ให้ User ไป

### Notification Service

Notification Service ในระบบนี้ทำหน้าที่หลักอยู่ 2 อย่าง:
- **แจ้งผล Upload/Download** - บอก User ว่า Upload/Download File สำเร็จหรือล้มเหลว
- **แจ้งว่ามีคนอื่นแก้ไฟล์ที่ User กำลังเปิดดูอยู่** เพื่อบอกว่าไฟล์นี้มี Version ใหม่แล้ว ควร Refresh Page เพื่อดูข้อมูลล่าสุดนะ

Notification Service สามารถใช้ได้ทั้ง 2 แบบ จะ Long Polling ก็ได้ หรือว่า WebSocket ก็ได้เหมือนกัน แต่ในกรณีนี้เชียร์ Long Polling มากกว่า ด้วย 2 เหตุผล
- ไม่ได้ต้องการการสื่อสารแบบ 2 ทาง ส่วนใหญ่เป็นการส่งข้อมูลจาก Server ไปหา Client
- ไม่ใช่สถานการณ์ที่ต้องการ Real-time ระดับต่ำมาก ๆ แบบ Chat หรือ Online Game

![[googledrive_longpolling.png]]

โดยหลักการของ Long Polling คือ Client จะส่ง Request ไปหา Server แล้ว Server จะยังไม่ตอบกลับทันที แต่จะถือ Connection นี้เอาไว้ก่อนเพื่อรอว่ามี Event ใหม่เกิดขึ้นหรือไม่

เช่น User A เปิด File ค้างไว้ แล้ว User B แก้ไฟล์เดียวกันแล้ว Save 
1. Block บางส่วนของไฟล์มีการเปลี่ยนแปลง 
2. เกิด File Version ใหม่ใน Metadata DB หรือก็คือเกิด Event

เมื่อ Server ตรวจพบ Event นี้ ก็จะส่งข้อมูลกลับไปบอก User A ว่า "ไฟล์นี้มี Version ใหม่แล้วนะ" ทำให้ Connection นั้นจบลง จากนั้น Client จะส่ง Request ใหม่ไปหา Server ทันที เพื่อรอ Notification ครั้งถัดไป (Notification สนใจแค่ระดับ File/Version ว่าเปลี่ยนหรือยัง ไม่ได้ลงรายละเอียดว่า Block ไหนเปลี่ยนบ้าง)

อ่านรายละเอียดได้ใน [[06 Notification System|Notification System]]

### How to Save Storage Space

ไม่มีอะไรมาก ด้วยความที่เราทำข้อมูลเป็น Block เล็กๆ ทำให้มี Block ใน Storage เยอะ ละมันก็อาจจะมี Change แค่ไม่กี่ Block รวมถึงมีทั้ง Block เก่า Block ใหม่อีก เลยต้องมีการจัดการเรื่องนี้เพื่อ Save เนื้อที่ใน Storage แต่จะทำยังไงได้บ้างนะ...

1. เวลามีการ save file ถ้า Block ไหนไม่มี Change ก็ไม่ต้องเก็บ Block นั้นซ้ำ เก็บเฉพาะ Block ที่มี Change ก็พอ
2. จำกัดว่า Block นั้นจะเก็บไว้สูงสุดกี่ Version เพื่อให้ประวัติการแก้ไขของ Block นั้นไม่ล้นเกินไป เช่น คืนแค่ 20 Version ก็แปลว่า เราทำระบบให้เก็บได้ 20 Version / Block ก็พอ
3. Block ไหนที่ไม่ได้มีการแก้ไขนานๆ ก็โยกไป Cold Storage เช่น Amazon S3 Glacier ที่มี Cost เบากว่า Amazon S3 เป็นต้น

จาก Discussion ทั้งหมด System Design ที่ได้จะเป็นประมาณนี้นั่นเอง
![[googledrive_lowlevel_design.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Google Drive ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---

