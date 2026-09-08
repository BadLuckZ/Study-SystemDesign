---
title: 05 URL Shortener
tags:
  - systemdesign
  - system
  - url_shortener
---

URL Shortener ก็ตามตัวแหละ คือสิ่งที่ทำให้ URL มีความยาวสั้นลง ส่วนจะ implement ยังไง มาดูกัน...

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: อยากรู้ว่า URL Shortener นี่มันทำงานยังไง
A1: สมมติว่าเรามี URL ที่ยาวๆ ซักอันนึง เช่น `https://www.systemdesigninterview.com/q=chatsystem&c=loggedin&v=v3` สิ่งที่ URL Shortener ทำคคือการสร้าง URL ใหม่ที่สั้นกว่าเดิม เช่น `https://www.obsidian.com/abcdef` ที่เมื่อ User click URL นี้แล้วจะพาไปยัง URL เดิม

Q2: อยากรู้ว่ามี User Traffic แค่ไหน
A2: คิดว่าน่าจะ 100,000,000 URL ต่อวันนะ

Q3: URL ใหม่ที่ได้สามารถใช้อักขระอะไรได้บ้าง ละมันควรยาวได้แค่ไหน
A3: มีตัวเลข, ตัวอังกฤษพิมพ์เล็ก, และตัวอังกฤษพิมพ์ใหญ่อ่ะ ละก็ไม่ควรเกิน 100 ตัวอักษรนะ

Q4: URL ที่สร้างไปแล้วสามารถถูกแก้ไขหรือลบออกจากระบบได้มั้ย
A4: เอาเป็นว่าไม่ได้ละกัน เดี๋ยว User งง

เพื่อให้เห็นภาพมากขึ้นว่าทำไม Constraint เป็นงี้

- 100,000,000 URL ต่อวัน = 1160 URL ต่อวินาที ถ้า Service นี้มีการทำงานตลอด 1 ปี เท่ากับว่าจะต้องเก็บ URL ไว้ถึง 36.5 พันล้าน URL อ่ะ
- ถ้า URL 1 อันยาว 100 ตัวอักษร เท่ากับว่าจะใช้เนื้อที่ประมาณ 3.65 TB เลยด้วย ต่อ 1 ปี

---

## 2. Sketch ภาพระบบคร่าวๆ

ในระบบนี้มันจะมี 2 API หลักๆ ที่เกี่ยวข้อง คือ URL Shortening API ไว้สำหรับย่อ URL ที่ User ส่งมา กับ URL Redirecting API ที่เมื่อ User เอา API ของ Service เราไปแปะ ก็ต้องได้เป็น URL ตัวเต็มเพื่อ redirect ไปยังที่ที่ต้องการ

#### 1. URL Shortening API

```swagger
POST api/v1/data/shorten
	request parameter: {
		longUrl: string
	}
	return: {
		shortUrl: string
	}
```

โดย Flow ของการทำงานจะเป็นประมาณนี้
![[url_shortener_shortening.png]]

หมายความว่า Database ควรจะเก็บข้อมูลเป็นคู่กันระหว่าง ShortURL: Key (จะเก็บทั้ง ShortURL ไปเลย หรือจะเก็บจะ Hash Value ก็ได้) และ LongURL: Value เพื่อใช้สำหรับ Redirect

#### 2. URL Redirecting API

```swagger
GET api/v1/shortUrl
	return: {
		longUrl: string
	}
```

โดย Flow ของการทำงานจะเป็นประมาณนี้
![[url_shortener_redirecting.png]]

> [!Note] Status 301 คืออะไร?
> **301 Moved Permanently** ใช้บอก Client ว่า URL นี้ถูกย้ายไปยัง URL ใหม่แบบถาวร ดังนั้น Client/Browser สามารถ cache การ redirect นี้ไว้ได้ และในบางกรณี Search Engine จะถือว่า URL เดิมถูกย้ายไป URL ใหม่อย่างถาวร
> ส่วนอีกตัวที่คู่กันคือ **302 Found (หรือ Temporary Redirect)** ใช้บอกว่า URL นี้ redirect ไปยัง URL อื่น **ชั่วคราว** โดย Client ไม่ควรถือว่าการเปลี่ยนแปลงนี้เป็นการย้ายถาวร
>
> - **301** → เหมาะถ้า Short URL จะ redirect ไปยัง Long URL เดิมตลอดไป และต้องการให้ Browser/Cache ลดจำนวน request ที่ส่งมายัง URL Shortener
> - **302** → เหมาะถ้าต้องการให้ทุกครั้งที่ User เข้า Short URL ระบบยังมีโอกาสรับ request และทำการ redirect เอง เช่น ต้องการเก็บข้อมูลจำนวน Click หรือทำ Analytics

---

## 3. เจาะลึกจุดสำคัญๆ

#### Data Schema

เรื่อง Schema นี่...มีรูปแบบที่ทำง่ายๆ อยู่ ตามภาพนี้เลย

![[url_shortener_schema.png]]

จะมีจุดที่น่าสังเกตอยู่ 1 จุดคือการใช้ ShortUrl เป็น varchar หมายความว่าเราจะเก็บค่า shortUrl เป็น Hash Value พอ ไม่ต้องเก็บทั้ง URL (เพราะ Prefix มันเหมือนกันหมดอ่ะนะ คือ Domain ของ Service เราอ่ะ)

#### Hash Function

ก่อนอื่นเลย เราควรจะทำให้ Hash Value มีความยาวแค่ไหนดี คำตอบคือ...

> _เราก็วนกลับมาที่ Requirement ของเราก่อน..._

ด้วยความที่อักขระที่เราใช้ได้ คือ [0-9, a-z, A-Z] หมายความว่าเรามีตัวอักษรรวมทั้งหมด 62 ตัว และเราต้องการให้ Service นี้มีการใช้งาน 1 ปี หรือก็คือมี URL ทั้งหมด 36.5 พันล้าน URL

ดังนั้นเราต้องใช้ Hash Value ที่มีความยาวอย่างน้อยที่สุด 7 ตัว (62^7 > 36.5 billion) ถึงจะพอกับความต้องการของ Service นี้

แล้วเราจะทำยังไงให้ได้ Hash Value ที่มีความยาว 7 ตัวนะ... เพราะ Hash Function ปกติมันจะให้ความยาวที่เกิน 7 หมดเลย ตามตารางนี้
![[url_shortener_hash_function.png]]

หรือจะใช้ก็ได้ แบบว่า...เอาแค่ 7 ตัวแรกงี้ ละก็เช็คก่อนว่ามี Hash Value นี้ใน Database หรือยัง ถ้ามีแล้วก็ต้องแก้ LongURL ให้เป็นตัวใหม่แล้ว hash เป็น Hash Value ใหม่ แล้วตัดแค่ 7 ตัวแรกมาเช็ค ทำซ้ำเรื่อยๆ จนกว่าจะไม่ซ้ำ ซึ่งมันก็ดูไม่ค่อย effective เท่าไหร่นะ...

#### Base62

วิธีที่ basic กว่านั้นคือการทำเป็นเลขฐาน 62 (เลขฐานตามจำนวนอักขระที่เรามีใน List) โดยที่ 0_62 = 0, 1_62 = 1, ... , a_62 = 10, b_62 = 11, ..., A_62 = 36, B_62 = 37, ..., Z_62 = 61 เป็นต้น

จากนั้นก็นำเลข id ใน database ที่ generate ได้มาทำเป็นเลขฐาน 10 แล้วเปลี่ยนเป็นเลขฐาน 62 อ่ะ ตามภาพนี้เลย
![[url_shortener_base62.png]]

อย่างในภาพนี้ จะได้ว่า Hash Value คือ 2TX นั่นเอง

#### Comparison

เปรียบเทียบลักษณะของวิธี Hash Function กับ Base62 ได้ตามตารางนี้

| Hash Function + Collsion                                     | Base62 Conversion                                                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| ความยาวของ Short URL **คงที่**                               | ความยาวของ Short URL **ไม่คงที่** และจะเพิ่มขึ้นตามค่า ID                                   |
| ไม่จำเป็นต้องมี Unique ID Generator                          | **ต้องพึ่งพา Unique ID Generator**                                                          |
| มีโอกาสเกิด **Collision** และต้องมีกลไกจัดการ Collision      | **ไม่เกิด Collision** เพราะ ID มีค่าไม่ซ้ำกัน                                               |
| ไม่สามารถคาดเดา Short URL ถัดไปได้ เพราะไม่ได้ขึ้นอยู่กับ ID | สามารถคาดเดา Short URL ถัดไปได้ง่าย หาก ID เพิ่มขึ้นทีละ 1 ซึ่งอาจเป็น **Security Concern** |

Flow ของ Service จะเป็นตามนี้
![[url_shortener_flow.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ URL Shortener ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
