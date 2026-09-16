---
title: 03 Password Security
tags:
  - systemdesign
  - authentication
  - security
  - password
  - salt_pepper
  - hash_function
---
ก่อนหน้านี้ในเรื่อง [[01 Access Token|Access Token]] กับ [[02 Refresh Token|Refresh Token]] เรามี User Schema ที่เก็บ Email และ Password ของ User เนอะ เพื่อใช้เป็นข้อมูลในการยืนยันตัว User แต่ว่าการเก็บข้อมูลพวกนี้เข้าไปตรงๆ เลยเนี่ย เกิดระบบ leak ขึ้นมาแล้วข้อมูลเหล่านี้หลุดออกไป มีได้บันเทิงกันแน่ๆ

Goal จริงๆ ของ Email และ Password คือการ verify ว่าคนที่พิมพ์ Email นี้เข้ามารู้ Password มั้ย แปลว่า จริงๆ แล้วระบบไม่จำเป็นต้องเก็บค่า Password แบบตรงๆ ก็ได้ เป็นค่าแทนค่าจริงที่ convert กลับมาเป็นค่าจริงได้ก็สามารถใช้ได้เหมือนกัน แล้วถ้า leak ออกไปก็ทำให้ attacker ได้ค่าแทน ซึ่งไม่ใช่ค่าจริงๆ ไป จึงปลอดภัยกว่าการเก็บค่า Password จริง

แล้วเราจะสร้างค่าแทนค่าจริงเหล่านั้นยังไง? -> Hash Function ยังไงหล่ะ

---

## 1. Hash Function ทำงานยังไง?

Hash Function คึอ Function ที่รับข้อมูลออกมาแล้วคืนผลลัพธ์ออกมาเป็นชุดอักษรที่มีความยาวคงที่ ที่มันเหมาะเพราะ 3 เหตุผล
1. Input เดิมให้ Output เดิม ทำให้เทียบได้ว่า Input ที่ User กรอกมา พอแปลงด้วย Hash Function แล้ว สามารถนำ Output ที่ได้มาเทียบกับที่เก็บใน Database ว่าตรงกันมั้ยได้
2. Hash Function ย้อนกลับไม่ได้ คือหมายความว่า Output ที่ได้จะไม่เหลือร่องรอยอะไรให้สาวไปถึง Input ได้
3. การแก้ Input เพียงนิดเดียว ทำให้ Output ที่ได้ต่างจาก Output เดิมคนละโขเลย

>[!Caution] Hash ไม่ใช่ Encryption เพราะการจะเรียก Encryption ได้ คือเราจะต้อง decrypt ข้อมูลกลับไปเป็นข้อมูลก่อนจะถูก encrypt ได้ แต่ Hash Result ทำแบบนั้นไม่ได้

---

## 2. ทำไม Hash Function ที่แปลงค่าได้เร็วจึงไม่เหมาะ?

ฟังดูแปลกๆ นะ คือถ้า Function มันแปลง Input เป็น Output ได้เร็วๆ ก็น่าจะฟังแล้วดูดีมะ เช่น SHA-256 หรือ MD5 งี้ มันแปลงได้เร็วมาก แถมเป็น Hash Function ที่ให้คุณสมบัติตรงกับทั้ง 3 ข้อเลยนะ...

ไอ่การที่มันทำได้เร็วนี่แหละมันอันตราย เพราะต่อให้ attacker ได้ข้อมูลที่เป็นตัวแทนไป แล้วจะ convert กลับมาเป็นข้อมูลที่เป็น Input ได้ แต่ว่ามันก็มีวิธีทื่อๆ กว่านั้นที่ attacker สามารถทำได้ คือนั่งสุ่มตรงๆ จนกว่าจะถูก แบบว่า...ปล่อยบอทให้ไล่ทุกอักขระบนแป้นไปเรื่อยๆ จาก Input 1 ตัว เป็น Input 2 ตัว 3 ตัว 4 ตัว ไล่ไปเรื่อยๆ จนกว่าจะเจอ Input ที่ให้ Output ตรงกับที่ attacker ได้มา ก็จบละ

ยื่งสมัยนี้มี GPU ที่แปลง Input เป็น Output ได้พันล้านครั้งต่อวินาทีอีก แปปเดียวเดะก็ได้ละ XD

---

## 3. Rainbow Table

นอกเหนือจากการแปลง Input เป็น Output ที่เร็วเกินไปแล้วเนี่ย... ไอ่การที่ Input เดิมให้ Output เดิมมันก็ทำให้เกิดปัญหาด้านความปลอดภัยเหมือนกัน

เพราะว่าถ้ามี User ที่ใช้รหัสเหมือนกัน ก็กลายเป็นว่า 2 คนนี้ได้ Output เหมือนกัน พอ attacker เจอ Input ของ User คนนึงแล้ว ดันกลายเป็นว่าก็เอา Input นี้ไปสวมรอยเป็น User อีกคนนึงได้ด้วย

หรือกระทั่งว่า attacker สามารถทำตารางที่จด Input, Output ที่ตนเองเคยใช้ไปได้ พอเจอ Output ก็ลองเช็คจากตารางนี้ก่อน ถ้าเจอก็ใส่ Input ได้เลย ก็สวมรอยได้ชิวเลยงี้ เราเรียกตารางนี้ว่า Rainbow Table

ฟังดูแล้ว Hash Function ก็อาจจะไม่ใช่ Solution ที่ใช่แล้ว...รึเปล่านะ?

---

## 4. Salt and Pepper

ในเมื่อ Input เดิมให้ Output เดิม เราก็ทำให้ Input มันไม่เหมือนเดิม เราก็จะได้ Output ที่ไม่เหมือนเดิมแล้ว เป็นไงหล่ะ จีเนียสป่ะหล่ะ xd

แต่ว่าเราคงจะให้ User ที่ใช้รหัสเหมือนกันมาเปลี่ยนรหัสให้ต่างกันก็คงไม่ได้อ่ะนะ ยิ่ง User มีเยอะ โอกาสที่รหัสจะเหมือนกันก็ยิ่งมีเยอะอ่ะ งั้นทำยังไงให้ Input มันไม่เหมือนกันได้นะ...

หลักการคือ ก่อนจะให้ Hash Function แปลง Input เป็น Output ก็ให้ระบบเติม "ข้อความสุ่ม" ไม่ซ้ำกันให้กับ Password ของ User ก่อนจะส่งให้ Hash Function แปลง แล้วให้ Database เก็บข้อความสุ่มนี้ไว้ คู่กับ Password ที่ User ใส่มา เป็น 2 Column นั่นเอง -> เราเรียกข้อความสุ่มนี้ว่า Salt

![[auth_password_salt.png]]

ด้วยการเติมข้อความแบบสุ่มนี้ แม้ว่า User จะใช้ Password ที่เหมือนกัน แต่ก็จะได้ว่า Hash Function ให้ผลลัพธ์ที่เหมือนกัน สำหรับ User ก็ไม่ได้เกิดความแตกต่างอะไร แต่ว่าสำหรับ attacker มันก็เพิ่มความยากเข้าไปทันที เพราะจะกลายเป็นว่าแทบจะไม่มี User คนไหนที่ใช้ Input ที่เหมือนกันเลย เพราะ Input มันประกอบด้วย Input จริงๆ ของ User กับข้อความสุ่มจากระบบ ทำให้โอกาสการจะใช้ Input เดียวแล้วสวมรอยได้หลาย User เนี่ยก็แทบไม่มีเลย -> Rainbow Table ก็ใช้ไม่ได้ละ

แต่มันก็จะยังวนกลับไปที่เดิมว่า ถ้าระบบมัน Leak ขึ้นมา มันก็จะเห็น Salt แล้ว attacker ก็ยังสามารถ brute force ได้อยู่ดี เพราะงั้นเราก็ต้องมีข้อความสุ่มอีกรูปแบบนึงที่ไม่เก็บใน Database และมีเพียง Organization ที่รู้ เราเรียกข้อความสุ่มอีกแบบนี้ว่า Pepper

พอมี Pepper แล้วเนี่ย ก็กลายเป็นว่า attacker จะขาดข้อมูลสุ่มไปส่วนนึง เพราะถึงจะได้ Output กับ Salt ไป ก็ยังขาดไอ่ตัวนี้ที่ Developer เป็นคนสร้างขึ้นมา ทำให้การที่จะ Brute Force หา Password ได้ก็แทบเป็นไปไม่ได้อีกเช่นกัน

---

## 6. Cost Factor

พอเราแก้เรื่อง Brute Force ได้แล้ว ย้อนกลับมาที่ปัญหาแรกที่ว่า Hash Function มันทำงานเร็วไป ทำให้ attacker มี throughput เยอะ เราจะทำยังไงดี

มันจะมี Hash Function สำหรับ Password โดยเฉพาะอย่าง bcrypt, scrypt และ argon2 ที่มี Cost Factor หรือตัวเลขที่ Developer สามารถกำหนดได้ว่าจะให้ Hash Function คำนวณให้ Output มันออกมาซับซ้อนแค่ไหน โดยแลกกับเวลาที่ต้องใช้เพิ่มขึ้น

มันเป็นตัวเลขที่ปรับได้ เพราะงั้นถึง Hardware จะเร็วแค่ไหน เราก็ปรับ Cost Factor ให้สูงขึ้น เพื่อให้ attacker ต้องเสียเวลาในการหา Password ก็พอแล้ว

มี Code มานำเสนอ

```python
import hashlib
import time
import bcrypt

def run(make_hash, duration=2):
    count = 0
    start = time.time()
    while time.time() - start < duration:
        make_hash(count)
        count += 1
    elapsed = time.time() - start
    return count, elapsed

def md5_hash(i): hashlib.md5(str(i).encode()).hexdigest()
def sha1_hash(i): hashlib.sha1(str(i).encode()).hexdigest()
def sha512_hash(i): hashlib.sha512(str(i).encode()).hexdigest()
def bcrypt_hash(i):
    salt = bcrypt.gensalt()
    bcrypt.hashpw(str(i).encode(), salt)

algorithms = [("MD5", md5_hash), ("SHA1", sha1_hash), ("bcrypt", bcrypt_hash), ("SHA512", sha512_hash)]
times = 5

for i in range(1, times+1):
    print(f"Run #{i}")
    print(f"{'Algorithm':<10} {'#Hashes':>10} {'Time(s)':>10} {'#Hashes/s':>15}")
    for name, func in algorithms:
        count, elapsed = run(func)
        rate = count / elapsed
        print(f"{name:<10} {count:>10} {elapsed:>10.3f} {rate:>15,.1f}")
    print()
```

Code นี้คือการ simulate ว่าแต่ละ hash function ใช้เวลาในการแปลง Input เป็น Output เป็นยังไง ระหว่าง SHA1, SHA512, MD5 ละก็ bcrypt โดยผลลัพธ์จะเป็นงี้

![[auth_password_hash_speed.png]]

ในเวลาที่เท่ากัน ประมาณ 2 วินาที bcrypt สามารถสร้าง Output ได้เพียงระดับหลักหน่วย ในขณะที่แบบอื่นปาไปหลักแสนหรือหลักล้านกัน เพราะงั้นการที่ attacker จะลอง Brute Force ด้วย bcrypt เนี่ย บอกเลยว่ายากส์ และช้ากว่าเดิมระดับแสนเท่ากันเลย

---

## 7. Application

พอเราจะ apply หลักการ password security เข้าไปจริงๆ มันก็หนีไม่พ้น Login Flow กับ Reset Password Flow อ่ะเนอะ แต่ว่ามันอาจจะเป็นรายละเอียดเล็กๆ น้อยๆ ที่ Developer อาจจะหลงลืมไป

### Login Flow

ในทางปฏิบัติเนี่ย มันก็ต้องเอา Password ที่ User ใส่ มาผ่าน Hash Function แล้วได้ออกมาเป็น Output แล้วเทียบกับใน Database เนอะ แต่ว่าไอ่คำว่าเทียบเนี่ย อยากจะบอกว่ามันเป็นช่องโหว่ให้ attacker ได้เหมือนกันนะ กะอีแค่บอกว่าตรง/ไม่ตรงอ่ะ

หลักการคือ เวลาเราเทียบ เราก็ไล่เช็คทีละตัวเนอะ ว่า Char 1 ของ Output1 กับ Output2 ตรงกันมั้ย ไล่ไปจนถึง Char สุดท้าย ซึ่งแปลว่าการไล่เช็คแล้วเจอว่าไม่ใช่ตอนช่วง Char ท้ายๆ ใช้เวลาเยอะกว่าเจอว่าไม่ใช่ตอนช่วง Char แรกๆ

ด้วยความต่างของเวลาเพียงเสี้ยววินี่แหละที่ attacker สามารถใช้เดา Password ได้ เรียกว่า Timing Attack เพราะงั้นทางแก้คือ ระบบควรจะต้องเทียบ Password แบบ Constant Time คือไม่ว่าจะเปรียบเทียบกันที่ Character ตัวที่เท่าไหร่ ก็ต้องใช้เวลาเท่าเดิมเสมอ

อีกจุดคือ Wording: การบอกว่าไม่พบ Email นี้ หรือรหัสผ่านไม่ถูกต้อง มันเป็นการให้ข้อมูลทั้ง User และ attacker ซึ่งก็เสี่ยงอันตรายอ่ะนะ เลยควรเป็น "Email หรือรหัสผ่านไม่ถูกต้อง" จะดีกว่า เพราะในมุม User มันก็ไม่ต่างอะไรจากเดิม แต่กับ attacker มันต่างนะ เพราะไม่รู้ว่าอะไรผิดอ่ะ

### Reset Password Flow

เวลา User ลืม Password เนี่ย... สิ่งที่ระบบทำได้คือการสร้าง Reset Password Token สำหรับ Email นั้นๆ ให้กับ User ไป เช่น www.test.com/reset?token=abc123

พอเข้าหน้า UI สำหรับแก้ Password แล้วก็ให้ User กรอก Email ละก็ Password ใหม่ที่จะใช้ไป ละก็ให้ระบบ verify ข้อมูล

โดยการ verify นี้ จะ verify ว่า token ที่ใช้อย่าง abc123 เนี่ย มันเป็นของ Email ที่ User กรอกมารึเปล่า ถ้าไม่ใช่ก็ไม่ให้ reset password แต่ถ้าใช้ ก็จะให้ reset password ได้

ตัว Reset Token นี้ไม่ควรที่จะใช้งานได้นาน และใช้ได้ครั้งเดียว ถ้าใช้ reset password สำเร็จแล้วก็ควรจะถูก revoke ทันที เพราะงั้น Schema เลยจะเป็นแบบนี้

![[auth_password_reset_password_token_schema.png]]

สุดท้าย คือเวลา Reset Password แล้ว เราอย่าลืมที่จะ revoke Refresh Token ของ User ไปด้วยนะ เพราะถ้าไม่ revoke จะทำให้ attacker ก็ยังสวมรอยเป็น User ได้อยู่ดี

---

