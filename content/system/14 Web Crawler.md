---
title: 14 Web Crawler
tags:
  - system
  - systemdesign
  - web_crawler
  - algorithm
  - bfs
  - hash_ring
---
วันนี้เอาเป็น Web Crawler ละกัน - Web Crawler คือ System ที่คอยค้นหาข้อมูลใน web ว่ามี content อะไรใหม่ๆ หรือมี content อะไรที่ update ไปบ้าง 

![[web_crawler_overview.png]]

Web Crawler สามารถเอาไปใช้ได้หลายวัตถุประสงค์อ่ะนะ เช่น...
- Search Engine Indexing - รวบรวมข้อมูลของ content เพื่อนำมาสร้าง index สำหรับ Search Engine เช่น Googlebot ที่ทำงานให้ Google Search Engine ไรงี้
- Web Monitoring - คอยค้นหาใน internet ว่ามี web ใดบ้างที่ขโมย content ของตนไปใช้โดยไม่ได้รับบอนุญาต
- Web Mining - สำรวจหา content ใหม่ๆ เพื่อนำมาหา insight ต่างๆ

จะเห็นได้ว่ามันก็เป็นระบบนึงที่ช่วยทำให้มีข้อมูลเพื่อนำไปต่อยอดในเรื่องต่างๆ ได้เลย แล้วระบบนี้มัน implement กันยังไง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Web Crawler ที่จะให้ทำนี่จะเน้นใช้งานเรื่องอะไรเหรอครับ?
A1: เอาไว้ทำ Search Engine Indexing ละกัน

Q2: แต่ละเดือนเนี่ย Web Crawler จะต้องเก็บข้อมูล web มาเยอะแค่ไหนเหรอครับ?
A2: 1 billion web pages ละกัน

Q3: แล้วจะเก็บ content อะไรบ้างเหรอครับ?
A3: เอาแค่ HTML พอละ pdf หรือรูป หรืออื่นๆ ไม่จำเป็น

Q4: Web ที่มี content ซ้ำกันควรเอามาด้วยมั้ยครับ?
A4: ไม่เอา ที่เอามาต้องมี content ที่ต่างกันเท่านั้น

หลักการในเบื้องต้นของ Web Crawler มี 3 steps คือ

1. กำหนด List ของ URLs เพื่อให้ Web Crawler ไป download web pages เหล่านั้นมา
2. Web Crawler ไปตาม URL ดังกล่าวละดึง content มา จากนั้นก็ค้นหา URL ใน web pages เหล่านั้นมาด้วย
3. นำ URL ที่ได้มาเติมเข้า List ของ URL เพื่อใช้ค้นหาใน web pages ต่อไป

นอกจากนี้ พวก Non-Functional Requirement ที่ควรจะมี ก็เช่น...

- Scalability - Web Pages มีเป็นล้าน Web Crawlers มันควรจะทำงานแบบ Parallel กันได้อ่ะ
- Robustness - Web Pages มีหลายรูปแบบ บางอัน HTML ไม่ดี บางอันไม่ Responsive หรือบางอันเข้าไปแล้วล่ม หรือมีปัญหาอะไรบางอย่าง Web Crawler ควรจะจัดการเรื่องพวกนี้ได้อย่างเหมาะสม
- Politeness - เพื่อไม่เป็นการรบกวนการทำงานของ Web Page ที่เราเข้าไปมากจนเกินไป Web Crawler ของเราไม่ควรจะยิง Request ไปหา Web Page เหล่านั้นเยอะเกินไป

---

## 2. Sketch ภาพระบบคร่าวๆ

![[web_crawler_high-level_design.png]]

Flow ก็ตามภาพเลย ประมาณว่า...
1. กำหนด seed URLs หรือว่า List ของ URL เริ่มต้นที่จะให้ Web Crawler ค้นหาลงใน URL Frontier
2. HTML Downloader ดึง URL จาก URL Frontier ไป fetch -> ต้องไปขอข้อมูลจาก DNS Resolver ก่อนว่า URL ที่ว่าอยู่ที่ IP อะไร
3. เมื่อได้ IP มาแล้วก็ download content ที่ได้จากการ fetch URL
4. Content Parser ใน Web Crawler ก็แปลง HTML ที่ได้ในรูปแบบที่สามารถวิเคราะห์ต่อได้
5. เมื่อแปลง HTML แล้ว validate content ที่ได้แล้ว ก็ส่ง content นั้นไปเช็คว่าเป็น content ที่เคยเห็นหรือยัง โดยเช็คกับ Content Storage
	- ถ้ามี content นี้แล้ว ก็ปล่อย HTML ต้นทางไป
	- ถ้ายังไม่มี content ก็เก็บ content นี้เข้า Content Storage ละก็ส่ง HTML ต้นทางไปหา Link Extractor
6. Link Extractor ดึง URL ที่พบใน HTML ออกมา
7. นำ URL ที่เจอไป validate ที่ URL Filter ว่าเป็น URL จริงๆ หรือไม่
8. ถ้าเป็น URL จริงๆ ก็นำไปเช็คต่อว่าเป็น URL ที่เคยเห็นหรือยัง โดยเช็คกับ URL Storage
	- ถ้ามี URL นี้แล้วก็ปล่อย URL นี้ไป
	- ถ้าไม่มี URL ก็เก็บ URL เข้า URL Storage ละก็เอา URL นี้ไปเก็บใน URL Frontier เพื่อวน cycle นี้ต่อไป

---

## 3. เจาะลึกจุดสำคัญๆ

### URL Frontier

ด้วย Flow ด้านบน มันจะมีตัวละครสำคัญคือ URL Frontier ที่เหมือนเป็นแหล่งรวม URL ที่ทยอยส่ง URL ให้เข้าไปตาม Flow เลยนะ เพราะงั้นเราจะมาดูกันว่า URL Frontier นี้มีอะไร

บนพื้นฐานของ URL Frontier ก็จะมี List เนอะที่เก็บ URL เอาไว้ ละก็จะมี Algorithm ที่คอยจัดการว่าจะเอา URL ตัวไหนออกมาจาก List แล้วส่งให้ Flow ไป หลักๆ ก็จะมี 2 ตัวก็คือ DFS ละก็ BFS แหละ

DFS เราไม่ค่อยใช้กันเพราะว่ามันจะแทง content ลึกลงไปเรื่อยๆ จนทำให้ content ของ web อื่นๆ ไม่ถูกนำมาเข้า Flow เลย มีแต่ content ของ web เดียว

แต่ถึงจะบอกแบบนั้น การใช้ BFS ก็จะยังเจอกับปัญหาอยู่ดี เพราะ
- หลายๆ web page จะมีการ redirect กลับไปที่หน้าเดิม / URL เดิมที่เราเคยค้นหาไปแล้ว ถึงต่อให้ใน Flow เราจะมีตัวกรอง แต่ว่า URL นั้นก็ถูก fetch ไปแล้ว เท่ากับเราใช้ Resources ของเค้าไปแล้วนั่นเอง โดยไม่เกิดประโยชน์อะไรด้วย -> Impolite --> นำไปสู่การ DDoS ด้วยซ้ำ (ยิง Request หลายๆ อันเพื่อทำให้ Server ทำงานหนักขึ้น)
- ไม่มีการทำเรื่อง Priority ว่า ควรจะ check web pages ที่มี priority สูงๆ ก่อน เช่น web pages ที่มี content ที่ดี หรือตรงตามความต้องการของผู้ใช้ Web Crawler งี้ -> No Priority

เพราะงั้น URL Frontier ก็ต้องมีการจัดการ 2 ปัญหานี้ด้วยนั่นเอง
#### Politeness

![[web_crawler_politeness.png]]

เรื่องของการเข้า URL ที่เคยเข้าถึงไปแล้วซ้ำๆ เราสามารถทำได้ผ่านการทำเป็น Queue ที่แยก Domain กัน เช่น เวลามี URL จาก Wikipedia เข้ามา ก็เอาไปใส่ Queue A, URL จาก Medium เข้ามา ก็เข้าไปใส่ Queue B เป็นต้น ละก็กระจายให้ Worker ทำงานแค่เฉพาะ Domain ที่ได้รับมอบหมายเท่านั้น ก็จะได้เรื่อง Parallelism ไปด้วยในตัว

#### Priority

![[web_crawler_priority.png]]

สมมติว่าข้อมูลจาก Medium มันมีประโยชน์กับเรา เราก็อยากให้วิเคราะห์จาก Medium เป็นหลัก หรือก็คือ Medium คือ Rank1 ส่วนข้อมูลจาก Wikipedia ถือว่ารองลงมา หรือก็คือ Rank2 ก็นำ URL จาก Medium ใส่ใน Queue F1 ส่วน Wikipedia ก็เอาไปใส่ใน Queue F2 ไรงี้ ก็เป็นการส่ง Content ที่เราต้องการได้ โดยไม่ถูกคั่นกลางจาก Content ที่เราเห็นว่าไม่จำเป็นได้ ผ่านการเช็คจาก Queue ที่ Rank สูงๆ ก่อนนั่นเอง

### HTML Downloader

อีกตัวละครที่สำคัญคือ HTML Downloader เนอะ แล้วมันมีอะไรให้วิเคราะห์นะ มันก็แค่ download web page ผ่าน HTTP Protocol มา ก็แค่นั้นป่ะ

จริงๆ ก็ใช่แหละ แต่ว่ามันก็มีมากกว่านั้นอยู่ คือเรื่อง Politeness นี่แหละ เพราะการเป็น Crawler ที่ดี ไม่ใช่แค่โหลดเป็น แต่ต้อง **โหลดอย่างมีมารยาท** และ **โหลดให้เร็ว** ด้วย

#### Robots.txt

ก่อนจะโหลดหน้าไหนจากเว็บนึง Crawler ที่ดีต้องเช็ค **`robots.txt`** ของ web นั้นก่อนเสมอ ไฟล์นี้คือมาตรฐานที่เว็บไซต์ใช้บอก Crawler ว่า **หน้าไหนโหลดได้ หน้าไหนห้ามแตะ** เช่น `amazon.com/robots.txt` ที่ห้าม Googlebot เข้าบาง path:

```txt
User-agent: Googlebot
Disallow: /creatorhub/*
Disallow: /gp/aw/cr/
```

โดยทางที่ดี คือ Web Crawler เราควรจะมีการ Cache robots.txt ของ Domain นั้นๆ เอาไว้ เพื่อไม่เป็นการเสียเวลาโหลด robots.txt ใหม่ทุกครั้งๆ ทั้งๆ ที่ Web Crawler เคยมา Domain นี้แล้ว

#### Performance Optimization

นอกจากโหลดอย่างมีมารยาทแล้ว ก็ต้องโหลดให้เร็วด้วย มีเทคนิคหลักๆ อยู่ 4 อย่าง

**1. Distributed Crawl**: กระจายงานโหลดลงหลาย Server แต่ละ Server รันหลาย Thread โดยแบ่ง URL ออกเป็นส่วนๆ ให้แต่ละตัวรับผิดชอบคนละชุด ไม่ทับกัน

**2. Cache DNS Resolver**: ตรงนี้เป็นคอขวดที่คนมองข้าม เพราะการแปลง domain เป็น IP (DNS lookup) มันช้า (10-200ms) และมักเป็นแบบ Synchronous คือระหว่างที่ Thread นึงรอ DNS ตอบ Thread อื่นก็ต้องรอตามไปด้วย ทางแก้คือ Cache mapping ของ `domain → IP` ไว้ แล้วอัปเดตเป็นระยะ เพื่อไม่ต้องยิงถาม DNS บ่อยๆ

**3. Locality**: วาง Crawl Server ไว้ใกล้ๆ กับเว็บที่จะโหลด ยิ่งใกล้ยิ่งโหลดเร็ว หลักการนี้ใช้ได้กับทุก component เลย ทั้ง Server, Cache, Queue, Storage

**4. Short Timeout**: บางเว็บตอบช้าหรือไม่ตอบเลย ถ้ารอไปเรื่อยๆ ก็เสียเวลา เลยตั้งเวลารอสูงสุดไว้ ถ้าเว็บไม่ตอบภายในเวลาที่กำหนด ก็ข้ามไปโหลดหน้าอื่นต่อเลย

### Robustness

NFR อีกข้อนึงที่เราต้อง concern คือเรื่อง robustness อย่างการที่เราต้องดึง content ออกมาได้ แม้ว่า content เหล่านั้นจะมี errors ก็ตาม แล้วจะทำยังไงได้บ้างนั้น... มาดูกัน
1. **Consistent Hashing**: เพื่อกระจายโหลดระหว่าง Downloader หลายๆ ตัว ข้อดีคือเวลาจะเพิ่มหรือถอด Downloader ออก ก็ทำได้โดยกระทบ Server ตัวอื่นน้อยที่สุด ไม่ต้องยกเครื่องกระจายโหลดใหม่ทั้งหมด (อ่านต่อได้ใน [[03 Consistent Hashing|Consistent Hashing]])
2. **Save Crawl States and Data**: เขียน state และข้อมูลของการ crawl ลง Storage เป็นระยะๆ เผื่อว่าระบบล่มกลางคัน จะได้ไม่ต้องเริ่มใหม่ตั้งแต่ศูนย์ แค่โหลด state ที่เซฟไว้แล้ว crawl ต่อจากจุดเดิมได้เลย
3. **Exception Handling**: ในระบบขนาดใหญ่ Error เป็นเรื่องปกติที่เลี่ยงไม่ได้ Crawler ต้องจัดการ Error ได้อย่างนุ่มนวล คือเจอปัญหาที่หน้าไหนก็ข้ามไป ทำงานต่อได้ ไม่ใช่พังทั้งระบบเพราะหน้าเดียว
4. **Data Validation**: ตรวจสอบความถูกต้องของข้อมูลที่ดึงมาก่อนเอาไปใช้ต่อ เพื่อกัน Error ที่จะลามไปทำให้ระบบส่วนอื่นเพี้ยนตาม

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Web Crawlers ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---