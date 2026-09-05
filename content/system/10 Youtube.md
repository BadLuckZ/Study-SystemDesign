---
title: 10 Youtube
tags:
  - system
  - systemdesign
  - youtube
  - cdn
  - blob
  - message_queue
---
วันนี้เราจะมา discuss กันในเรื่องของ Streaming Platform ประเภทนึงที่ได้รับความนิยมอย่าง Youtube ที่เป็น Platform สำหรับดู Video ต่างๆ นั่นเอง บนความสะดวกสบายและใช้งานง่าย เบื้องหลังมีการทำงานอะไรยังไงบ้าง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: อยาก focus feature ไหนเป็นหลักอ่ะครับ
A1: การ upload video และดู video ได้ละกัน อย่างอื่นค่อยว่ากัน

Q2: จะให้ใช้งานบน Devices อะไรได้บ้าง
A2: ใช้ได้ทั้ง Mobile, Web ละก็ Smart TV ด้วย

Q3: คิดว่ามีผู้ใช้งานประมาณไหน
A3: ราวๆ 5,000,000 DAU (Daily Active Users) ละกัน

Q4: แล้วแต่ละคนจะใช้งาน product นานขนาดไหนอ่ะๆ
A4: ราวๆ 30 นาทีต่อคนละกัน

Q5: แล้วจะให้รองรับการใช้งานจาก users ต่างประเทศมั้ยนะ
A5: ควรนะ

Q6: แล้วรองรับ Video แบบไหนได้บ้างอ่ะ ความละเอียดเท่าไหร่ นามสกุลไฟล์อะไร ขนาดไฟล์ได้แค่ไหน
A6: คิดว่าก็ควรรองรับได้หมดนะ ความละเอียดก็สูงสุดเท่าที่ users นั้นรับได้ ส่วน Video ที่ upload ได้... เอาเป็นว่าไม่เกิน 1 GB ละกัน

#### Analysis เพิ่มเติม

5,000,000 DAU -> 5 ล้านคนต่อวัน

สมมติว่า user ดูคนละ 5 คลิป ละก็มี user ประมาณ 10% ที่มีการ upload คลิป และ upload 1 คลิปต่อคน ละก็ video ที่ upload มีขนาดเฉลี่ย 300 MB ละกัน

เท่ากับว่า Storage ที่ต้องใช้ต่อวัน = 0.1 x 5_000_000 x 300 MB = 150 TB เลย

ละก็สมมติว่าเราใช้ CDN ของ Third Party ละกัน (เพราะทำเองมันไม่คุ้ม) ละสมมติว่า 100% ของการดู Video ทั้งหมดมาจากประเทศ A ที่คิดค่าใช้จ่าย 0.02 ดอลลาร์ต่อ 1 GB 

เท่ากับว่า ค่าใช้จ่ายสำหรับ CDN ของประเทศ A จะเป็น 5_000_000 x 5 x 0.3 GB x 0.02 = 150,000 ดอลลาร์ต่อวัน

นี่ยังไม่นับค่าใช้จ่ายของประเทศอื่นๆ อีกนะ หรือกระทั่งค่าใช้จ่ายเรื่องอื่นๆ อีก เยอะจัดๆ แต่ว่ามันก็เป็นเรื่องที่ช่วยไม่ได้แหละนะ เพราะ บ. อื่นๆ เค้าก็ทำละก็ยังใช้ได้ เช่น Netflix ก็ใช้บริการ Amazon หรือ Facebook ก็ใช้ Akamai ไรงี้ 

---

## 2. Sketch ภาพระบบคร่าวๆ

### 1. Video Uploading Flow

![[youtube_video_uploading_flow.png]]

Flow ของการเพิ่ม Video จะเป็นดังนี้
1. เมื่อ User เพิ่ม Video เข้ามาในระบบ Video จะถูกเก็บไว้ใน Storage ที่เป็น Blob: Binary Large OBject ซึ่งเก็บข้อมูลเป็น Binary เพื่อประหยัดเนื้อที่การเก็บข้อมูล
2. Transcoding Servers แปลงข้อมูล Video ที่เก็บใน Storage จาก Binary เป็น Format ของ Video เช่น mpeg, hls ที่ทำให้การ stream video ทำได้อย่างเหมาะสมที่สุดกับ devices ของ user

>[!tip] About Transcoding... 
 Transcoding Server จะ**แปลง Video ต้นฉบับให้ออกมาหลาย Format** (เช่น AVI, MOV, MP4) **และหลาย Resolution** (เช่น 1080p, 720p, 480p, 360p) ไม่ใช่แค่แปลง Format เฉยๆ เพื่อรองรับ Internet Speed ของ Users ที่มีหลากหลาย เพื่อให้ Users สามารถเลือก Resolution ที่เหมาะสมได้

3. เมื่อแปลง Video สำเร็จ จะเกิด 2 เหตุการณ์ไปพร้อมๆ กัน
	1. Video ที่แปลงสำเร็จจะถูกส่งไปเก็บไว้ใน Storage อีกตัวนึง เพื่อให้ CDN ส่งข้อมูล Video ไปหา Device ของ User เมื่อ User ดู Video
	2. Transcoded Server ส่ง Message ว่าแปลง Video สำเร็จไปเก็บไว้ใน Queue จากนั้นเมื่อ Worker ดึง Message ไปแล้ว ก็จะมีการ update ข้อมูลใน Database และ Cache ที่เกี่ยวกับ Metadata ว่า Video upload สำเร็จแล้ว
4. เมื่อ API Servers ดูข้อมูลใน Metadata Cache/Database แล้วเห็นว่า Video นี้ upload เสร็จแล้ว ก็จะแจ้งข้อมูลกลับไปให้ user

### 2. Video Streaming Flow 

ต้องเล่าก่อนว่า การดู Video นึงๆ เนี่ย ไม่ใช่ว่าระบบส่ง Video ทั้งหมดมาให้ User ดูทีเดียว...

สิ่งที่ระบบทำคือการค่อยๆ ส่ง Video มาทีละส่วน โดยในระหว่างที่ User กำลังดู Video ที่ถูกส่งจาก Server มายัง Device ของ User แล้ว Server ก็จะค่อยๆ ส่งส่วนถัดๆ ไปของ Video มาให้ User ทำซ้ำไปเรื่อยๆ จนส่ง Video ครบทั้งหมด

ทีนี้ คำถามคือ Video แต่ละส่วนที่ว่านี้ Client ไปขอจากไหน? คำตอบคือ**ไม่ได้ขอตรงจาก Server ของเราเลย** แต่ขอผ่าน CDN แทน - เพราะ Video เป็นไฟล์ขนาดใหญ่และ User กระจายอยู่ทั่วโลก การให้ User ทุกคนดึง Video ตรงจาก Server ที่คนที่ upload video ใช้เนี่ย มันจะช้ามากในการดึง Video มาให้ Users อื่นดู (สามารถอ่านเนื้อหาเพิ่มเติมได้ใน [[07 CDN (Content Delivery Network)|CDN]])

Flow เลยเป็นแบบนี้:

1. หลัง Transcoding เสร็จ Video ที่แปลงแล้วหลาย Resolution จะถูก Push ขึ้นไปเก็บที่ **CDN**
2. ตอน User กด Play, Client จะขอ Video ทีละส่วน (Chunk) จาก **Server ของ CDN ที่อยู่ใกล้ User ที่สุด** แทนที่จะขอจาก Origin Server
3. พอ Chunk แรกโหลดมาถึง Client ก็เริ่มเล่นได้ทันทีเลย (ไม่ต้องรอโหลดทั้งไฟล์) แล้ว Client จะทยอยขอ Chunk ถัดไปเรื่อยๆ ล่วงหน้าไว้ก่อนที่ Chunk ปัจจุบันจะเล่นจบ (เรียกว่า **Buffering**)
4. ถ้า Internet ของ User ช้าลงกลางทาง Client จะเปลี่ยนไปขอ Chunk ที่ Resolution ต่ำกว่าแทน เพื่อไม่ให้ Video กระตุก

---

## 3. เจาะลึกจุดสำคัญๆ

### Video Transcoding

เมื่อตะกี้เราได้พูดเรื่องของ Video Transcoding ไปเนอะ ว่าเป็นการแปลง format ของ video จาก binary เป็น video format ต่างๆ รวมถึงมีการทำ Resolution ไว้หลากหลายแบบ เพื่อให้รองรับ Internet Speed ที่หลากหลายของ User นั่นเอง

ตรงนี้จะเป็น List เหตุผลว่าทำไมต้องมีการทำ Video Transcoding
- Video Format เดิมมันกินเนื้อที่เยอะ การเก็บเป็น Blob มันเลยประหยัดเนื้อที่กว่า
- Device ของ User รองรับ Format Video ได้ไม่เหมือนกัน เพราะงั้นถ้าทำ Video ไว้หลาย Format ก็ทำให้ Device ของ User สามารถใช้งานได้
- Internet Speed ของ User มีการเปลี่ยนแปลงอยู่ตลอด การมี Video ไว้หลาย Resolution เลยทำให้การดู Video ของ User ไม่สะดุด (แต่ว่าก็ยอมแลกกับ Quality ของ Video อ่ะนะ ซึ่งเข้าใจได้)

### DAG (Directed Acyclic Graph) Model

ทีนี้มีปัญหาเพิ่มเติมอีกนิดนึง คือการ Transcode Video มันใช้ Resource เยอะ และใช้เวลานาน แถม video ที่ upload เข้ามาก็อาจจะมีความต้องการที่ต่างกัน เช่น ต้องมี Watermark, ใข้ Thumbnail ของตัวเอง, หรือดัน upload Video ความละเอียดสูงมาก ไรงี้ การออกแบบให้ Pipeline ของการ process Video มีความยืดหยุ่นและ parallelism จึงมีความสำคัญ 

เช่น ระบบของ Facebook ก็แก้ปัญหานี้ด้วยการใช้ **DAG (Directed Acyclic Graph) Model** คือการกำหนดงานทั้งหมดเป็น Stage ต่อๆ กัน โดยแต่ละ Stage จะรันแบบเรียงลำดับ (Sequential) หรือรันพร้อมกัน (Parallel) ก็ได้ ขึ้นกับว่า Stage ไหนต้องรอผลลัพธ์จาก Stage ไหนก่อน

![[ัyoutube_dag_model.png]]

ใน case นี้ Model DAG ถูกใช้โดยการนำ Video ต้นฉบับมาแยกออกเป็น 3 ส่วนที่ทำงานพร้อมกัน คือ **Video, Audio, Metadata** จากนั้นแต่ละส่วนก็ไปผ่าน Task ต่างๆ ตามที่ Config ไว้ 

ตัวอย่าง Task ที่ทำได้กับ Video มีดังนี้:
- **Inspection**: ตรวจสอบว่า Video มีคุณภาพดีพอ และไฟล์ไม่เสีย
- **Video Encoding**: แปลง Video ให้รองรับหลาย Resolution เช่นได้ผลลัพธ์เป็น `360p.mp4`, `480p.mp4`, `720p.mp4`, `1080p.mp4`, `4k.mp4`
- **Thumbnail**: สร้างภาพตัวอย่างของ Video ให้อัตโนมัติ หรือถ้า User Upload มาเองแล้วก็ใช้ของ User แทน
- **Watermark**: ใส่ภาพซ้อนทับบน Video เพื่อบอกข้อมูลระบุตัวตนของเจ้าของ Video

พอผ่านครบทุก Task แล้ว ก็นำผลลัพธ์ของ Video, Audio, และ Metadata มารวมกันเป็น Video ที่สมบูรณ์

### Video Transcoding Architecture

![[video_transcoding_architecture.png]]

มาดูกันว่า Architecture จริงๆ ที่ใช้ Cloud Service เพื่อรองรับ DAG Model นี้มีหน้าตาเป็นยังไง

**1. Preprocessor**
- Split Video ออกเป็นกลุ่ม Frame เล็กๆ หรือ GOP (Group Of Pictures) ที่เล่นแยกได้ ยาวไม่กี่วินาที
- สร้าง DAG ขึ้นจาก Configuration File (เช่น กำหนด Task ชื่อ `download-input` ที่มีหน้าที่รับ URL มา แล้วส่งต่อไปที่ Task `transcode` ที่มีหน้าที่สร้าง DAG Model)

![[youtube_dag_configuration_files.png]]

- เป็น Cache สำหรับบันทึก Video ที่ถูก split เป็นกลุ่ม Frames เล็กๆ ด้วย โดยเก็บทั้งกลุ่ม Frames และ Metadata ไว้ใน Temporary Storage เผื่อว่า encode ล้มเหลว

![[youtube_dag_scheduler.png]]

**2. DAG Scheduler** ที่แบ่ง DAG ออกเป็น Stage ของ Task ต่างๆ แล้วส่งเข้า Queue ของ Resource Manager เช่นแบ่ง Video เป็น 2 Stage - **Stage 1** แยกเป็น Video, Audio, Metadata และ **Stage 2** เอา Video ไป encode ต่อและสร้าง Thumbnail พร้อมกัน ส่วน Audio ก็ไป encode แยกต่างหาก

![[youtube_resource_manager.png]]

**3. Resource Manager** ทำหน้าที่บริหารจัดการ Resource ให้มีประสิทธิภาพที่สุด ประกอบด้วย 3 Queue กับ Task Scheduler 1 ตัว:
- **Task Queue**: Priority Queue ที่เก็บ Task ที่รอถูกรัน เพื่อให้ Resource Manager ดึง Task ที่มี Priority สูงสุดไปให้ Task Workers
- **Worker Queue**: Priority Queue ที่เก็บข้อมูลว่า Worker แต่ละตัวว่างมากน้อยแค่ไหน เพื่อให้ Resource Manager เลือก Worker ที่เหมาะสมที่สุดไปเป็น Task Worker
- **Running Queue**: เก็บข้อมูลของ Task/Worker โดย Task ก็มาจาก Task Queue ส่วน Worker ก็มาจาก Worker Queue
- **Task Scheduler**: เลือก Task/Worker ที่เหมาะสมที่สุด แล้วสั่งให้ Task Worker นั้นรันงาน เมื่อ run เสร็จก็เอา Task/Worker นั้นออกจาก Running Queue

**4. Task Workers** คือ Worker ที่ run Task จริงๆ ตามที่ DAG กำหนดไว้ เช่น Worker สำหรับ Watermark, สำหรับ Encoder, สำหรับ Thumbnail, หรือสำหรับ Merger แยกกันไปเลย

**5. Temporary Storage** โดยจะใช้ Storage หลายแบบผสมกัน ตามลักษณะข้อมูล เช่น 
- Metadata ที่ถูกเรียกใช้บ่อยและมีขนาดเล็ก จึงเหมาะกับการ Cache ไว้ใน Memory 
- Video/Audio ที่มีขนาดใหญ่ จะเก็บใน Blob Storage แทน
เมื่อ Worker ทำงานเสร็จก็ลบข้อมูล Task ออกจาก Storage

**6. Encoded Video** คือผลลัพธ์สุดท้ายของ Pipeline เช่น `funny_720p.mp4`

---

### System Optimizations

#### Speed Optimization

![[youtube_speed_optimization_gop.png]]

**1. Parallelized Video Uploading** โดยตัด Video ออกเป็นชิ้นเล็กๆ ตาม GOP Alignment ตั้งแต่ตอน upload Video เลย ถ้า upload ล้มเหลว สามารถเริ่ม upload ใหม่ได้เร็วจากจุดที่ล้มเหลวได้เลย

![[youtube_speed_optimization_upload_center.png]]

**2. วางตำแหน่ง Upload Center (Server สำหรับ upload video) ให้ใกล้ User** ตั้ง Upload Center กระจายไว้หลายจุดทั่วโลก เพื่อลด Latency เวลา user upload videos

![[youtube_system_optimization_parallelism.png]]

**3. Parallelism ทุกจุดที่ทำได้** ผ่านการใช้ Message Queue เพื่อให้ Step ถัดๆ ไปสามารถ check Queue ของ Task ก่อนหน้า ละดึง Task ไปทำต่อได้เลยแบบไม่ต้องรอให้ Step ก่อนหน้าทำให้เสร็จ

#### Safety Optimization

![[youtube_security_optimization_presign_url.png]]

**1. ทำ Pre-signed Upload URL** เพื่อให้มีแค่ User ที่ได้รับอนุญาตเท่านั้นที่สามารถ upload Video ไปยังตำแหน่งที่ถูกต้องได้

เผื่อยังงงๆ อยู่ -> Flow ปกติของการ upload video คือ 

> 1. Client ส่งไฟล์ Video
> 2. API Server รับ Video จาก Client 
> 3. API Server ส่ง Video เก็บใน Storage

ซึ่งมีการประมวลผล Video ที่ API Server ถึง 2 รอบ ซึ่งทำให้ API Server กลายเป็นคอขวดอย่างชัดเจนเวลา Video มีขนาดใหญ่ 

การมี pre-signed URL เลยเป็นเหมือนของยืนยันตัวตนจาก API Server ว่า User สามารถ access เข้า Storage ได้ผ่าน URL นี้เลยอ่ะ 

Flow เลยเป็นแบบนี้

> 1. Client ส่งไฟล์ Video
> 2. API Server รับ Video จาก Client 
> 3. API Server ส่ง Pre-signed URL

จากนั้นก็

>4. Client upload ไฟล์ตรงๆ ผ่าน Pre-signed URL
>5. Storage ยอมให้ Client upload Video เก็บไว้ได้

**2. ป้องกัน Video ของ Content Creator** เช่น การเข้ารหัส Video, การแปะลายน้ำ / ภาพซ้อนทับที่ระบุตัวตนของ User ที่ upload Video

#### Cost-saving Optimization

![[youtube_cost_optimization.png]]

เมื่อช่วงต้นที่มีการวิเคราะห์ถึง Cost สำหรับ Youtube นั้น ก็ปฏิเสธไม่ได้ว่าควร optimize เพื่อลดค่าใช้จ่ายอ่ะนะ แล้วจะทำยังไงได้บ้างนะ...
1. **ให้ CDN ส่งเฉพาะ Video ยอดนิยม** ส่วน Video ที่เหลือให้ส่งจาก Video Server ความจุสูงของเราเองแทน (ประหยัดกว่าการเก็บทุก Video ไว้ที่ CDN)
2. **Video ที่ไม่ค่อยมีคนดู ไม่จำเป็นต้อง Encode ไว้หลาย Version ล่วงหน้า** ไว้เมื่อไหร่มีคนเริ่มดู จะเริ่ม encode ตอนนั้นก็ไม่เสียหาย
3. **Video บางคลิปเป็นที่นิยมแค่บางภูมิภาค** ก็ไม่จำเป็นต้องกระจายไปเก็บที่ภูมิภาคอื่นที่ไม่มีคนดู
4. **สร้าง CDN ของตัวเอง** (แบบที่ Netflix ทำ) แล้วจับมือเป็นพาร์ทเนอร์กับ ISP (Internet Service Provider เช่น Comcast, AT&T, Verizon) เพื่อให้สามารถ optimize cost ต่างๆ ได้อย่างอิสระ แต่ก็แลกมากับกระบวนการที่ซับซ้อนมากด้วยเช่นกัน

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Youtube / Video Streaming Platform ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---

