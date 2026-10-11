---
title: 18 Digital Wallet
tags:
  - system
  - systemdesign
  - digital_wallet
  - idempotency
  - sharding
  - tc/c
  - cqrs
  - saga
  - 2pc
  - raft
  - database_partition
  - event_sourcing
---

วันนี้เรามาต่อเพิ่มเติมในเรื่อง Digital Wallet ละกัน เป็นเหมือนส่วนขยายจาก [[15 Payment System|Payment System]] ตรงในส่วนที่เป็น Paypal หรือ PSP อ่ะว่าทำงานยังไง เพราะงั้นเรามาดูกัน

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: ผมควร focus features ไหนบ้างเหรอครับ ใน Digital Wallet นี้?
A1: เอาแค่เรื่องการโอนเงินจาก Wallet A ไป Wallet B ก่อนละกัน

Q2: ถ้าเราสามารถสร้างระบบให้เกิด Transactional Guarantees ได้ ถือว่าครบถ้วนยังครับ หรือว่ามี Requirement อะไรเพิ่มเติมรึเปล่าครับ?
A2: ก็โอเคนะ

Q3: ระบบเรามี Availability 99.99% นี่...ถือว่าพอมั้ยครับ
A3: โอเคเลย

Q4: พวกเรื่องค่าเงินนี่... ต้องนำมาพิจารณามั้ยครับ?
A4: ไม่ต้องละกัน Assume ว่าหน่วยเงินเดียวกันทั้ง 2 Wallet

Q5: แล้วระบบต้องรองรับการโอนเงินเยอะขนาดไหนเหรอครับ  
A5: ประมาณ 1 ล้าน Transactions ต่อวินาที (TPS) ละกัน

---

## 2. Sketch ภาพระบบคร่าวๆ

### API Design

ระบบนี้ใช้ API แค่เส้นเดียว คือ **POST** `/v1/wallet/balance_transfer` สำหรับโอนเงินจาก Wallet หนึ่งไปอีก Wallet หนึ่ง โดยจะมี Parameter ประมาณนี้...

| Field          | Description                                   | Type   |
| -------------- | --------------------------------------------- | ------ |
| from_account   | บัญชีที่ถูกหักเงิน                            | string |
| to_account     | บัญชีที่เงินเข้า                              | string |
| amount         | จำนวนเงิน                                     | string |
| currency       | หน่วยเงิน                                     | string |
| transaction_id | ID สำหรับป้องกันการโอนซ้ำ ตามหลัก Idempotency | uuid   |

Note:

- `amount` เป็น string เหมือนกับที่อธิบายไว้ใน [[15 Payment System|Payment System]] เลย คือกันปัญหาความแม่นยำของทศนิยม
- `transaction_id` ทำหน้าที่เป็น Idempotency Key เพื่อป้องกันการที่ระบบจะทำ Transaction นั้นซ้ำ

แต่ถึง API มีแค่เส้นเดียวก็จริง แต่สิ่งที่ยาก คือ จะออกแบบระบบให้เก็บยอดเงินของทุก Wallet ไว้ที่ไหน และจะโอนเงินจาก Wallet หนึ่งไปอีก Wallet หนึ่งยังไงให้เงินไม่หายระหว่างทาง แถมยังต้องรองรับ 1 ล้าน TPS ด้วย

เราจะลองค่อยๆ ไล่ไปเรื่อยๆ ละกัน... พื้นฐาน คือ Wallet ก็เป็นการจับคู่ระหว่าง User กับยอดเงิน ซึ่งเก็บเป็น Key-Value ได้ตรงๆ เพราะงั้นวิธีการแรกที่ทำได้ คือ การทำ In-Memory Sharding

### In-memory Sharding

In-memory Sharding ก็คือการใช้ In-memory Database อย่าง Redis ในการเก็บข้อมูล และทำ Sharding เพื่อให้รองรับ TPS จำนวนมากได้ โดยจะกระจายบัญชีไปหลาย Node ด้วยการเอา `accountID` มา hash แล้ว mod ด้วยจำนวน Node ส่วนข้อมูลว่ามีกี่ Node และแต่ละ Node อยู่ที่ไหนก็เก็บไว้ใน Zookeeper

![[digital_wallet_in-memory_sharding.png]]

จากภาพ คือเวลาโอน 1 Dollar จาก A ไป B Wallet Service จะสั่ง 2 คำสั่งไปยัง 2 Node คือหัก 1 Dollar จาก A และเพิ่ม 1 Dollar ให้ B

**ปัญหาคือ** ไม่มีอะไรรับประกันว่าทั้ง 2 คำสั่งจะสำเร็จพร้อมกัน ถ้า Wallet Service ล่มหลังหักเงิน A ไปแล้วแต่ยังไม่ได้เพิ่มให้ B เงิน 1 Dollar จะหายไปเลย ทั้ง 2 คำสั่งจึงต้องเป็น **Atomic** คือสำเร็จทั้งคู่หรือล้มเหลวทั้งคู่ เพราะงั้นก็ต้องมีการทำให้ Transaction เป็น Atomic นั่นเอง ผ่านการทำ Distributed Transaction

### Distributed Transaction

ขั้นแรกคือเปลี่ยนจาก Redis มาเป็น Relational Database ที่รองรับ Transaction ทำให้ได้ ACID Properties แต่ก็ยังแก้ปัญหาได้ไม่หมด เพราะบัญชี A กับ B อาจอยู่คนละ Database ซึ่ง Transaction ปกติครอบข้ามกันไม่ได้ จึงต้องใช้ **Distributed Transaction** ที่มี 3 แบบให้เลือก

#### 1. 2PC (Two-phase Commit)

ให้ Coordinator สั่งทุก Database ที่เกี่ยวข้อง เตรียมพร้อม และล็อคข้อมูลไว้ก่อน ถ้าทุกตัวตอบว่าพร้อมก็สั่ง Commit พร้อมกัน ถ้ามีตัวไหนไม่พร้อมก็สั่ง Abort ทั้งหมด

![[digital_wallet_2pc.png]]

ข้อเสียคือช้า เพราะล็อคข้อมูลค้างไว้นานระหว่างรอทุกตัวตอบ และ Coordinator กลายเป็น Single Point of Failure ถ้ามันล่มกลางทาง Database ทุกตัวจะค้างล็อคอยู่แบบนั้น เพราะงั้นเลยมี Concept TC/C มาให้พิจารณากัน

#### TC/C (Try-Confirm/Cancel)

แบ่งเป็น 2 Phase เหมือนกัน แต่แต่ละ Phase เป็น Transaction แยกที่จบในตัวเอง ไม่ต้องล็อคค้างข้าม Phase

| Phase               | A            | C              |
| ------------------- | ------------ | -------------- |
| Try                 | หัก 1 Dollar | ไม่ทำอะไร      |
| Confirm (ถ้าสำเร็จ) | ไม่ทำอะไร    | เพิ่ม 1 Dollar |
| Cancel (ถ้าล้มเหลว) | คืน 1 Dollar | ไม่ทำอะไร      |

Try
![[digital_wallet_tc-c_try.png]]

Confirm
![[digital_wallet_tc-c_confirm.png]]

Cancel
![[digital_wallet_tc-c_cancel.png]]

ฟังดูแปลกๆ นะ เพราะถ้าล้มเหลว ระบบก็หักเงินจาก A ไปแล้วหนิ

ใช่... และ Transaction ใน Try จบไปแล้วด้วย ย้อนกลับไม่ได้ ทางแก้คือให้ Cancel สร้าง Transaction ใหม่ที่ทำสิ่งตรงข้าม คือคืน 1 Dollar ให้ A เพื่อหักล้างกัน

อีกคำถามคือ ถ้า Wallet Service ล่มกลางทางล่ะ พอกลับมามันจะรู้ได้ยังไงว่าทำถึงไหนแล้ว คำตอบคือบันทึกความคืบหน้าของแต่ละ Phase ไว้ใน **Phase Status Table** พอกลับมาก็เปิดดูได้ว่าต้อง Confirm หรือ Cancel ต่อ

![[digital_wallet_phase_status_table.png]]

#### 3. **Saga**

เป็นอีกทางเลือกที่นิยมในสาย [[10 Microservices|Microservices]] โดยเรียง Operation ต่อกันเป็นลำดับ ทำทีละขั้น ถ้าขั้นไหนล้มเหลวก็สร้าง Transaction ที่ตรงข้ามย้อนกลับไปทีละขั้นจนถึงขั้นแรก ต่างจาก TC/C ตรงที่ต้องทำเรียงกันทีละขั้นเสมอ ทำขนานกันไม่ได้

![[digital_wallet_saga.png]]

**ปัญหาคือ** ตอนนี้รับประกันได้ว่าโอนเงินครบ
แต่ถ้า User ใส่คำสั่งผิดตั้งแต่แรก เราจะย้อนดูได้ยังไงว่าเกิดอะไรขึ้น ยิ่งระบบการเงินมักโดน Audit ด้วยคำถามอย่าง _**ยอดเงิน ณ เวลาใดเวลาหนึ่งคือเท่าไหร่**_ หรือ _**จะพิสูจน์ได้ยังไงว่ายอดปัจจุบันถูกต้อง**_

วิธีนี้จะตอบคำถามเหล่านี้ไม่ได้ เพราะเก็บแค่ยอดล่าสุด ประวัติหายไปหมดตอนอัปเดต เพราะงั้นเราจะทำ Event Sourcing เพิ่มเติมกัน

### Event Sourcing

Event Sourcing คือการแก้ปัญหาว่าจะเก็บ History ยังไงดีด้วยการเก็บ **ทุกการเปลี่ยนแปลง** ไว้เป็นประวัติที่แก้ไขไม่ได้ แทนที่จะเก็บแค่ยอดล่าสุด มีศัพท์สำคัญ 4 ตัวที่ควรรู้

- **Command**: คำสั่งที่ส่งเข้ามา เช่น "โอน 1 Dollar จาก A ไป C"
- **Event**: ผลลัพธ์ของ Command ที่ผ่านการตรวจแล้ว เช่น “A ถูกหัก 1 Dollar” และ “C ได้รับ 1 Dollar” ถือเป็นข้อเท็จจริงที่เกิดขึ้นแล้ว
- **State**: ยอดเงินของทุกบัญชีในตอนนั้น
- **State Machine**: ตัวตรวจ Command ว่าเป็น Command ที่รู้จัก และสามารถทำได้มั้ย แล้วสร้าง Event และเอา Event ไปอัปเดต State

![[digital_wallet_event_sourcing.png]]

Command และ Event ต้องเรียงตามลำดับเสมอ จึงเก็บไว้ใน Queue แบบ FIFO (มาก่อน ก็ได้ออกก่อน) เช่น Kafka นั่นเอง

**ข้อดีของ Event Sourcing คือ Reproducibility** เพราะ Event ที่ได้จะไม่สามารถถูกแก้ไขได้ และ State Machine ทำงานแบบเดิมเสมอ เราจึงสร้างยอดเงิน ณ เวลาใดก็ได้ขึ้นมาใหม่ด้วยการเล่น Event ซ้ำตั้งแต่ต้นจนถึงเวลานั้น

![[digital_wallet_reproduce_states.png]]

> [!Note] CQRS คืออะไร?
> **CQRS (Command Query Responsibility Segregation)** เป็นการแยก Write กับ Read จากกัน โดยที่...
>
> - ฝั่ง Write จะมี State Machine ตัวเดียวคอยรับ Command
> - ฝั่ง Read จะมี State Machine แบบที่อ่านอย่างเดียว แต่มีหลายตัว เพื่อให้แต่ละตัวอ่าน Event แล้วสร้างมุมมองข้อมูลของตัวเอง และให้ Client มาดึงข้อมูลจากฝั่ง Read แทนที่จะไปอ่านในจุดที่ฝั่ง Write มีการแก้ไข
>
> ![[digital_wallet_cqrs.png]]

---

## 3. เจาะลึกส่วนสำคัญๆ

ตอนนี้เราได้ Event Sourcing ที่ย้อนดูประวัติได้แล้ว แต่ยังเหลือโจทย์อีก 3 เรื่อง คือ...

1. ทำยังไงให้ระบบทำงานได้เร็วขึ้น
2. ทำยังไงให้ระบบไม่ล่ม
3. ทำยังไงให้ระบบรองรับ 1 ล้าน TPS ได้จริง

### ทำยังไงให้ระบบทำงานได้เร็วขึ้น?

คือ... การคุยกับ Kafka และ Database ผ่าน Network ทุกครั้งมันจะเสียเวลาไปหน่อย เราเลยควรจะย้าย Command, Event และ State มาเก็บเป็นไฟล์ไว้ในเครื่องเดียวกันแทน เรียกว่า File-based command and event storage อ่ะ

- **Command กับ Event** จะถูกเขียนต่อท้ายไฟล์ และใช้ **mmap** ผูกไฟล์เข้ากับ Memory ทำให้อ่านเขียนได้เร็วเกือบเท่า Memory
- **State** จะถูกเก็บใน **RocksDB** ซึ่งเป็น Key-Value Store แบบไฟล์ในเครื่อง ออกแบบมาให้เขียนเร็ว
- **Snapshot** จะบันทึก State ไว้เป็นระยะ เช่น ทุกๆ เที่ยงคืน เวลาจะดูยอดย้อนหลังก็โหลดจาก Snapshot แล้วเล่น Event ต่อจากตรงนั้น ไม่ต้องเริ่มจาก Event แรกสุด

![[digital_wallet_file-based_command_and_event_storage.png]]

### ทำยังไงให้ระบบไม่ล่ม?

ทีนี้...พอเก็บทุกอย่างไว้ในเครื่องเดียวละ ก็กลายเป็นว่า...เครื่องนั้นก็กลายเป็น Single Point of Failure แทน เราก็ต้องคิดต่อว่าข้อมูลส่วนไหนที่ต้องปกป้องบ้าง

พอไล่ๆ เช็คไปใน Command, Event, State, State Machine, Snapshot ก็มีแค่ **Event อย่างเดียว**อ่ะนะ เพราะ State กับ Snapshot สร้างใหม่ได้เสมอจากการเล่น Event ซ้ำ แต่เราไม่สามารถนำ Command มาสร้าง Event ซ้ำให้ได้ผลเหมือนเดิมไม่ได้ เพราะตอนตรวจ Command อาจจะมีปัจจัยสุ่ม หรือมีการเรียกระบบภายนอกเข้ามาเกี่ยวได้อ่ะนะ

การจะ replicate Event ไปหลายเครื่องจะใช้ **Raft** ซึ่งเป็น Algorithm ที่ทำให้ทุกเครื่องมีรายการ Event ตรงกันเป๊ะ ตราบใดที่ยังมีจำนวนเครื่องมากกว่าครึ่งยังทำงานอยู่ ไว้เป็นตัวสำรองที่เก็บ Situation ของเครื่องหลักไว้ ไว้ให้เราเอามา replay ย้อนหลังอ่ะ

![[digital_wallet_raft.png]]

คือในกลุ่มนี้จะมี **Leader** หนึ่งเครื่องคอยรับ Command แปลงเป็น Event แล้วส่งต่อให้ **Follower** ทุกเครื่องเอา Event ไปอัปเดต State ของตัวเองจนตรงกันหมด ถ้า Leader ล่ม Raft ก็จะเลือก Leader ใหม่จากเครื่องที่เหลือให้อัตโนมัติ

#### ทำยังไงให้ระบบรองรับ 1 ล้าน TPS ได้จริง?

Raft กลุ่มเดียวก็จะรับโหลดได้จำกัด จึงต้องแบ่งบัญชีออกเป็นหลาย Partition โดยแต่ละ Partition ก็คือ Raft หนึ่งกลุ่ม ทีนี้ถ้าบัญชี A กับ C อยู่คนละ Partition ก็วนกลับไปใช้ **Saga** หรือ **TC/C** จากส่วนที่ 2 มาประสานงานข้าม Partition อีกที

อีกเรื่องที่ต้องคิดคือ Client จะรู้ได้ยังไงว่าโอนสำเร็จแล้ว ถ้าให้ Client คอยถามซ้ำๆ จะช้าและเปลืองโหลด จึงเพิ่ม **Reverse Proxy** ไว้ตรงกลาง แล้วให้ฝั่ง Read ของแต่ละ Partition **Push** สถานะกลับมาทันทีที่ได้ Event ผลลัพธ์จึงไปถึง Client แทบจะทันที

![[digital_wallet_final_design.png]]

**สรุปการโอน 1 Dollar จาก A ไป C** ในระบบสุดท้ายจะเป็นแบบนี้

1. User A ส่งคำสั่งโอนเงินไปที่ Saga Coordinator ซึ่งประกอบด้วย 2 Operation คือ A-1 Dollar และ C+1 Dollar
2. Saga Coordinator สร้าง Record ใน Phase Status Table เพื่อติดตามสถานะของ Transaction นี้
3. Saga Coordinator ดูลำดับของ Operation แล้วพบว่าต้องทำ A-1 Dollar ก่อน จึงส่ง A-1 Dollar เป็น Command ไปที่ Partition 1 ซึ่งเก็บข้อมูลบัญชีของ A
4. Raft Leader ของ Partition 1 รับ Command A-1 Dollar แล้วบันทึกลง Command List จากนั้นตรวจสอบ Command ถ้าถูกต้องก็แปลงเป็น Event แล้วใช้ Raft ซิงค์ข้อมูลไปยังทุก Node ในกลุ่ม เมื่อซิงค์เสร็จจึงค่อยทำ Event นั้นจริง คือหักเงิน 1 Dollar จากบัญชี A
5. หลังจากซิงค์ Event เสร็จ Partition 1 ส่งข้อมูลต่อไปยังฝั่ง Read ตามหลัก CQRS ฝั่ง Read ก็สร้าง State และสถานะการทำงานขึ้นมาใหม่
6. ฝั่ง Read ของ Partition 1 Push สถานะกลับไปให้ Saga Coordinator ซึ่งเป็นผู้ส่งคำสั่งมา
7. Saga Coordinator ได้รับสถานะว่าสำเร็จจาก Partition 1
8. Saga Coordinator บันทึกลง Phase Status Table ว่า Operation ที่ Partition 1 สำเร็จแล้ว
9. เมื่อ Operation แรกสำเร็จ Saga Coordinator ก็ทำ Operation ที่สองต่อ คือส่ง C+1 Dollar เป็น Command ไปที่ Partition 2 ซึ่งเก็บข้อมูลบัญชีของ C
10. Raft Leader ของ Partition 2 รับ Command C+1 Dollar แล้วบันทึกลง Command List ถ้าถูกต้องก็แปลงเป็น Event แล้วใช้ Raft ซิงค์ข้อมูลไปยังทุก Node ในกลุ่ม เมื่อซิงค์เสร็จจึงค่อยทำ Event นั้นจริง คือเพิ่มเงิน 1 Dollar ให้บัญชี C
11. หลังจากซิงค์ Event เสร็จ Partition 2 ส่งข้อมูลต่อไปยังฝั่ง Read ตามหลัก CQRS ฝั่ง Read ก็สร้าง State และสถานะการทำงานขึ้นมาใหม่
12. ฝั่ง Read ของ Partition 2 Push สถานะกลับไปให้ Saga Coordinator
13. Saga Coordinator ได้รับสถานะว่าสำเร็จจาก Partition 2
14. Saga Coordinator บันทึกลง Phase Status Table ว่า Operation ที่ Partition 2 สำเร็จแล้ว
15. ตอนนี้ทุก Operation สำเร็จหมดแล้ว ถือว่า Distributed Transaction เสร็จสมบูรณ์ Saga Coordinator ตอบผลลัพธ์กลับไปให้ผู้ส่งคำสั่ง

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Digital Wallet ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
