---
title: 23 Distributed Message Queue
tags:
  - system
  - systemdesign
  - distributed_message_queue
  - message_queue
  - pub_sub
  - database_partition
  - database_replication
  - service_discovery
  - delivery_sementics
---
วันนี้เราจะมาสร้าง Distributed Message Queue กัน คือตัวที่เราหยิบมาใช้แทบทุกบทอย่าง Kafka นั่นแหละ หน้าที่หลักคือ**รับ Message จาก Producer เก็บไว้อย่างปลอดภัย แล้วส่งต่อให้ Consumer ตามลำดับ** (พื้นฐานอ่านได้ใน [[09 Message Queue|Message Queue]])

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: Message มีรูปแบบอะไร และขนาดประมาณไหนเหรอครับ? 
A1: ข้อความอย่างเดียว ขนาดหลัก KB

Q2: Message ถูกอ่านซ้ำได้มั้ยครับ? 
A2: ได้นะ Consumer หลายตัวอ่าน Message เดียวกันได้

Q3: Message ต้องถูกอ่านตามลำดับที่ส่งเข้ามามั้ยครับ? A3: ต้องตามลำดับ

Q4: ต้องเก็บ Message ลง Disk ด้วยมั้ยครับ แล้วต้องเก็บนานแค่ไหน? 
A4: ต้องเก็บ เอาเป็น 2 สัปดาห์ละกัน

Q5: ต้องรองรับ Producer กับ Consumer กี่ตัวเหรอครับ? 
A5: ยิ่งเยอะยิ่งดี

Q6: ต้องรองรับ Delivery Semantic แบบไหนครับ? At-most Once, At-least Once หรือ Exactly Once? 
A6: At-least Once ต้องมีแน่นอน แต่ถ้าได้ก็อยากให้มีทุกแบบ และให้ User ตั้งค่าเลือกได้

Q7: แล้ว Throughput กับ Latency ต้องการประมาณไหนครับ? 
A7: ต้อง Throughput สูงได้สำหรับงานอย่างรวม Log และต้อง Latency ต่ำได้สำหรับงาน Message Queue ทั่วไป

### Functional Requirement

- Producer ส่ง Message เข้า Queue ได้
- Consumer ดึง Message จาก Queue ได้
- Message ถูกอ่านซ้ำได้ หรืออ่านครั้งเดียวก็ได้
- ข้อมูลเก่าถูกลบทิ้งได้ตาม Retention (ถึงกำหนดก็ลบได้เลย)
- Message มีขนาดระดับ KB
- ส่ง Message ให้ Consumer ตามลำดับที่ส่งเข้ามา
- User ตั้งค่า Delivery Semantic ได้เอง (At-least Once, At-most Once, Exactly Once)

### Non-Functional Requirement

- **High Throughput หรือ Low Latency**: ตั้งค่าได้ตาม Use Case
- **Scalable**: เป็น Distributed System รับ Message ที่พุ่งขึ้นกะทันหันได้
- **Persistent and Durable**: เก็บข้อมูลลง Disk และ Replicate ไว้หลาย Node

---

## 2. Sketch ภาพระบบคร่าวๆ

### Basic Knowledge 101

ก่อนออกแบบ มีศัพท์ที่ต้องรู้ก่อน
- **Producer / Consumer**: Producer ส่ง Message เข้า Queue ส่วน Consumer Subscribe แล้วดึง Message ไปใช้ ทั้งคู่เป็น Client ส่วน Message Queue เป็น Server![[dmq_key_components.png]]

- **Messaging Model**: มี 2 แบบ
    - **Point-to-Point**: Message 1 ชิ้นถูกอ่านโดย Consumer **แค่ตัวเดียว** อ่านแล้วก็ลบทิ้งเลย 
    - **Publish-Subscribe**: Message ถูกส่งให้ **Consumer ทุกตัวที่ Subscribe Topic นั้น** 
- **Topic**: หมวดหมู่ของ Message มีชื่อไม่ซ้ำกันทั้งระบบ โดยเมื่อ Producer ส่งเข้า Topic ไหน Consumer ก็อ่านจาก Topic นั้น
- **Partition**: ถ้า Topic หนึ่งมีข้อมูลเยอะเกินกว่า Server เครื่องเดียวจะรับไหว ก็แบ่ง Topic ออกเป็นหลาย Partition (หรือก็คือ Sharding) แล้วกระจายไปอยู่หลาย Server
    - แต่ละ Partition ทำงานแบบ **FIFO** ทำให้ Message ใน Partition เดียวกันเรียงลำดับได้ตลอด
    - อยากให้ Topic รับข้อมูลได้มากขึ้น ก็แค่เพิ่มจำนวน Partition![[dmq_partition.png]]

- **Offset**: ตำแหน่งของ Message ใน Partition
- **Message Key**: เช่น `user_id` Message ที่มี Key เดียวกันจะไปอยู่ Partition เดียวกันเสมอ ถ้าไม่มี Key ก็สุ่ม Partition ให้
- **Broker**: Server ที่เก็บ Partition

### Consumer Group

**Consumer Group** เป็นกลุ่มของ Consumer ที่ช่วยกันอ่าน Message จาก Topic แต่ละ Group จดตำแหน่งที่อ่านถึง (Offset) แยกของตัวเอง เช่น Group หนึ่งไว้ทำ Billing อีก Group ไว้ทำ Accounting

![[dmq_consumer_groups.png]]

เหมือนในภาพนี้...
- Consumer Group 1 Subscribe Topic A
- Consumer Group 2 Subscribe ทั้ง Topic A และ Topic B
- Topic A ถูก Subscribe ทั้ง 2 Group แปลว่า Message เดียวกันถูกอ่านหลายครั้ง ซึ่งก็คือ **Publish-Subscribe**

แต่ถ้า Consumer 2 ตัวใน Group เดียวกันอ่าน Partition เดียวกันพร้อมกัน ก็การันตีลำดับไม่ได้ เพราะไม่รู้ว่าใครจะประมวลผลเสร็จก่อน

เราเลยมีกฎว่า **Partition หนึ่งถูกอ่านได้โดย Consumer แค่ตัวเดียวใน Group เดียวกัน** ซึ่งทำให้...
- ถ้าเอา Consumer ทุกตัวไว้ใน Group เดียว Message แต่ละชิ้นก็ถูกอ่านโดยตัวเดียว ซึ่งก็คือ **Point-to-Point**
- ถ้า Consumer ใน Group มีมากกว่าจำนวน Partition ตัวที่เกินจะว่างงาน เช่น ในภาพ Consumer 3 อ่าน Topic B ไม่ได้ เพราะ Consumer 4 อ่านไปแล้ว เพราะงั้นควรจอง Partition ไว้เยอะๆ ตั้งแต่แรก พอต้อง Scale ก็แค่เพิ่ม Consumer

### High-Level Design

![[dmq_high-level_design.png]]

จากภาพ ระบบนี้จะถูกแบ่งเป็น 3 ส่วน ก็คือ...
1. Clients
2. Core Service and Storage
3. Coordination Service

#### Clients
- **Producer**: ส่ง Message เข้า Topic
- **Consumer Group**: Subscribe Topic แล้วอ่าน Message

#### Core Service and Storage
- **Broker**: เก็บ Partition โดยแต่ละ Partition เก็บ Message บางส่วนของ Topic
- **Data Storage**: เก็บ Message ใน Partition
- **State Storage**: เก็บสถานะของ Consumer เช่น อ่านถึง Offset ไหนแล้ว
- **Metadata Storage**: เก็บ Config ของ Topic เช่น จำนวน Partition, Retention

#### Coordination Service
- **Service Discovery**: รู้ว่า Broker ตัวไหนยังทำงานอยู่
- **Leader Election**: เลือก Broker ตัวหนึ่งเป็น **Active Controller** (มีได้ตัวเดียวทั้ง Cluster) มีหน้าที่แบ่ง Partition ให้ Broker ต่างๆ นิยมใช้ Zookeeper หรือ etcd

### Message Structure

โครงสร้างของ Message คือ contract ระหว่าง Producer, Queue และ Consumer ถ้าทุกฝ่ายใช้โครงสร้างเดียวกัน ก็ส่งต่อกันได้เลยโดยไม่ต้อง Copy และแปลงข้อมูล

| Field     | Description                                              | Type    |
| --------- | -------------------------------------------------------- | ------- |
| key       | ใช้เลือก Partition ด้วย `hash(key) % numPartitions`      | byte[]  |
| value     | เนื้อหาของ Message                                       | byte[]  |
| topic     | Topic ที่ Message นี้อยู่                                | string  |
| partition | Partition ที่ Message นี้อยู่                            | integer |
| offset    | ตำแหน่งใน Partition ที่ Message นี้อยู่                  | long    |
| timestamp | เวลาที่ Message ถูกเก็บ                                  | long    |
| size      | ขนาดของ Message                                          | integer |
| crc       | Cyclic Redundancy Check สำหรับการเช็คว่าข้อมูลไม่เสียหาย | integer |

>[!Note] Key not Key in Key-Value Store
>คือ Key ของ Message นี้ไม่ใช่ Key ใน Key-Value Store Key ของ Message ซ้ำกันได้ ไม่ใส่ก็ได้ และไม่ได้เอาไว้หา Message แต่เอาไว้เลือก Partition ซึ่งมักเป็นข้อมูลทางธุรกิจ เช่น user_id ส่วนเลข Partition เป็นเรื่องภายใน Queue ไม่ควรให้ Client เห็น

---

## 3. เจาะลึกจุดสำคัญๆ

เพื่อให้ได้ Throughput สูงและสามารถเก็บข้อมูลได้นานๆ หลักการที่ใช้ตลอดทั้ง Design มี 3 ข้อ
1. **อ่านเขียน Disk แบบเรียงกัน**: ใช้ประโยชน์จากความเร็วของ Sequential Access และ Disk Cache ของ OS
2. **ใช้โครงสร้าง Message เดียวตลอดทาง**: Message ไม่ต้องถูกแปลงระหว่าง Producer → Queue → Consumer เลยไม่ต้อง Copy ข้อมูล ซึ่งแพงมากในระบบที่ Traffic สูง
3. **Batch ทุกที่ที่ทำได้**: Producer ส่งเป็น Batch, Broker เขียนเป็น Batch, Consumer ดึงเป็น Batch

### 1. Data Storage

Pattern การใช้งานของ Message Queue คือ
- Write หนัก และ Read หนัก
- ไม่มี Update หรือ Delete
- อ่านเขียนเรียงกันเกือบทั้งหมด

ซึ่งจะมีให้เลือก 2 ทาง
- **Database**: สร้างตาราง Topic แล้วเขียน Message เป็นแถว แต่ Database ที่รับทั้ง Write หนักและ Read หนักในระดับนี้ได้ ออกแบบยากมาก และจะกลายเป็นคอขวดของระบบ
- **Write-Ahead Log (WAL)**: ไฟล์ธรรมดาที่เขียนต่อท้ายไปเรื่อยๆ (Append-only) แบบเดียวกับ Redo Log ของ MySQL อ่านเขียนเรียงกันล้วนๆ ซึ่ง Disk ทำได้ดีมาก และ Disk แบบจานหมุนก็จุเยอะราคาถูก

เพราะงั้นเราเลือก **WAL**
![[dmq_append_message.png]]

Message ใหม่จะถูกเขียนต่อท้าย Partition พร้อม Offset ที่เพิ่มขึ้นเรื่อยๆ แต่ไฟล์จะใหญ่ไปเรื่อยๆ ไม่ได้ เลยแบ่งเป็น **Segment**
- Message ใหม่เขียนลง **Active Segment** เท่านั้น
- พอ Active Segment ใหญ่ถึงขนาดที่กำหนด ก็สร้างตัวใหม่ แล้วตัวเก่ากลายเป็น Inactive
- Inactive Segment ใช้อ่านอย่างเดียว และถูกลบได้เมื่อเกิน Retention หรือเกินความจุ

Segment ของ Partition เดียวกันจะอยู่ในโฟลเดอร์ `Partition-{:partition_id}`
![[dmq_data_segment.png]]

> [!Note] Disk ช้าจริงเหรอ? 
> Disk แบบจานหมุนช้าแค่ตอน **Random Access** ถ้าอ่านเขียนเรียงกัน Disk หลายลูกที่ต่อแบบ RAID ทำได้หลายร้อย MB ต่อวินาทีสบายๆ
> 
> และ OS สมัยใหม่ยังใช้ Memory ที่ว่างมา Cache ข้อมูลจาก Disk ซึ่ง WAL ก็ได้ประโยชน์จากตรงนี้ด้วย

### 2. Producer Flow

Producer จะส่ง Message ไป Partition หนึ่ง แล้วต้องต่อกับ Broker ตัวไหน?

#### a. Routing Layer
![[dmq_routing_layer.png]]

1. Producer ส่ง Message ไปที่ Routing Layer
2. Routing Layer อ่าน **Replica Distribution Plan** (แผนว่า Replica ของแต่ละ Partition อยู่ Broker ไหน) จาก Metadata Storage มา Cache ไว้ แล้วส่ง Message ไปที่ Leader Replica ของ Partition 1 บน Broker 1
3. Follower Replica มาดึงข้อมูลจาก Leader
4. พอ Replica sync "พอ" แล้ว Leader ก็ Commit ทำให้ Consumer อ่านได้ แล้วตอบกลับ Producer

แต่วิธีนี้ Routing Layer เพิ่ม Network Hop อีกทอด และไม่ได้ทำ Batching เลย ก็เลยมีอีกวิธี...
#### b. ย้าย Routing และ Buffer เข้าไปใน Producer
![[dmq_producer_w_buffer_and_routing.png]]

ใส่ Routing และ Buffer ไว้ใน Client Library ของ Producer เลย ข้อดีคือ...
- **Network Hop น้อยลง**: Latency ก็ต่ำลง
- **Producer เลือก Partition เองได้**
- **Buffer รวม Message ไว้ใน Memory**: แล้วส่งเป็น Batch ใหญ่ทีเดียว Throughput ก็สูงขึ้น

ส่วนขนาด Batch ก็เป็น Trade-off คลาสสิก
- **Batch ใหญ่**: Throughput สูง แต่ Latency สูง เพราะต้องรอรวม Message
- **Batch เล็ก**: Latency ต่ำ แต่ Throughput ลดลง (ถ้าอยากได้ทั้งคู่ อาจต้องเพิ่ม Partition ต่อ Topic เพื่อชดเชย)

### 3. Consumer Flow

Consumer บอกว่าจะอ่าน Partition ไหน ตั้งแต่ Offset ไหน แล้วได้ Message กลับมาเป็นก้อน เช่น ในภาพ Consumer 1 อ่านถึง Offset 6 ส่วน Consumer 2 อ่านถึง Offset 13
![[dmq_consumer_flow.png]]

#### Push vs Pull

- **Push**: Broker ส่ง Message ไปให้ทันทีที่ได้รับ ทำให้ **Latency ต่ำ** แต่ถ้า Consumer ประมวลผลช้ากว่าที่ Producer ส่งเข้ามา Consumer ก็จะรับไม่ไหว
- **Pull**: Consumer ดึงเอง ทำให้ **Consumer คุมความเร็วเองได้** ถ้าช้าก็แค่เพิ่ม Consumer หรือค่อยๆ ไล่ตามทีหลัง และ**เหมาะกับ Batch** เพราะดึง Message ทั้งหมดหลังตำแหน่งปัจจุบันมาได้ทีเดียว ส่วนข้อเสียคือถ้าไม่มี Message ใหม่ก็ยัง Pull อยู่ แก้ได้ด้วย **Long Polling** (ให้ Request ค้างรอ Message ใหม่ได้สักพัก)

เพราะงั้น Message Queue ส่วนใหญ่จึงเลือก **Pull**
![[dmq_pull_model.png]]

1. Consumer ใหม่อยากเข้า Group 1 และ Subscribe Topic A ก็เอาชื่อ Group ไป Hash เพื่อหา Broker ที่ดูแล Group นี้ ทำให้ Consumer ทุกตัวใน Group เดียวกันต่อกับ Broker ตัวเดียวกัน ซึ่งเรียกว่า **Coordinator**
2. Coordinator ยืนยันว่า Consumer เข้า Group แล้ว และแบ่ง Partition 2 ให้ (เช่น แบ่งแบบ Round-robin หรือ Range)
3. Consumer ดึง Message ต่อจาก Offset ล่าสุดที่เก็บอยู่ใน State Storage
4. Consumer ประมวลผล แล้ว Commit Offset กลับไปที่ Broker (ลำดับของการประมวลผลกับการ Commit จะกำหนด Delivery Semantic ซึ่งจะพูดถึงทีหลัง)

> [!Note] Coordinator ไม่ใช่ Coordination Service 
> **Coordinator** คือ Broker ที่ดูแล Consumer Group หนึ่งๆ ส่วน **Coordination Service** (Zookeeper) ดูแล Broker ทั้ง Cluster

#### Consumer Rebalancing

**Consumer Rebalancing** คือการแบ่งใหม่ว่า Consumer ตัวไหนดูแล Partition ไหน เกิดขึ้นเมื่อ Consumer เข้า Group ใหม่, ออกจาก Group, ล่ม หรือจำนวน Partition เปลี่ยน -> ที่ต้องมี Rebalancing เพื่อให้ Partition มี Consumer ที่สมดุลกัน ไม่หนักที่ Partition ใดจนเกินไป

โดย Coordinator มีหน้าที่...
- รับ Heartbeat จาก Consumer และจดรายชื่อ Consumer ใน Group
- พอรายชื่อเปลี่ยน ก็เลือก **Leader ของ Group** ใหม่
- Leader สร้าง **Partition Dispatch Plan** (แผนแบ่ง Partition) ส่งให้ Coordinator แล้ว Coordinator กระจายให้ Consumer ตัวอื่นๆ

![[dmq_consumer_rebalance'.png]]

ลองดูตอนมี Consumer ใหม่เข้ามา สมมติ Topic มี 4 Partition
1. ตอนแรกมีแค่ Consumer A ดูแลทุก Partition และส่ง Heartbeat ให้ Coordinator อยู่
2. Consumer B ขอเข้า Group
3. Coordinator รู้ว่าต้อง Rebalance แต่จะไม่แจ้งไปเอง ต้องรอ Heartbeat จาก A ก่อน แล้วตอบกลับไปว่าให้ Rejoin
4. พอทุกตัว Rejoin ครบ Coordinator ก็เลือก Leader แล้วแจ้งผลให้ทุกตัวรู้
5. Leader สร้าง Partition Dispatch Plan ส่งให้ Coordinator ส่วน Follower ก็ถาม Coordinator ว่าตัวเองได้ Partition ไหน
6. Consumer เริ่มอ่าน Partition ที่ได้ใหม่ เช่น A ได้ Partition 1, 3 ส่วน B ได้ Partition 2, 4

ส่วนกรณีอื่นก็จะต่างกันแค่จุดเริ่มต้น หลังจากนั้นทำเหมือนขั้นที่ 3. – 6.
- **Consumer ขอออกจาก Group**: A แจ้งออกเอง Coordinator ก็ให้ B Rejoin สุดท้าย B ได้ทั้ง 4 Partition 
- **Consumer ล่ม**: A ไม่ส่ง Heartbeat มาเกินเวลาที่กำหนด Coordinator ก็ถือว่า A ตายแล้ว และเริ่ม Rebalance 

### 4. State Storage, Metadata Storage และ Zookeeper

**State Storage** มีหน้าที่ในการเก็บ 2 อย่าง
1. Partition ไหนถูกอ่านโดย Consumer ตัวไหน
2. Offset ล่าสุดที่แต่ละ Consumer Group อ่านถึงในแต่ละ Partition

![[dmq_last_consumed_offset.png]]

จากภาพ Consumer Group 1 Commit Offset 6 ไว้ ถ้า Consumer ตัวนี้ล่ม ตัวใหม่ใน Group เดียวกันก็อ่าน Offset ล่าสุดจาก State Storage แล้วอ่านต่อจาก Offset 7 ได้เลย

ข้อมูลนี้อ่านเขียนบ่อยแต่ปริมาณไม่เยอะ อัปเดตบ่อยแต่แทบไม่ลบ อ่านเขียนแบบสุ่ม และต้อง Consistent

**Metadata Storage** เก็บ Config ของ Topic เช่น จำนวน Partition, Retention, การกระจาย Replica ซึ่งเปลี่ยนไม่บ่อย ปริมาณน้อย แต่ต้อง Consistent สูง

ทั้ง 2 อย่างเลยเหมาะกับ **Zookeeper** ซึ่งเป็น Key-Value Store แบบมีลำดับชั้น นิยมใช้ทำ Distributed Configuration, Synchronization และ Naming Registry
![[dmq_w_zookeeper.png]]

พอใช้ Zookeeper Design ก็ง่ายขึ้น โดย...
- ย้าย State Storage และ Metadata Storage ไปไว้ใน Zookeeper
- Broker เหลือแค่เก็บ Message จากเดิมที่มีทั้ง Data และ State
- Zookeeper ช่วยเลือก Leader ของ Broker Cluster ด้วย

### 5. Replication

Hardware เสียเป็นเรื่องปกติ ถ้า Disk พัง ข้อมูลก็หาย ทางแก้ก็คือ **Replication** (พื้นฐานอ่านได้ใน [[05 Database Replication|Database Replication]])
![[dmq_replication.png]]

- แต่ละ Partition มี 3 Replica อยู่คนละ Broker โดยมี **Leader** 1 ตัว ที่เหลือเป็น **Follower**
- Producer ส่ง Message ไปที่ Leader เท่านั้น แล้ว Follower คอยดึง Message ใหม่จาก Leader
- พอ Message ซิงค์ไปที่ Replica "พอ" แล้ว Leader ถึงจะตอบ Ack กลับไปให้ Producer

การกระจาย Replica แบบนี้เรียกว่า **Replica Distribution Plan** สร้างโดย Active Controller แล้วเก็บไว้ใน Metadata Storage เช่น ในภาพS
```txt
Topic A Partition 1: Broker 1 (Leader), 2, 3
Topic A Partition 2: Broker 2 (Leader), 3, 4
Topic B Partition 1: Broker 3 (Leader), 4, 1
```

#### In-Sync Replicas (ISR)

**ISR** คือ Replica ที่ "ตามทัน" Leader ซึ่งนิยามว่าตามทันขึ้นกับ Config เช่น `replica.lag.max.messages = 4` แปลว่า Follower ตามหลัง Leader ได้ไม่เกิน 3 Message ถึงจะยังอยู่ใน ISR (Leader อยู่ใน ISR เสมอ)
![[dmq_in-sync_replica.png]]

- **Committed Offset** ของ Leader คือ 13 แปลว่า Message ถึง 13 ซิงค์ไปที่ทุก Replica ใน ISR แล้ว ส่วน 14 กับ 15 ยังไม่ Commit
- Replica 2 และ 3 ตามทัน เลยอยู่ใน ISR
- Replica 4 ตามไม่ทัน เลยไม่อยู่ใน ISR พอตามทันเมื่อไหร่ก็กลับเข้า ISR ได้

> [!Note] ทำไมต้องมี ISR? 
> ถ้าอยากชัวร์ว่า Message ไม่หาย ก็ต้องรอทุก Replica ซิงค์ก่อนตอบ Ack แต่ถ้ามี Replica ช้าตัวเดียว ทั้ง Partition ก็ช้าตาม ISR เลยช่วยให้**รอแค่ตัวที่ตามทัน** ซึ่งเป็นจุดสมดุลระหว่าง Performance กับ Durability

#### ACK Setting

Producer เลือกได้ว่าจะรอ Ack แบบไหน
- **ACK=all**: ได้ Ack เมื่อ**ทุก Replica ใน ISR** ได้รับ Message แล้ว -> ช้าที่สุด แต่ **Durable ที่สุด** ![[dmq_ack_all.png]]

- **ACK=1**: ได้ Ack เมื่อ **Leader** เขียนแล้ว เร็วขึ้น แต่ถ้า Leader ล่มก่อน Follower ที่ sync Message นั้นอยู่ก็จะหาย เหมาะกับงานที่ต้องการ Latency ต่ำ และยอมให้หายได้บ้าง ![[dmq_ack_1.png]]

- **ACK=0**: ส่งไปเรื่อยๆ ไม่รอ Ack และไม่ Retry **Latency ต่ำที่สุด** แต่อาจหาย เหมาะกับงานอย่างเก็บ Metrics หรือ Log ![[dmq_ack_0.png]]

> [!Note] ทำไม Consumer อ่านจาก Leader อย่างเดียว?
> - ออกแบบและดูแลง่าย
> - Partition หนึ่งถูกอ่านโดย Consumer ตัวเดียวใน Group อยู่แล้ว Connection ที่เข้า Leader เลยไม่เยอะ
> - ถ้า Topic ฮิตมากจริงๆ ก็แค่เพิ่ม Partition และ Consumer
> 
> ยกเว้นถ้า Consumer อยู่คนละ Data Center กับ Leader ก็อาจให้อ่านจาก ISR ที่ใกล้ที่สุดแทน

### 6. Scalability

ระบบจะมี 4 ส่วนที่ต้อง Scale ได้ ก็คือ...
1. Producer
2. Consumer
3. Broker
4. Partition

#### Producer และ Consumer
- **Producer**: ง่ายสุด เพราะไม่ต้องประสานงานกับใคร เพิ่มลดได้ตามสบาย
- **Consumer**: แต่ละ Group แยกกันอิสระ ส่วนภายใน Group ก็มี Rebalancing คอยจัดการตอนเพิ่ม ลด หรือล่ม

#### Broker
ก่อนจะ Scale มาดูก่อนว่าถ้า Broker ล่มจะกู้คืนยังไง โดยใช้ Replica Distribution Plan เดียวกับในหัวข้อ Replication
![[dmq_broker_crash.png]]

ถ้า Broker 3 ล่ม Replica บน Broker 3 ก็หายไปหมด Active Controller ก็จะสร้างแผนใหม่ให้ Broker ที่เหลือ โดย Replica ใหม่เป็น Follower และค่อยๆ ไล่ตาม Leader
```txt
Topic A Partition 1: Broker 1 (Leader), 2, 4 (ใหม่)
Topic A Partition 2: Broker 2 (Leader), 4, 1 (ใหม่)
Topic B Partition 1: Broker 4 (Leader ใหม่), 1, 2 (ใหม่)
```

เรื่องที่ต้องคิดเพิ่ม
- **จำนวน ISR ขั้นต่ำ**: ต้องได้รับ Message กี่ Replica ถึงจะถือว่าสำเร็จ ยิ่งเยอะยิ่งปลอดภัย แต่ก็ยิ่งช้า
- **Replica ห้ามอยู่ Broker เดียวกัน**: ไม่งั้น Broker นั้นล่มทีเดียวก็หายหมด
- **ถ้า Replica ของ Partition ล่มหมดทุกตัว ข้อมูลหายถาวร**: กระจายข้าม Data Center ปลอดภัยกว่า แต่ Latency และค่าใช้จ่ายก็สูงขึ้น

ส่วนการเพิ่ม Broker แทนที่จะย้าย Replica ทันที ให้ Controller **ยอมให้มี Replica เกินจำนวนที่ตั้งไว้ชั่วคราว**
![[dmq_broker_add.png]]

1. ตอนแรกมี 3 Broker, 2 Partition, Partition ละ 3 Replica
2. เพิ่ม Broker 4 เข้ามา Controller เปลี่ยนแผนของ Partition 2 เป็น Broker (2, 3, 4) Replica ใหม่บน Broker 4 เริ่มดึงข้อมูลจาก Leader ตอนนี้ Partition 2 มี Replica เกินชั่วคราว
3. พอ Replica บน Broker 4 ตามทันแล้ว ค่อยลบ Replica ที่เกินบน Broker 1 ออก

ทำแบบนี้ข้อมูลไม่หายระหว่างเพิ่ม Broker และการลด Broker ก็ใช้หลักเดียวกัน

#### Partition
หลักๆ ก็คือ scale partition ผ่านการเพิ่ม/ลดจำนวนได้ตามปกติ แต่ว่าก็มีเรื่องที่ต้อง concern ก็คือ...เวลาจำนวน Partition เปลี่ยน Producer จะรู้ตอนคุยกับ Broker ตัวไหนก็ได้ ส่วน Consumer ก็จะเกิด Rebalancing เพราะงั้นที่ต้องคิดจริงๆ คือฝั่งข้อมูล
- **เพิ่ม Partition**: ง่ายมาก Message เก่ายังอยู่ Partition เดิม ไม่ต้องย้าย ส่วน Message ใหม่ก็กระจายไปทุก Partition ![[dmq_partition_increase.png]]

- **ลด Partition**: Partition ที่ถูกปลดจะไม่รับ Message ใหม่ แต่ยังลบทันทีไม่ได้ เพราะอาจมี Consumer ที่กำลังอ่าน Message ใน Partition นั้นอยู่ ต้องรอจนเลย Retention แล้ว Consumer Group ค่อย Rebalance ใหม่ เพราะงั้นการลด Partition **ไม่ช่วยคืนพื้นที่ Disk ทันที** ![[dmq_partition_decrease.png]]

### 7. Data Delivery Semantics

Data Delivery Semantics มีให้เลือก 3 แบบ ก็คือ...
1. At-most Once
2. At-least Once
3. Exactly Once

#### At-most Once
Message ถูกส่ง**ไม่เกิน 1 ครั้ง** อาจหายได้ แต่ไม่ซ้ำแน่นอน
![[dmq_at_most_once.png]]

- **Producer**: ส่งแบบไม่รอ Ack (`ACK=0`) ส่งไม่สำเร็จก็ไม่ Retry
- **Consumer**: **Commit Offset ก่อนประมวลผล** ถ้าล่มหลัง Commit ก็จะไม่ได้อ่าน Message นั้นอีกเลย

เหมาะกับงานอย่าง Monitoring Metrics ที่ข้อมูลหายบ้างไม่เป็นไร

#### At-least Once
Message ถูกส่ง**อย่างน้อย 1 ครั้ง** ไม่หายแน่นอน แต่อาจซ้ำ
![[dmq_at_least_once.png]]

- **Producer**: ส่งแบบรอ Ack (`ACK=1` หรือ `ACK=all`) ถ้าไม่สำเร็จหรือ Timeout ก็ Retry
- **Consumer**: **Commit Offset หลังประมวลผลสำเร็จ** แต่ถ้าประมวลผลเสร็จแล้วล่มก่อน Commit พอกลับมาก็จะอ่าน Message นั้นซ้ำ

เหมาะกับงานที่ซ้ำได้ หรือ Consumer กรองตัวซ้ำได้เอง เช่น ใส่ Unique Key ในทุก Message แล้วให้ Database ปฏิเสธตอนเขียนซ้ำ

#### Exactly Once
Message ถูกส่ง**ครั้งเดียวเป๊ะๆ** ไม่หายไม่ซ้ำ ทำยากที่สุด และแลกกับ Performance และความซับซ้อน
![[dmq_exactly_once.png]]

เหมาะกับงานการเงิน เช่น Payment, Trading, Accounting ที่ซ้ำไม่ได้เลย และปลายทางไม่รองรับ Idempotency (ตัวอย่างการทำอ่านได้ใน [[15 Payment System|Payment System]]) ซึ่งใน Kafka ทำได้ด้วย **Idempotent Producer** (Broker ตัดตัวซ้ำจาก Retry ด้วย Producer ID + Sequence Number) คู่กับ **Transaction** (เขียน Output และ Commit Offset ใน Transaction เดียวกัน)

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Distributed Message Queue ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---