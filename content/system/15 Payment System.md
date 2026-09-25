---
title: 15 Payment System
tags:
  - system
  - systemdesign
  - payment_system
  - webhook
  - psp
  - reconciliation
  - communication
  - retry
  - idempotency
---
รอบนี้เรามาระบบที่ยิ่งใหญ่ ละก็สำคัญมากๆ ระบบนึงหน่อยละกัน ก็คือ Payment System นั่นเอง สำคัญมั้ยหล่ะ เงินๆ ทองๆ อ่ะ พลาดขึ้นมานี่ชิบหายได้เลยนะ จะทำยังไง มาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Payment System ที่จะทำนี่มันคือยังไงเหรอครับ?
A1: ก็คือสร้าง Backend สำหรับ E-Commerce Website เช่น Amazon อ่ะแหละ เวลาลูกค้าเลือกของสำเร็จแล้วกดปุ่มชำระเงินอ่ะ ก็คือเริ่มตั้งแต่ตรงนั้นจนจบที่ลูกค้าชำระเงินสำเร็จเลย

Q2: จะให้ลูกค้าชำระเงินด้วยอะไรได้บ้างเหรอครับ? Credit Card, Paypal, Truemoney หรือมีอะไรบ้างครับ?
A2: จริงๆ ก็ควร support หมดนะ แต่ตอนนี้เอาแค่ Credit Card ก่อนละกัน

Q3: แล้วเราต้องจัดการเรื่องการชำระเงินเองมั้ยครับ
A3: อ่า...ไม่ต้องละกัน ใช้ Third Party แบบ Stripe, Braintree, หรือ Square ก็ได้

Q4: แล้วเราสามารถเก็บข้อมูล Credit Card ได้มั้ยครับ
A4: ไม่ควร

Q5: แล้วเราต้อง support ค่าเงินที่หลากหลายด้วยมั้ยครับ
A5: จริงๆ ก็ควรนะ แต่ว่าตอนนี้เอาแค่หน่วยเงินเดียวทั้งระบบก่อนละกัน

Q6: คิดว่าจะมี Transaction เยอะขนาดไหนเหรอครับ
A6: ประมาณ 1 million transactions ต่อวันอ่ะ

Q7: ถ้าเป็นเหมือน E-Commerce แปลว่าก็จะมีคนเอาของมาลงใช่มะ ระบของเราควรจะ support เรื่องการเอาเงินจากของที่ขายได้มาให้กับคนที่เอาของมาลงด้วยมั้ยครับ
A7: ก็ควรนะ ดีเลยแหละ

### Functional Requirement

- Pay-in Flow: ระบบจะเก็บเงินจากคนที่ซื้อของได้
- Pay-out Flow: ระบบจะส่งเงินให้คนขายได้

### Non-Functional Requirement

- Reliability and Fault Tolerance: ระบบจะต้องไม่เกิดข้อผิดพลาดในการชำระเงิน

---

## 2. Sketch ภาพระบบคร่าวๆ

เผื่อว่ายังงงๆ ว่า E-Commerce มันทำงานอะไรยังไง ละก็อะไรคือ Pay-in อะไรคือ Pay-out ก็เอาภาพนี้ไป

![[payment_system_overview.png]]

ระบบของ E-Commerce จะแบ่งเป็น 2 จังหวะ
1. เวลาที่ลูกค้าซื้อของ แล้วจ่ายเงินสำเร็จ เงินจะเข้าไปอยู่ในบัญชีของ E-Commerce Website -> Flow ตรงนี้เราจะเรียกว่า Pay-in
2. เมื่อมีเงินเข้ามาสู่บัญชีของ E-Commerce Website ก็จะมีการนำเงินนี้ไปให้คนขายอย่างถูกต้อง -> Flow ตรงนี้เราจะเรียกว่า Pay-out

เพราะงั้นเราจะแบ่งการ Sketch ออกเป็น 2 ส่วนละกัน ก็คือ Pay-in และ Pay-out

### Pay-in System

![[payment_system_pay-in.png]]

Flow ของระบบจะเป็นประมาณนี้...
1. เวลาที่คนซื้อกดชำระเงิน ระบบก็จะสร้าง Payment Event มาให้ Payment Service ที่มีข้อมูลต่างๆ เช่น สินค้าที่ลูกค้าซื้อ จำนวนเงินที่ชำระ วิธีการชำระเงิน เป็นต้น
2. Payment Service ก็จะเอาข้อมูล Payment ที่ได้รับเก็บไว้ใน Database (เหมือนเป็น Log ว่ามีใครบ้างกดชำระเงินอ่ะ)
3. Payment Service จะแบ่งข้อมูล Payment ออกเป็น **Payment Order** ตามคนขาย (คนซื้อ 1 ครั้งอาจซื้อของจากคนขายหลายคน ก็แตกเป็นหลาย Payment Order) แล้วส่งแต่ละ Payment Order ให้ Payment Executor จัดการทีละอัน เพื่อให้คนขายแต่ละคนได้รับเงินในส่วนที่ควรได้
4. Payment Executor เอาข้อมูล Payment Order เก็บใส่ Database ไว้ด้วย
5. Payment Executor ส่งต่อข้อมูล Payment Order ให้ PSP (Payment Service Providers) ที่เป็น Third Party ไปจัดการต่อ รอสถานะการชำระเงินจาก PSP

>[!Note] Payment Service Providers (PSP)
>PSP มีหน้าที่ในการย้ายเงินจาก Account A ไปหา Account B

6. หลังจาก PSP แจ้งว่าชำระเงินสำเร็จ Payment Executor ก็ส่งสถานะการชำระเงินกลับมาให้ Payment Service จากนั้นก็ส่งข้อมูลให้ Wallet Servers update เรื่องการเงินของคนซื้อ และคนขาย
7. Wallet Servers บันทึกข้อมูลการเงินของคนซื้อ และคนขายเข้าใน Database
8. หลังจาก Wallet Servers บันทึกข้อมูลการเงินสำเร็จ Payment Service ก็จะส่งข้อมูลให้ Ledger Servers เพิ่มข้อมูล Transaction ระหว่างคนซื้อและคนขาย
9. Ledger Servers เพิ่มข้อมูล Transaction ระหว่างคนซื้อกับคนขายเข้าไปใน Database

#### API Design

1. **POST** `/v1/payments` สำหรับเริ่ม flow การชำระเงิน โดยจะมี parameters ต่อไปนี้
	- `id`: id การชำระเงิน
	- `buyer_info`: ข้อมูลของคนซื้อ -> json
	- `credit_card_info`: ข้อมูลบัตรเครดิต -> json
	- `payment_orders`: รายการของที่ซื้อ -> list
		- `seller_info`: ข้อมูลคนขาย -> json
		- `amount`: ยอดขาย -> string
		- `currency`: หน่วยเงิน -> string
		- `payment_order_id`: id รายการจ่ายแยกตามคนขาย

การมี id ส่งไปให้ ทั้ง id ของการชำระเงินหลัก ละก็ในรายการแยกตามคนขาย จะช่วยในเรื่อง Idempotency ป้องกันไม่ให้ PSP ทำรายการซ้ำนั่นเอง แบบว่า...สามารถใช้ id เช็คได้ว่าเคยทำรายการนี้ไปหรือยัง

ส่วนสาเหตุที่ใช้ amount เป็น string ก็เพื่อป้องกัน precision ที่มันอาจจะคาดเคลื่อนไปได้ (เช่น case ของ JS ที่ 0.1 + 0.2 ไม่ได้เป็น 0.3 อ่ะ) อีกอย่างนึงคือ เลขเงินในหลายประเทศมันเยอะมากๆ หรือน้อยมากๆ การใช้เป็น double มันอาจจะบรรจุเลขไม่พอได้ ก็เก็บเป็น String ละก็ค่อย parse เอาตอนจะคำนวณก็ง่ายกว่า

2. **GET** `/v1/payments/:id` ใช้สำหรับเช็คข้อมูลของ `payment_order_id` 

#### Schema

หน้าตาของ Payment Event Schema กับ Payment Order Schema จะเป็นประมาณนี้

![[payment_system_schema.png]]

โดยที่ status ใน Payment Order Schema ก็มี `NOT_STARTED` `EXECUTING` `SUCCESS` ละก็ `FAILED` 

`is_payment_done` จะเป็น True ก็ต่อเมื่อทุก Payment Order ที่ `payment_orders.event_id = payment_event.id` มี status เป็น `SUCCESS` 

### Pay-out System

Pay-out ก็คล้ายๆ กับ Pay-in นะ โดยใช้ PSP ในการ move เงินจาก E-Commerce Account ไปหา Account ของคนขาย ที่เหลือก็คล้ายๆ กับ Pay-in เลย

---

### 3. เจาะลึกจุดสำคัญๆ

ด้วยความที่เป็นเรื่องเงินๆ ทองๆ เพราะงั้นจะเน้นเรื่องของความปลอดภัย น่าเชื่อถือแม้จะเกิด cases ต่างๆ เช่นการกด pay หลายๆ ครั้ง หรือตอนที่ network ช้า เป็นต้น

### PSP Integration

เราจะมาเพิ่มเติมในส่วนของการทำงานของ PSP ว่ามีการทำงานยังไง

![[payment_system_psp_flow.png]]

1. User กดชำระเงิน เป็นการสร้าง Payment Event ให้กับ Payment Service
2. Payment Service ส่งข้อมูลให้ PSP โดยจะมี `nonce` (สามารถใช้เป็น payment_event.id เลยก็ได้)เป็น id ที่ป้องกันไม่ให้ PSP ทำ transaction รายการซ้ำ รวมถึงมี `redirect_url` ให้ไป เพื่อให้ PSP พา User ไปหน้าที่ชำระเงินแล้วได้อย่างถูกต้องเมื่อ Transaction เสร็จสิ้น
3. เมื่อ PSP ได้รับข้อมูลมา ก็จะส่ง Payment Token คืนกลับไปให้ Payment Service โดย Payment Token นี้จะเป็น UUID จาก PSP ที่ไว้เช็คสถานะของ Transaction ใน PSP ได้
4. Payment Service เก็บ Payment Token ไว้ใน Database
5. เมื่อเก็บ Payment Token แล้ว ระบบจะพา User ไป Payment Page ของ PSP ด้วย Payment Token ที่ได้รับมา

![[payment_system_payment_page_psp.png]]

6. User ใส่ข้อมูล Credit Card ของตนเอง แล้วเริ่มการชำระเงิน
7. PSP ส่งสถานะการชำระเงินว่าสำเร็จหรือไม่กลับมา
8. พอ User จ่ายเสร็จบนหน้า PSP, PSP จะพา Browser ของ User **redirect กลับมาที่ `redirect_url`** ที่เราส่งไปให้ตั้งแต่ข้อ 2 โดยแนบสถานะการจ่ายมากับ URL ด้วย เช่น `https://your-company.com/?tokenID=JIOUIQ123NSF&payResult=X324FSa` หน้านี้จะเป็นหน้า checkout ของ E-Commerce ที่โชว์ให้ User เห็นว่า "จ่ายสำเร็จแล้วนะ"
9. Webhook ของ PSP ส่ง request มาแบบ asynchronous กลับมาให้ Payment Service เพื่อแจ้งสถานะ Transaction เพื่อให้ Payment Status เอาสถานะที่ได้ไป **update `payment_order_status`** ใน Database

> [!important] Redirect URL ≠ Webhook
> 
> - **Redirect URL** - Browser ของ User เด้งกลับมา ไว้ให้ **User เห็น** ผล อาจไม่มาถ้า User ปิดจอ
> - **Webhook** - PSP ยิงมาหา **Server เรา** โดยตรง ไว้ให้ **ระบบ** อัปเดตสถานะจริง มาเสมอไม่ว่า User จะปิดจอหรือไม่
> 
> เพราะงั้นระบบต้องยึด Webhook เป็นแหล่งความจริง ไม่ใช่ Redirect URL

### 2. Reconciliation

Reconciliation คือการตรวจเช็คว่าข้อมูลมีความถูกต้องตั้งแต่ต้นจนจบ เป็นหัวใจที่โจทย์อื่นไม่มี เพราะระบบสื่อสารกันแบบ asynchronous เยอะ (ทั้งภายใน และกับ PSP/ธนาคาร) จึงไม่มีอะไรการันตีว่าทุก message จะถึงและตรงกันเสมอ

![[payment_system_reconciliation.png]]

วิธีการคือ ในแต่ละคืน PSP หรือธนาคารจะส่ง **Settlement File** มาให้ Payment System โดยจะเป็นไฟล์ที่มียอดคงเหลือ รวมถึงรายการ transaction ของวันนั้น ทำให้ Payment System สามารถนำ Settlement File มาเช็คกับ Ledger ของระบบเราได้ว่าตรงกันหรือไม่

ถ้ารายการไม่ตรงกัน ก็ดูเป็น Case ไป
1. รู้สาเหตุที่รายการไม่ตรงกัน แล้วทำ automated ได้ -> เขียน Automated Scripts แก้ไขไป
2. รู้สาเหตุที่รายการไม่ตรงกัน แต่ว่ามัน automated ไม่ได้ หรือได้แต่ยากเกิน -> ส่งให้ทีมงานไปแก้แบบ Manual ให้รายการตรงกัน
3. ไม่รู้สาเหตุ -> เรียกทีมงานมาตรวจสอบอย่างเร่งด่วน

### 3. Handling Payment Processing Delays

บาง payment ไม่ได้เสร็จในไม่กี่วินาที อาจใช้เวลาเป็นชั่วโมงหรือวัน เช่น PSP มองว่า Payment Event นี้มีความเสี่ยงเลยส่งให้คนรีวิว หรือบัตรต้องยืนยันตัวตนเพิ่มด้วย **3D Secure** เป็นต้น

วิธีรับมือคือ PSP จะคืนสถานะ `PENDING` กลับมาให้ Payment System ก่อน เพื่อให้ระบบของเรานำไปแสดงให้ User เห็นพร้อมหน้าเช็คสถานะ แล้วรอให้ PSP ยิง webhook มาบอกตอนเสร็จจริง

เพราะงั้น Payment System ของเราก็ต้องมีการ design UI และหลังบ้านให้รองรับ payment event ที่ใช้เวลานานได้ ไม่ใช่ถือว่า fail ถ้าไม่เสร็จในระยะเวลาสั้นๆ

### 4. Communication among Internal Services

ด้วยความที่มันมีการส่งข้อมูลข้าม Services กันระหว่าง Payment Services กับ PSP เพราะงั้นก็เลี่ยงไม่ได้ที่จะต้องมาเลือกว่า Synchronous หรือ Asynchronous ดี?

Recap สั้นๆ (อ่านรายละเอียดเพิ่มเติมได้ใน [[14 Synchronous VS Asynchronous|Syn VS Asyn]])

- **Synchronous Communication** (เช่น HTTP) implement ง่ายแต่พอ scale ใหญ่จะมีปัญหา performance, failure isolation แย่ เหมาะกับ scale งานที่เล็กๆ คนใช้งานไม่เยอะ
- **Asynchronous Communication** ผ่าน Queue แบ่งเป็น Single Receiver (message ถูกกินแล้วหายจาก queue) กับ Multiple Receiver (เช่น Kafka ที่ message ไม่หาย ทำให้ service หลายตัวอ่านได้)

เพราะงั้น สำหรับ Payment ที่มี business logic ซับซ้อนและพึ่ง third-party เยอะ ยิ่งมีคนใช้งานเยอะๆ อีก **Asynchronous เป็นทางเลือกที่ดีกว่า** ยอมแลกความเรียบง่ายกับ scalability ได้ และ failure resilience (ความเสียหายน้อยเวลาระบบล้มเหลว)

### 5. Handle Failed Payment

ก็เลี่ยงไม่ได้จริงๆ อ่ะนะ กับการที่จะมี Payment Event บางตัวไม่สำเร็จ แล้วเราจะรับมือหรือเตรียมการยังไงได้บ้าง มาดูกัน

![[payment_system_handle_failed_payment.png]]

ภาพรวมคือ เช็คก่อนว่า error นั้น **retry หรือลองทำซ้ำอีกครั้งนึงได้มั้ย** ถ้าเป็น error ชั่วคราว (เช่น network หลุด) ก็เข้า **Retry Queue** วนมาลองใหม่ ถ้า retry เกินจำนวนครั้งที่กำหนดแล้วยังไม่สำเร็จ ก็ส่งเข้า **Dead Letter Queue (DLQ)** เพื่อแยกออกมาให้ทีมสืบสวนว่าทำไมถึงไม่สำเร็จ ส่วน error ที่ retry ไม่ได้ตั้งแต่แรก (เช่น input ผิด) ก็เก็บลง Database ไป

### 6. Exactly-once Delivery

แน่นอนว่าการทำซ้ำรายการได้ หรือว่ามีการหักเงินซ้ำขึ้นมา ถือเป็นหายนะ เพราะมันจะทำให้เงินของลูกค้าหายไป ไม่ใช่เรื่องที่ดีแน่ๆ เราต้องทำให้ payment ทำงาน **exactly-once** หรือมั่นใจได้จริงๆ ว่ามันจะเกิดการหักเงินเพียงครั้งเดียวเท่านั้น ก็พูดยากเนอะ แต่ว่าเราเอาหลักการคณิตมาปรับใช้ได้อ่ะนะ

> ถ้าเรารู้ว่า x <= 1 และ x >= 1 เราก็สามารถสรุปได้เนอะว่า x = 1 

เพราะงั้นตรงนี้ก็คล้ายๆ กัน เราก็แบ่งเป็น 2 ส่วน คือ
- **at-least-once** (ทำอย่างน้อย 1 ครั้ง) ผ่านการทำ Retry ยิงจนกว่าจะสำเร็จ พอสำเร็จแล้วก็หยุดส่ง
	![[payment_system_retry.png]]

- **at-most-once** (ทำไม่เกิน 1 ครั้ง) ผ่านการทำ Idempotency เช็คว่าเคยทำรายการนี้สำเร็จไปแล้วหรือยัง โดยการเช็คที่ `payment_event.id` ว่าเคยทำไปแล้วหรือยัง
	![[payment_system_idempotency.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Payment System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---