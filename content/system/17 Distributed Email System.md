---
title: 17 Distributed Email System
tags:
  - system
  - systemdesign
  - distributed_email_system
  - email
  - email_protocols
  - database_replication
  - elastic_search
---

ห่างหายไปนาน... วันนี้มาเป็น Email System ละกัน เช่นพวก Outlook, Gmail หรือเก่าๆ หน่อยก็ Yahoo ว่า implement กันยังไง

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Email System นี้จะมีคนใช้งานเยอะแค่ไหนเหรอครับ?
A1: ประมาณ 1 Billion Users ละกัน

Q2: Email System นี้ผมจะ Focus Features ต่อไปนี้ก่อนนะครับ

- รับส่ง Emails
- ดึงข้อมูล Emails ทั้งหมด
- Filter Emails ผ่าน read/unread status
- Search Emails ผ่าน Subject, คนส่ง, เนื้อหา
- Anti-spam
  มีอะไรที่ควรเพิ่มมั้ยครับในเบื้องต้น
  A2: โอเคละนะ ไม่มีละ

Q3: แล้ว Users ต่อกับ Mail Servers ยังไงเหรอครับ?
A3: ชีวิตจริงจะ Connect ผ่าน SMTP, POP3, IMAP อ่ะนะ แต่ในที่นี้เอาเป็น HTTP ละกัน

> [!Tips] Email Protocols
>
> - SMTP (Simple Mail Transfer Protocol) คือ โปรโตคอลสำหรับส่ง Email จาก User ไปยัง Mail Server หรือส่งต่อ Email ระหว่าง Mail Servers
> - POP3 (Post Office Protocol 3) คือ โปรโตคอลสำหรับ รับ/ดาวน์โหลด Email จาก Mail Server มาไว้ที่เครื่องของ User โดยทั่วไปเน้นการดาวน์โหลดเมลมาเก็บที่เครื่อง
> - IMAP (Internet Message Access Protocol) คือ โปรโตคอลสำหรับ เข้าถึงและจัดการ Email บน Mail Server โดยเมลยังคงอยู่บน Server ทำให้สามารถเปิดดูและ Sync จากหลายอุปกรณ์ได้

Q4: แล้วมี Attachment Files ได้มั้ยครับ
A4: ได้นะ

### Non-Functional Requirements

- Reliability - ไม่ควรมี Emails ที่หายไป
- Availability - Emails ควรจะมีการ replicated ไว้ในหลายๆ Nodes เพื่อป้องกัย System Failures
- Scalability - ระบบสามารถ handle จำนวน Users ที่เพิ่ม/ลดเองได้อย่างเหมาะสม และ Performance ของระบบไม่ drop ลง
- Flexibility - เราสามารถดัดแปลง Email Protocol เพื่อให้เหมาะสมกับการใช้งานของระบบเราได้

---

## 2. Sketch ภาพระบบคร่าวๆ

โจทย์เราคือ Distributed Email System เนอะ แปลว่าเราต้องคิดถึง Traditional Mail Server ก่อนว่าหน้าตาเป็นประมาณไหน

![[email_system_traditional.png]]

Traditional Mail Servers จะมีหน้าตาประมาณนี้

1. Alice ต้องการจะส่งเมลให้ Bob โดย Alice ก็จะส่งเมลผ่าน Outlook ทำให้ Email ที่ Alice ส่งจะถูกส่งไปให้ SMTP Server
2. SMTP Server ของฝั่ง Alice ก็เก็บ Email ที่ Alice ส่งไว้ ละก็ส่ง Email ให้ SMTP Server ของฝั่ง Bob
3. SMTP Server ของฝั่ง Bob เก็บ Email ที่ได้ไว้ใน Storage
4. IMAP/POP3 Server ดึง Email จาก Storage มาแสดงให้ Bob เห็น

ซึ่ง Storage จะเก็บ Email ไว้เป็น Directory เหมือน Directory ในคอมอ่ะ

![[email_system_storage.png]]

ในเมื่อหน้าตาของ Traditional Mail Servers เป็นงี้ละ เรามา discuss ต่อเรื่อง Distributed Mail Servers กันต่อ

### Email APIs

1. **POST** /v1/messages -> ส่ง message ไปหาคนอื่นๆ
2. **GET** /v1/folders -> ดึง folders ทุกอันออกมา

```swagger
{
	id:      string    # folder id
	name:    string
	# folder name (All, Archive, Drafts, Flagged, Junk, Sent or Trash)
	user_id: string    # account owner reference
}[]
```

3. **GET** /v1/folders/:folder_id/messages -> ดึง messages ทั้งหมดใน folder ออกมา

```swagger
{
	id:      string                          # message id
	user_id: string                          # account owner reference
	from:    {name: string, email: string}   # <name, email> of sender
	to:      {name: string, email: string}   # <name, email> of receiver
	subject: string                          # email subject
	body:    string                          # email body
	is_read: boolean                         # this message is read or not
}[]
```

4. **GET** /v1/messages/:message_id -> ดึง message ออกมา

### Email Sending Flow

![[email_system_distributed_sending.png]]

1. User เขียน Email แล้วส่ง ทำให้ Email ที่ส่งมาถูกส่งไปยัง Load Balancer
2. Load Balancer เช็ค Rate Limit แล้วส่งต่อให้ Web Servers ที่พร้อมทำงาน
3. Web Servers validate Email รวมถึงเช็คขนาดไฟล์ว่าเกินที่กำหนดมั้ย และเช็คว่า Domain ที่ส่ง Email ตรงกับ Email ของ Sender หรือไม่ หากเช็คแล้วผ่านหมดก็จะนำ Email เก็บใส่ Database ประเภทต่างๆ เช่น
   - Metadata Database สำหรับเก็บ Metadata ของ Email
   - Search Database สำหรับให้ระบบออกแบบ เพื่อให้ค้นหา Email ได้เร็วขึ้น
   - Object Database สำหรับเก็บ Files ที่ attach มาใน Email
   - Cache สำหรับบันทึก Email ไว้ในระยะเวลานึง
4. เมื่อบันทึก Email ไว้ใน Storage แล้ว ก็จะนำ Message ไปเก็บไว้ใน Queue
   - ถ้า 3. เช็คแล้วผ่านหมดก็จะเก็บ Email ไว้ใน Outgoing Queue
   - ถ้า 3. เช็คละไม่ผ่านก็จะเก็บ Email ไว้ใน Error Queue
5. SMTP Workers ดึง Email ใน Outgoing Queue ออกมา
6. SMTP Worker เอา Email ที่จะส่งไปเก็บใน Directory "Sent Folder"
7. เมื่อเก็บใน Directory แล้ว ก็จะส่ง Emaiil ไปยัง Mail Server ของผู้รับผ่าน Internet

### Email Receiving Flow

![[email_system_distributed_receiving.png]]

1. Email ที่ถูกส่งมาจะเข้า Load Balancer
2. Load Balancer เช็ค Rate Limit แล้วส่งต่อให้ SMTP Web Servers ที่พร้อมทำงาน
3. SMTP Web Server เก็บ Attachment ใน Email ไว้ใน Object Store Database
4. SMTP Server ส่ง Email ไปเก็บไว้ใน Incoming Queue
5. Mail Processing Workers ดึง Emails ใน Incoming Queue ไปเช็คว่าเป็น Spam มั้ย หรือเป็น Virus มั้ย รวมถึง validate email ในประเด็นอื่นๆ
6. Email จะถูกนำไปเก็บไว้ใน Storage ต่างๆ ทั้ง Metadata Database, Search Store Database, Object Store Database, และ Cache
7. ถ้าผู้รับ Email Online อยู่ Workers ก็จะส่ง Emails ต่อไปให้ Real-Time Servers
8. Real-Time Servers ส่ง Email ผ่าน Websocket ไปให้ Webmail ให้ User เห็น
9. ถ้าผู้รับ Email Offline อยู่ก็เก็บ Email ไว้ใน Storage จนพอผู้รับ Email Online ปุ๊ป Web Client ก็จะ trigger Web Servers ผ่าน REST APIs
10. Web Servers จะไปดึง Emails ทั้งหมดที่ยังไม่เคยแสดงให้ User เห็นออกมาจาก Database

---

## 3. เจาะลึกส่วนที่สำคัญๆ

### Metadata Database

ก่อนจะเลือก Database ต้องเข้าใจลักษณะของข้อมูล Email ก่อน ซึ่งมีลักษณะอยู่ 5 ข้อ

- Header มีขนาดเล็ก แต่ถูกเรียกใช้บ่อย
- Body มีขนาดตั้งแต่เล็กไปจนใหญ่มาก แต่ถูกเปิดอ่านไม่บ่อย ปกติอ่านแค่ครั้งเดียว
- Operation ส่วนใหญ่ผูกกับ User คนเดียว เช่น ดึง Email, mark ว่าอ่านแล้ว, ค้นหา Email
- ข้อมูลใหม่ถูกใช้มากกว่าข้อมูลเก่า eg. เราไม่ค่อยสนใจ Email เก่าๆ อยู่แล้วป่ะ
- ห้ามมีข้อมูลหายเด็ดขาด

ทีนี้มาดู Database Option ที่มีกันบ้าง

| ตัวเลือก                 | ปัญหา                                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Relational DB            | เหมาะกับข้อมูลชิ้นเล็ก แต่ Email มักจะมีขนาดใหญ่ (หลาย KB ถึงหลัก 100 KB) แต่ถ้าเก็บเป็น BLOB ก็ค้นหาได้ไม่มีประสิทธิภาพ |
| Object Storage (เช่น S3) | เหมาะเป็น Backup <br>แต่ทำ Feature อย่าง mark read, ค้นหา, จัด Thread ได้ยาก                                             |
| NoSQL                    | Gmail ใช้ Bigtable จึงเป็นทางที่เป็นไปได้ <br>แต่ไม่ได้ Open Source                                                      |

ก็ว่าง่ายๆ ว่าไม่มี Option อะไรที่จะทำได้เลย เลยจะจบที่การสร้าง Database ขึ้นมาเอง ซึ่งจะมีลักษณะที่สำคัญดังต่อไปนี้

- Column เดียวเก็บข้อมูลได้ระดับหลาย MB
- Strong Consistency
- ออกแบบมาเพื่อลด Disk I/O
- High Availability และ Fault-tolerant
- ทำ Incremental Backup ได้ง่าย

#### Data Model

เราจะใช้ `user_id` เป็น **Partition Key** เพื่อให้ข้อมูลของ User คนเดียวกันอยู่บน Shard เดียวกัน ซึ่งเข้ากับลักษณะที่ Operation ส่วนใหญ่ผูกกับ User คนเดียวพอดี

> [!Note] Partition Key กับ Clustering Key
>
> - **Partition Key** ใช้กระจายข้อมูลไปยัง Node ต่างๆ ควรเลือกให้กระจายได้สม่ำเสมอ
> - **Clustering Key** ใช้เรียงลำดับข้อมูลภายใน Partition เดียวกัน

ทีนี้มาดูว่า Query หลักที่ระบบต้องรองรับมีอะไรบ้าง แล้วออกแบบตารางตามนั้น

**Query 1: ดึง Folder ทั้งหมดของ User**
ใช้ `user_id` เป็น Partition Key ทำให้ Folder ของ User คนเดียวกันอยู่ใน Partition เดียวกัน

![[email_system_folders_by_user.png]]

**Query 2: แสดง Email ทั้งหมดใน Folder**
ปกติกล่องเมลจะเรียงจากใหม่ไปเก่า จึงใช้ `<user_id, folder_id>` เป็น Composite Partition Key และใช้ `email_id` ที่เป็น TIMEUUID (ID ที่มีเวลาฝังอยู่) เป็น Clustering Key เพื่อเรียงตามเวลา

![[email_system_emails_by_folder.png]]

**Query 3: สร้าง/ลบ/ดึง Email**
ใช้ตาราง `emails_by_user` เก็บรายละเอียดของ Email และแยกตาราง `attachments` ไว้ เพราะ Email หนึ่งฉบับมีไฟล์แนบได้หลายไฟล์

![[email_system_emails_by_user.png]]

**Query 4: ดึง Email ที่อ่านแล้ว / ยังไม่อ่าน**
ถ้าเป็น Relational DB ก็แค่ `WHERE is_read = true` แต่ NoSQL ส่วนใหญ่ Query ได้เฉพาะ Partition Key กับ Clustering Key เท่านั้น ซึ่ง `is_read` ไม่ได้เป็นทั้งคู่

ทางแก้คือ **Denormalization** แยกเป็น 2 ตาราง คือ `read_emails` กับ `unread_emails` เวลา mark ว่าอ่านแล้ว ก็ลบออกจาก `unread_emails` แล้วเพิ่มเข้า `read_emails` แทน

![[email_system_read_unread_emails.png]]

> [!Tip] Consistency vs Availability
> ระบบที่ Replicate ข้อมูลไว้หลาย Node ต้องเลือกระหว่าง Consistency กับ Availability
> สำหรับ Email ความถูกต้องสำคัญกว่า จึงให้แต่ละกล่องเมลมี **Primary เพียงตัวเดียว**
> เพราะงั้น...ถ้า Primary ล่มและกำลังสลับไปตัวสำรอง
> กล่องเมลนั้นจะเข้าถึงไม่ได้ชั่วคราว ยอมเสีย Availability เพื่อรักษา Consistency

### Email Deliverability

การตั้ง Mail Server แล้วส่งเมลออกไปไม่ยาก ที่ยากคือทำให้เมล **ไปถึง Inbox** ไม่ใช่ตกไปอยู่ใน Spam เพราะ Email ที่ส่งกันทั่วโลก มากกว่าครึ่งเป็น Spam ละ Mail Server ใหม่ที่ยังไม่มีชื่อเสียงก็มักจะโดนจับเป็น Spam ซะด้วย

วิธีเพิ่มโอกาสให้เมลถึง Inbox มีดังนี้

- **Dedicated IPs**: ใช้ IP เฉพาะสำหรับส่งเมลที่รู้แล้วว่ามันจะไม่ตกไปเป็น Spam เพราะพอใช้ IP ใหม่ก็อาจจะตกไปเป็น Spam ก็ได้
- **แยกประเภท Email**: ส่ง Email การตลาดกับ Email สำคัญ ด้วยคนละ IP กัน ไม่งั้นอาจจะโดนเหมารวมว่าเป็นโฆษณาทั้งหมด
- **Warm Up IP**: ค่อยๆ เพิ่มปริมาณการส่งจาก IP เดิม เพื่อทำให้ Mail Server ยอมรับ ละไม่ปัดไปเป็น Spam
- **Feedback Processing**: รับ Feedback จาก ISP มาจัดการ แยกเป็น 3 Queue
  - **Hard Bounce**: ส่งไม่ถึงเพราะที่อยู่ผู้รับไม่มีอยู่จริง
  - **Soft Bounce**: ส่งไม่ถึงเพราะเหตุชั่วคราว เช่น ISP ปลายทางยุ่งอยู่
  - **Complaint**: ผู้รับกดรายงานว่าเป็น Spam
    	![[email_system_feedback_loop.png]]

### Search

การค้นหา Email ต่างจากการค้นหาบน Google ค่อนข้างมาก เช่น

- Google จะค้นหาทั้ง Internet ในขณะที่ Email จะหาแค่จาก Email Inbox ของตัวเอง
- Google จะ sort ข้อมูลตามความเกี่ยวข้องกับ Query แต่ Email จะ sort ข้อมูลตาม Attribute ต่างๆ ที่ User เลือก เช่น เวลา, มีไฟล์แนบ, ยังไม่อ่าน
- Google Index ทำงานช้าได้ ผลไม่ขึ้นทันทีก็ไม่เป็นไร แต่ Email Index ต้องทำงานเร็วมาก ต้องเกือบ Real-time และแม่นยำมากเลยด้วยซ้ำ

อีกจุดที่ต่างคือ Email Search มี **Write มากกว่า Read** เพราะทุกครั้งที่ส่ง รับ หรือลบเมล ต้อง Reindex ใหม่ แต่การค้นหาจริงเกิดเฉพาะตอน User กดปุ่ม Search เท่านั้น

เพราะงั้นการ Search ที่แนะนำสำหรับ Email เลยจะมี 2 เทคนิค

##### Option 1: Elasticsearch

![[email_system_elasticsearch.png]]

ในระบบ Elasticsearch นั้น...

- เมื่อ User ค้นหา Email ระบบก็ต้องตอบกลับในทันที
- แต่การ Reindex ตอนส่ง รับ หรือลบเมลไม่จำเป็นต้องตอบ User ทันที จึงโยนเข้า Kafka แล้วให้ Consumer ไป Reindex ทีหลังได้
  ปัญหาหลักคือต้องคอย Sync ข้อมูลระหว่าง Database หลักกับ Elasticsearch ให้ตรงกันตลอด

##### Option 2: Custom Search Engine

![[email_system_lsm_tree.png]]

ผู้ให้บริการรายใหญ่มักสร้าง Search Engine ของตัวเอง ซึ่งจะมีปัญหาหลักๆ ที่ **Disk I/O** เพราะข้อมูลเพิ่มขึ้นระดับ Petabyte ต่อวัน และกล่องเมลหนึ่งกล่องอาจมีเมลหลายแสนฉบับ

เนื่องจากการสร้าง Index เน้นการเขียนเป็นหลัก จึงนิยมใช้ **LSM Tree (Log-Structured Merge-Tree)** ซึ่งเขียนข้อมูลใหม่ลง Memory ก่อน (Level 0) พอเต็มถึงขีดที่กำหนดค่อย Merge ลงไปเก็บใน Disk ระดับถัดไป ทำให้การเขียนเป็นแบบ Sequential ที่เร็วกว่า LSM Tree เป็นโครงสร้างหลักของ Database อย่าง Bigtable, Cassandra และ RocksDB

### Scalability

เนื่องจากพฤติกรรมการใช้งานของ User แต่ละคนแยกจากกัน Component ส่วนใหญ่ในระบบจึง Scale แบบ Horizontal ได้ตรงๆ

แต่เพื่อ Availability ที่ดีขึ้น ข้อมูลจะถูก Replicate ไปหลาย Data Center โดย User จะเชื่อมต่อกับ Data Center ที่อยู่ใกล้ที่สุด ถ้า Data Center นั้นมีปัญหา User ก็ยังเข้าถึงเมลผ่าน Data Center อื่นได้ (อ่านเพิ่มเติมได้ใน [[05 Database Replication|Database Replication]])

![[email_system_multi_data_center.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Distributed Email System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
