---
title: 09 Search Autocomplete System
tags:
  - system
  - systemdesign
  - search
  - trie
  - key_value
  - cache
  - sharding
---
วันนี้เรามา discuss กันเรื่อง Search Autocomplete System กันดีกว่า อาจจะงงว่าคืออะไร ก็คือ Search Engine ของ Google นี่แหละ ที่ความพิเศษของมันคือการ Suggest User ถึงคำที่อาจจะเกี่ยวข้องกับสิ่งที่ User กำลังพิมพ์อยู่ เค้าทำได้ยังไง มีอะไรให้วิเคราะห์ได้บ้าง ละสร้างขึ้นมาได้ยังไง มาดูกัน

--- 

## 1. ถามรายละเอียดของ Scope งาน

Q1: การ suggest words เนี่ย มันจะเริ่มตั้งแต่ตอนที่ user พิมพ์ตัวแรกเลย หรือว่าตอนที่ user พิมพ์ไปได้ซักพักแล้วอ่ะครับ?
A1: ตั้งแต่เริ่มเลย

Q2: แล้วระบบจะ suggest ให้ user กี่คำเหรอครับ
A2: เอาซัก 5 คำละกัน

Q3: แล้ว...อยากให้ระบบเลือก 5 คำนั้นด้วยเกณฑ์อะไรเหรอครับ
A3: ความยอดนิยมละกัน แบบว่า...มี users อื่นค้นหาคำนี้เยอะ ก็ suggest คำนี้ให้ไป

Q4: มี spell check ให้ user มั้ยครับ
A4: ไม่เอา spell check ละก็ไม่เอา autocorrect ด้วย

Q5: จะให้ user พิมพ์ตัวพิมพ์ใหญ่ได้มั้ย อักขระพิเศษอ่ะ
A5: ไม่เอาละกัน user พิมพ์ได้แค่ตัวพิมพ์เล็ก

Q6: แล้ว...จะมีคนใช้งานเยอะขนาดไหนเหรอครับ?
A6: 10,000,000 DAU (Daily Active Users)

#### Analysis เพิ่มเติม

10,000,000 DAU -> 10 ล้านคนต่อวัน

สมมติว่า 1 คน มีการค้นหา 10 ครั้ง (โดยเฉลี่ย) ละก็แต่ละครั้งที่ค้นหา จะใช้ query ที่มีความยาว 20 ตัวอักษร (โดยเฉลี่ย) หรือก็คือ แต่ละครั้ง ระบบจะต้องค้นหาคำจาก query ของ user 20 ครั้ง : พิมพ์ 1 ตัวอักษร ระบบหาคำในระบบ 1 ครั้ง

เท่ากับว่า ใน 1 วินาที ระบบจะต้องค้นหาคำในระบบได้ 10,000,000 * 10 * 20 / 24 / 36000 = 24000 ครั้ง -> เยอะมาก...

ละก็สมมติว่า 20% ของ query ของ users เป็นคำใหม่ที่ไม่มีในระบบ หมายความว่าใน 1 วันจะมีการเติมข้อมูลใหม่ใน database 10,000,000 * 10 * 20 * 0.2 = 400 MB เลยทีเดียว เยอะมากๆ อีกเหมือนกัน : 1 ตัวอักษรใช้เนื้อที่ 1 byte

---

## 2. Sketch ภาพระบบคร่าวๆ

ก็...ภาพง่ายๆ ที่ทำได้ ก็คือเรามี 2 Services

- Data gathering Service ที่ทำหน้าที่จด query ของ user แบบ real-time เพื่อจดคำที่ user ต้องการจะค้นหา ละก็จดจำนวนครั้งเอาไว้ด้วย
	![[search_autocomplete_data_gathering.png]]

- Query Service ที่ทำหน้านี้ represent คำที่จะ suggest ให้ user ตามที่ user พิมพ์ query เข้ามา
	![[search_autocomplete_query.png]]

	โดย basic ก็สามารถ suggest คำได้ผ่าน query นี้
	```sql
	SELECT * FROM frequency_table
	WHERE query LIKE `<word_from_user>%`
	ORDER BY frequency DESC
	LIMIT 5
	```

---

## 3. เจาะลึกจุดสำคัญๆ

 ก่อนอื่นเลย เราจะเก็บข้อมูลยังไงให้มีประสิทธิภาพที่สุดในการ query เอาข้อมูล คำตอบคือ ทำเป็น Trie หรือ Tree Data Structure
### Trie Data Structure

Trie คือ Data Structure ที่มีการ visualize ในรูปแบบต้นไม้ ที่มี root เป็น string เปล่า แต่ละ node จะเก็บเป็นคำที่ user พิมพ์เข้ามาแบบทีละตัวอักษร ละก็จะมีการจดจำนวนครั้งของคำที่เป็น leaf

![[search_autocomplete_trie_ds.png]]

ขั้นตอนในการทำงาน จะเป็นดังนี้ (สมมติว่า user คนพิมพ์ query ว่า "tr")
1. ไล่จาก root ลงไปตาม children nodes เรื่อยๆ จนเจอ node ที่เป็นคำว่า "tr"
2. จาก node "tr" ดึง children ของ node "tr" ออกมาทั้งหมด -> ในที่นี้จะเป็น tree, true และ try
3. sort ข้อมูล แล้วดึงมาแค่เฉพาะคำ top n ที่มีการ query มากที่สุด -> n = 2 ก็จะเอาแค่คำว่า true: 35 และ try: 29

โดยเราสามารถ optimize เพิ่มเติมได้ ผ่านการ cache ผลลัพธ์การค้นหา top n ว่าเป็นคำว่าอะไร ไว้ในทุกๆ nodes ตามภาพนี้
![[search_autocomplete_trie_ds_cache.png]]

ในที่นี้ สมมติว่า user พิมพ์คำว่า be และสนใจ top n = top 5 ตามที่ requirement กำหนดว่า ระบบก็จะ suggest 5 คำ คือคำว่า best, bet, bee, be, buy แบบอัตโนมัติ และรวดเร็วกว่า case ปกติ เพราะมีการ cache ผลลัพธ์การ query ไว้แล้วนั่นเอง

### Data Gathering Service

ปัญหาของ Design ที่ต้อง Update Trie แบบ Real-time ทุกครั้งที่ User พิมพ์ Query มีอยู่ 2 อย่าง:
- Query เข้ามาเยอะมาก ถ้า Update Trie ทุกครั้งจะทำให้ Query Service ช้า
- Top Suggestion ไม่ค่อยเปลี่ยนบ่อย Update ทุกครั้งเลยเสีย Resource เปล่าๆ

เลยออกแบบใหม่ให้ทำงานแบบ **Batch** (ประมวลผลเป็นรอบๆ) แทน โดยมี Component หลักดังนี้:

1. **Analytics Logs** ที่เก็บ Search Query ทั้งหมด แบบ Append-only (เพิ่มอย่างเดียว ไม่แก้/ลบ) ไม่ทำ Index (เน้นเขียนเร็ว)

|query|time|
|---|---|
|tree|2019-10-01 22:01:01|
|try|2019-10-01 22:01:05|
|toy|2019-10-01 22:02:22|

2. **Aggregators** ที่ทำหน้าที่สรุปข้อมูลจาก Log ให้อยู่ในรูปที่ Process ต่อได้ ส่วนจะสรุปถี่ขนาดไหน ก็แล้วแต่การใช้งาน

3. **Aggregated Data** หรือผลลัพธ์จาก Aggregator (`time` = วันเริ่มสัปดาห์, `frequency` = ผลรวมการค้นหาในสัปดาห์นั้น)

| query | time       | frequency |
| ----- | ---------- | --------- |
| tree  | 2019-10-01 | 12000     |
| toy   | 2019-10-01 | 8500      |

4. **Workers** คือ Server ที่ทำงาน Asynchronous เป็นรอบๆ มีหน้าที่เอา Aggregated Data มาสร้าง Trie แล้วเก็บลง Trie DB

5. **Trie DB** ที่เป็นข้อมูลหลักๆ โดยจะทำเป็น **Key-Value Store** เช่น `be` → `[be: 15, bee: 20, beer: 10, best: 35]`
	![[search_autocomplete_keyvalue_store.png]]

6. **Trie Cache** เป็น Distributed Cache ที่เก็บ Trie ไว้ใน Memory โดยจะรอรับ Snapshot จาก Trie DB เมื่อ Aggregators มีการ update ข้อมูล

### Query Service

![[search_autocomplete_query_service.png]]

Flow การทำงานตาม Trie Design จะเป็นแบบนี้:
1. Query ที่ User พิมพ์เข้ามา ถูกส่งไปที่ Load Balancer
2. กระจาย Request ไป API Servers
3. API Servers ดึง Trie จาก **Trie Cache** ใน Memory ของ User ก่อน
4. ถ้า **Cache Miss** ก็จะไปดึงจาก **Trie DB** แทน แล้วเติมกลับเข้า Trie Cache ของ User
 
### Trie Operations

- **Create**: Workers สร้าง Trie จาก Aggregated Data
 
- **Update**: Workers มีการ update Trie ผ่าน 2 รูปแบบ
	- อัปเดตทั้ง Trie เป็นรอบๆ แล้วแทนที่ของเก่าทั้งอัน
	- อัปเดตทีละ Node โดยตรง (ช้า ไม่ควรจะทำบ่อยๆ เว้นแต่ Trie เล็ก) แต่ต้องไล่อัปเดต **Ancestor ทุกตัวจนถึง Root ด้วย** เพราะ Ancestor เก็บ Top Query ของ Children ไว้ เช่นในภาพนี้ มีการ update คำว่า `beer` เป็น 30 ครั้ง ก็ต้องไล่ update ใน `bee` `be` และ `be` ด้วย
	![[search_autocomplete_update_node_manually.png]]

- **Delete**: มีการเพิ่ม **Filter Layer** ใน Trie Cache เพื่อกรอง Suggestion ที่ไม่เหมาะสมออก ส่วนข้อมูลจริงถูกลบออกจาก DB แบบ Asynchronous

### Scale the Storage

ด้วยความที่ Trie จะมีขนาดใหญ่ขึ้นเรื่อยๆ ก็ควรจะต้องมีการกระจายข้อมูลให้เป็นชิ้นเล็กๆ วิธีการที่ทำได้ คือ Sharding ตาม**ตัวอักษรตัวแรก** เช่น 2 Server แบ่ง 'a-m'/'n-z' ไล่ไปได้สูงสุด 26 Server (**First-level Sharding**) คือ a, b, c, ..., z ถ้าต้องการมากกว่านั้น ก็สามารถทำ Shard ต่อระดับ 2-3 ได้ (เช่น 'a' แบ่งเป็น 'aa-ag', 'ah-an' เป็นต้น)

แต่ว่าก็ยังมีปัญหาอยู่ คือคำที่ขึ้นต้นด้วย 'c' เยอะกว่า 'x' มาก ทำให้เกิด **Data Imbalance**

วิธีแก้คือใช้ **Shard Map Manager** - Lookup Table ที่บอกว่าข้อมูลไหนอยู่ Shard ไหน โดยอิงจาก Pattern การกระจายในอดีต เช่นถ้า 's' มีปริมาณพอๆ กับ 'u-z' รวมกัน ก็แยกเป็น 2 Shard ('s' อย่างเดียว กับ 'u-z' รวมกัน) แทนที่จะแบ่งตามตัวอักษรตรงๆ นั่นเอง

![[search_autocomplete_shard_map.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Search Autocomplete System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
