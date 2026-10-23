---
title: 22 Stock Exchange
tags:
  - system
  - systemdesign
  - raft
  - stock_exchange
  - event_sourcing
  - data_structure
  - ring_buffer
  - pub_sub
  - stateful
---
วันนี้เราจะมาสร้างตลาดหลักทรัพย์แบบอิเล็กทรอนิกส์กัน คือ... หน้าที่หลักของตลาดไม่เคยเปลี่ยนเลยตั้งแต่ยุคแลกของกัน คือ**จับคู่คนซื้อ กับคนขายให้เร็ว และถูกต้อง** 

แต่สมัยนี้ทำผ่าน Computer แทนการตะโกนกันในห้องแล้ว แต่ว่าการใช้งานก็หนักหน่วงเหมือนกัน เช่น NYSE มีการจับคู่กันหลายพันล้านครั้งต่อวัน ส่วน HKEX มีการซื้อขายกันราว 2 แสนล้านหุ้นต่อวัน เพราะงั้นระบบแบบนี้ต้องทำงานได้อย่างรวดเร็ว ทนทาน และห้ามผิดพลาดเลย จะเป็นไง มาดูกัน...

![[stockexchange_overview.png]]

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: เราจะซื้อขายอะไรบ้างเหรอครับ? หุ้น, Options หรือ Futures? 
A1: หุ้นอย่างเดียวละกัน

Q2: ต้องรองรับคำสั่งแบบไหนบ้างครับ? ส่งคำสั่งใหม่, ยกเลิก, แก้ไขคำสั่ง? แล้วประเภทคำสั่งต้องมี Limit Order, Market Order หรือ Conditional Order? 
A2: เอาแค่ส่งคำสั่งใหม่กับยกเลิกคำสั่ง ส่วนประเภทคำสั่งเอาแค่ Limit Order

Q3: ต้องรองรับการซื้อขายนอกเวลาทำการมั้ยครับ? 
A3: ไม่ต้อง เอาแค่เวลาทำการปกติ

Q4: ช่วยเล่าการทำงานหลักๆ ของตลาด และขนาดของระบบหน่อยได้มั้ยครับ เช่น มี User, จำนวนหุ้น และจำนวนคำสั่งเท่าไหร่? 
A4: Client ส่ง Limit Order ใหม่หรือยกเลิกได้ และได้รับผลการจับคู่แบบ Real-time ดู Order Book (รายการคำสั่งซื้อขาย) แบบ Real-time ได้ ต้องรองรับ User อย่างน้อยหลายหมื่นคนพร้อมกัน หุ้นอย่างน้อย 100 ตัว และคำสั่งหลายพันล้านคำสั่งต่อวัน นอกจากนี้ตลาดเป็นธุรกิจที่ถูกกำกับดูแล เลยต้องมี Risk Check ด้วย

Q5: Risk Check ที่ว่านี่แบบไหนเหรอครับ? 
A5: ง่ายๆ ละกัน เช่น User หนึ่งคนซื้อขายหุ้น Apple ได้ไม่เกิน 1 ล้านหุ้นต่อวัน

Q6: แล้วเรื่อง Wallet ของ User ต้องคิดด้วยมั้ยครับ? 
A6: ต้องเช็คว่า User มีเงินพอตอนส่งคำสั่ง และถ้าคำสั่งยังค้างอยู่ใน Order Book ก็ต้องกันเงินส่วนนั้นไว้ ไม่ให้เอาไปใช้ซ้ำ

### Functional Requirement

- ส่ง Limit Order ใหม่ได้
- ยกเลิก Order ได้
- ได้รับผลการจับคู่แบบ Real-time
- ดู Order Book แบบ Real-time ได้
- Risk Check ก่อนรับคำสั่ง
- เช็คเงินใน Wallet และกันเงินไว้สำหรับคำสั่งที่ยังค้างอยู่

### Non-Functional Requirement

- **Availability** - อย่างน้อย 99.99% แค่ล่มไม่กี่วินาทีก็เสียชื่อเสียงแล้ว
- **Fault Tolerance** - ต้องกู้คืนได้เร็ว เพื่อจำกัดความเสียหายเวลามีปัญหา
- **Latency** - Round-trip ต้องอยู่ระดับ Millisecond โดยเฉพาะ **Latency ที่ Percentile 99** ซึ่งวัดตั้งแต่ Order เข้าตลาดจนได้ผลการจับคู่กลับไป ถ้า P99 สูงตลอด User ส่วนหนึ่งจะได้ประสบการณ์ที่แย่มาก
- **Security** - ต้องมีระบบจัดการบัญชี ทำ **KYC (Know Your Client)** ยืนยันตัวตนก่อนเปิดบัญชีตามกฎหมาย และหน้าเว็บสาธารณะที่แสดงข้อมูลตลาดต้องป้องกัน DDoS ได้

---

## 2. Sketch ภาพระบบคร่าวๆ

### Business Knowledge 101

ก่อนออกแบบ มีศัพท์ทางธุรกิจที่ต้องรู้ก่อน
- **Broker** - User ทั่วไปส่วนใหญ่ซื้อขายผ่าน Broker เช่น Robinhood, Charles Schwab ที่มีหน้าจอให้ส่งคำสั่งและดูข้อมูลตลาดได้ง่ายๆ
- **Institutional Client** - ลูกค้าสถาบันที่ซื้อขายปริมาณมากผ่าน Software เฉพาะทาง เช่น กองทุนบำนาญที่ซื้อขายนานๆ ที แต่ครั้งละเยอะมาก ต้องมี Feature แบ่งคำสั่งเพื่อไม่ให้ราคาตลาดกระชาก หรือ Hedge Fund ที่ทำ Market Making ต้องการ Latency ต่ำมาก ดูข้อมูลผ่านเว็บหรือแอปแบบ User ทั่วไปไม่ได้
- **Limit Order** - คำสั่งซื้อขายที่กำหนดราคาไว้ อาจยังไม่ได้จับคู่ทันที หรือจับคู่ได้แค่บางส่วน
- **Market Order** - คำสั่งที่ไม่กำหนดราคา ซื้อขายที่ราคาตลาดตอนนั้นทันที ยอมเสียเรื่องราคาเพื่อให้ได้ซื้อขายแน่นอน เหมาะตอนตลาดเคลื่อนไหวเร็ว
- **Market Data Level**: ข้อมูลราคาของตลาดหุ้น แบ่งเป็น 3 ระดับ
	- **L1 (Level 1)** - มีแค่ราคาเสนอซื้อสูงสุด (Best Bid) กับราคาเสนอขายต่ำสุด (Best Ask) พร้อมจำนวน โดยจะมี **Bid Price**: ราคาสูงสุดที่คนซื้อยอมจ่าย และ **Ask Price**: ราคาต่ำสุดที่คนขายยอมขาย
	![[stockexchange_l1_data.png]]

	- **L2** - เห็นหลายระดับราคามากกว่า L1
	![[stockexchange_l2_data.png]]

	- **L3** - เห็นระดับราคา และจำนวนที่ต่อคิวอยู่ในแต่ละระดับราคาด้วย
	![[stockexchange_l3_data.png]]

- **Candlestick Chart**: แท่งเทียนแสดงราคาหุ้นในช่วงเวลาหนึ่ง บอกราคาเปิด (Open), ปิด (Close), สูงสุด (High) และต่ำสุด (Low) ช่วงเวลาที่นิยมคือ 1 นาที, 5 นาที, 1 ชั่วโมง, 1 วัน, 1 สัปดาห์, 1 เดือน
	![[stockexchange_candlestick.png]]

- **FIX (Financial Information eXchange)** เป็น Protocol กลางสำหรับแลกเปลี่ยนข้อมูลการซื้อขายหลักทรัพย์ สร้างขึ้นตั้งแต่ปี 1991 หน้าตาประมาณนี้
```txt
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### High-Level Design

![[stockexchange_high-level_design.png]]

ระบบจะมี 3 Flow ที่มีความต้องการด้าน Latency ต่างกัน ก็คือ...
1. Trading Flow
2. Market Data Flow
3. Report Flow
เรามาดูทีละ Flow กัน

#### Trading Flow (Critical Path)

![[stockexchange_high-level_design_trading_flow.png]]

Flow นี้จะต้องทำงานได้อย่างรวดเร็ว
1. Client ส่ง Order ผ่านเว็บหรือแอปของ Broker
2. Broker ส่ง Order เข้าตลาด
3. Order เข้าตลาดผ่าน **Client Gateway** ซึ่งตรวจเบื้องต้น เช่น Validate Input, Rate Limit, Authentication, Normalization แล้วส่งต่อให้ Order Manager
4. **Order Manager** ทำ Risk Check ตามกฎที่ Risk Manager กำหนด
5. ถ้าผ่าน Risk Check แล้ว Order Manager จะเช็คต่อว่า Wallet มีเงินพอสำหรับ Order นี้
6. Order ถูกส่งไป **Matching Engine** ถ้าจับคู่ได้ Matching Engine จะสร้าง **Execution** 2 ตัว คือฝั่งซื้อ 1 ตัว ฝั่งขาย 1 ตัว และเพื่อให้ผลการจับคู่เหมือนเดิมทุกครั้งเวลา Replay ทั้ง Order และ Execution จะถูกเรียงลำดับใน **Sequencer** (ตั้งแต่ขั้นที่ 6. – 9.)
7. Execution ถูกส่งกลับไปให้ Client (ตั้งแต่ขั้นที่ 10. – 14.)

#### Market Data Flow
![[stockexchange_high-level_design_market_data_flow.png]]

เป็น Flow ที่จะไล่ Execution จาก Matching Engine ไปถึง Broker ผ่าน Data Service
- **M1**: Matching Engine สร้าง Stream ของ Execution ตอนจับคู่ได้ ส่งไปที่ **Market Data Publisher**
- **M2**: Market Data Publisher สร้าง Candlestick Chart และ Order Book จาก Stream ของ Order และ Execution ซึ่งรวมเรียกว่า **Market Data** แล้วส่งไปที่ Data Service
- **M3**: Market Data ถูกเก็บใน Storage เฉพาะทางสำหรับวิเคราะห์ Real-time ส่วน Broker ก็ต่อกับ Data Service เพื่อเอา Market Data ไปให้ลูกค้า

#### Reporting Flow

![[stockexchange_high-level_design_reporting_flow.png]]

เป็น Flow สำหรับการสร้าง Report เพื่อนำไปวิเคราะห์ต่อตามความต้องการ
- **R1, R2**: **Reporter** รวบรวม Field ที่ต้องใช้ทำรายงาน (เช่น `client_id`, `price`, `quantity`, `order_type`, `filled_quantity`, `remaining_quantity`) จาก Order และ Execution แล้วเขียนลง Database

### Critical Path Components

ทีนี้เราจะมาดูว่าแต่ละตัวบน Critical Path ทำอะไรบ้าง? หลักๆ ก็จะมี 4 ตัว
1. Matching Engine
2. Sequencer
3. Order Manager
4. Client Gateway

#### Matching Engine

หรือเรียกว่า Cross Engine มีหน้าที่...
- **ดูแล Order Book ของหุ้นแต่ละตัว** (Order Book คือรายการคำสั่งซื้อและขายของหุ้นตัวหนึ่ง)
- **จับคู่คำสั่งซื้อกับคำสั่งขาย** จับคู่ได้ 1 ครั้งจะได้ Execution 2 ตัว (ฝั่งขาย 1 และ ฝั่งซื้อ 1) ต้องเร็วและถูกต้อง
- **ส่ง Stream ของ Execution ออกไปเป็น Market Data**

สิ่งที่สำคัญคือ ต้อง **Deterministic** 
คือถ้าใส่ลำดับ Order เดิมเข้าไป ต้องได้ลำดับ Execution เดิมออกมาเสมอ เพราะเป็นพื้นฐานของการกู้คืนระบบทั้งหมด

#### Sequencer

**Sequencer** มีหน้าที่ในการแปะ **Sequence ID** ให้ทุก Order ก่อนเข้า Matching Engine และแปะให้ทุกคู่ Execution ที่ออกมาด้วย แต่ละตัวจะมีลำดับ Sequence ID ของตัวเอง และ Sequence ID ต้องเป็นเลขเรียงต่อกัน จะได้เห็นทันทีถ้ามีเลขไหนหายไป

>[!Note] ทำไมต้องแปะ Sequence ID?
>- **Timeliness and Fairness** - ใครมาก่อนได้ก่อน ชัดเจน
>- **Fast Recovery / Replay** - ถ้าระบบล่ม ก็เอา Order ชุดเดิมมาเล่นซ้ำตามลำดับเดิม แล้วจะได้ State เดิมเป๊ะ
>- **Exactly-once Guarantee** - รู้ว่าอะไรทำไปแล้วบ้าง ไม่ทำซ้ำ

และเพราะสิ่งที่สำคัญคือ **ลำดับ** ไม่ใช่เวลาที่เกิดจริง ตอน Replay ก็ไม่ต้องรอตามจังหวะเวลาเดิม เอา Event ที่เกิดห่างกันไม่เท่ากันมาเรียงชิดกันแล้วเล่นรวดเดียวได้เลย ทำให้กู้คืนได้เร็วมาก

นอกจากนี้ Sequencer ยังทำหน้าที่เป็น **Message Queue** (ส่ง Order เข้า และส่ง Execution กลับ) และเป็น **Event Store** ที่เก็บ Order กับ Execution ไว้ด้วย คล้ายมี Kafka 2 Stream ต่อกับ Matching Engine จริงๆ จะใช้ Kafka ก็ได้ ถ้า Latency ของมันต่ำและคาดเดาได้มากกว่านี้

#### Order Manager

Order Manager มีหน้าที่รับ Order เข้ามาฝั่งหนึ่ง และรับ Execution กลับมาอีกฝั่ง ทำหน้าที่ดูแลสถานะของ Order โดยฝั่งรับ Order จาก Client Gateway จะ...
1. ส่ง Order ไป Risk Check เช่น เช็คว่า User ซื้อขายไม่เกิน $1M ต่อวัน
2. เช็คกับ Wallet ว่ามีเงินพอ (อ่านการออกแบบ Wallet ได้ใน [[18 Digital Wallet|Digital Wallet]])
3. ส่ง Order ไปที่ Sequencer โดยส่งแค่ Field ที่ Matching Engine ต้องใช้ เพื่อให้ Message เล็กลง

ส่วนฝั่งรับ Execution ก็ส่งกลับไปที่ Broker ผ่าน Client Gateway

ความยากของ Order Manager อยู่ที่การเปลี่ยนสถานะของ Order ซึ่งในตลาดจริงมีเป็นหมื่นกรณี เพราะงั้น **Event Sourcing** เลยเหมาะมาก (เล่าต่อใน Section 3)

#### Client Gateway

Client Gateway มีหน้าที่รับ Order จาก Client แล้วส่งต่อให้ Order Manager หน้าที่ก็เหมือน API Gateway ทั่วๆ ไป เช่น
- **Auth** - ยืนยันตัวตน
- **Validation** - ตรวจความถูกต้องของข้อมูล
- **Rate Limit** - จำกัดจำนวน Request (อ่านเพิ่มเติมได้ใน [[01 Rate Limiter|Rate Limiter]])
- **Normalization** - แปลงข้อมูลให้เป็นรูปแบบเดียวกัน
แต่ก็จะมีหน้าที่เฉพาะด้วย อย่างเช่น...
- **FIX Support** - รองรับ FIX Protocol

ด้วยความอยู่บน Critical Path เพราะงั้นก็ควรจะเป็น Component ที่เบา และสร้างได้เร็วที่สุด ส่วน Function ที่ซับซ้อนให้ไปอยู่ที่ Matching Engine กับ Risk Check แทน

### API Design

1. **POST** `/v1/order` ส่ง Order (ต้อง Authentication)

**Request Body**

| Field    | Description         | Type   |
| -------- | ------------------- | ------ |
| symbol   | สัญลักษณ์หุ้น       | String |
| side     | buy หรือ sell       | String |
| price    | ราคาของ Limit Order | Long   |
| quantity | จำนวนหุ้น           | Long   |

**Response**

| Field             | Description              | Type   |
| ----------------- | ------------------------ | ------ |
| id                | Order ID                 | Long   |
| createdAt         | เวลาที่ระบบสร้าง Order   | Long   |
| filledQuantity    | จำนวนที่จับคู่สำเร็จแล้ว | Long   |
| remainingQuantity | จำนวนที่ยังรอจับคู่      | Long   |
| status            | new / canceled / filled  | String |
| (ที่เหลือ)        | เหมือนใน Request Body    |        |

2. **GET** `/v1/execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}` ดูผลการจับคู่ (ต้อง Authentication)

**Response**

```swagger
{
	"executions": [
		{
			"id": 1,            # ID ของ Execution
			"orderId": 101,     # ID ของ Order
			"symbol": "AAPL",
			"side": "buy",
			"price": 10010,
			"orderType": "limit",
			"quantity": 200     # จำนวนที่จับคู่ได้
		}
	]
}
```

3. **GET** `/v1/marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}` ดู L2 Order Book ของหุ้นตามจำนวนระดับราคาที่ต้องการ

```swagger
{
	"bids": [{"price": 10008, "size": 2000}, ...],   # ฝั่งเสนอซื้อ
	"asks": [{"price": 10010, "size": 1800}, ...]    # ฝั่งเสนอขาย
}
```

4. **GET** `/v1/marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}` ดูข้อมูลแท่งเทียน

```swagger
{
	"candles": [
		{"open": 100.05, "close": 100.10, "high": 100.12, "low": 100.01}
	]
}
```

### Data Models

#### 1. Product, Order, Execution

- **Product**: หุ้นแต่ละตัว เช่น ประเภท, สัญลักษณ์, สกุลเงินที่ใช้ชำระ, Lot Size, Tick Size จะเก็บใน Database อะไรก็ได้และ Cache ได้ดี
- **Order**: คำสั่งซื้อหรือขายที่เข้ามา
- **Execution**: ผลการจับคู่ที่ออกไป (ไม่ใช่ทุก Order จะมี Execution)

![[stockexchange_order_execution_product_schema.png]]

Order กับ Execution โผล่ในทั้ง 3 Flow แต่เก็บต่างกัน
- **Trading Flow**: ไม่เก็บใน Database เพราะต้องเร็วมาก ทุกอย่างทำใน Memory แล้วเก็บไว้ใน Sequencer เพื่อกู้คืน และ Archive หลังตลาดปิด
- **Reporting Flow**: Reporter เขียนลง Database สำหรับ Reconciliation และรายงานภาษี
- **Market Data Flow**: สำหรับ Market Data Publisher (MDP) ในการสร้าง Order Book และ Candlestick Chart

#### 2. Order Book

**Order Book** คือรายการคำสั่งซื้อขายของหุ้นตัวหนึ่ง เรียงตามระดับราคา เป็นโครงสร้างหลักใน Matching Engine ที่ต้องรองรับ...
- **การหาได้ในเวลาคงที่**
- **การเพิ่ม/อัปเดต/ยกเลิก/จับคู่ได้อย่างรวดเร็ว**
- **หา Best Bid / Best Ask ได้**
- **ไล่ดูทีละระดับราคาได้**

![[stockexchange_limit_order_book.png]]

อย่างเช่นในภาพนี้...
1. มี Order ซื้อหุ้น Apple ขนาดใหญ่ 2,700 หุ้น 
2. ระบบจะจับคู่กับคำสั่งขายทั้งหมดในคิวราคา Best Ask (100.10) และถ้ายังได้หุ้นไม่ครบ ก็จะไล่ไปต่อที่คำสั่งขายตัวในคิวราคา 100.11 ไล่หักไปเรื่อยๆ จนครบ อย่าง case นี้ก็จะเป็น... 2,700 - 200 - 400 - 1,100 - 100 - 900 = 0 
3. พอจับคู่เสร็จ คิวที่ 100.10 ก็หมด Best Ask เลยขยับขึ้นไปเป็น 100.11

พอเราลอง Sketch OrderBook ขึ้นมาดูคร่าวๆ ตาม requirement ก็จะได้ประมาณนี้
```java
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

**ปัญหาคือ** `List<Order> orders` เป็น List ธรรมดา การลบ Order (ตอนยกเลิกหรือจับคู่) เลยไม่ได้ O(1) ทางแก้คือเปลี่ยนเป็น **Doubly-linked List** แทน List ไป
- **ส่ง Order ใหม่** - เพิ่ม Order ต่อท้าย PriceLevel ซึ่ง Doubly-linked List ทำได้ O(1)
- **จับคู่ Order** - ลบ Order ตัวแรกของ PriceLevel ซึ่งทำได้ O(1)
- **ยกเลิก Order** - ใช้ `orderMap` หา Order ที่จะยกเลิกได้ใน O(1) แล้วถ้าเป็น Singly-linked List ก็ต้องไล่หา Node ก่อนหน้าทั้ง List ถึงจะลบได้ ซึ่งเป็น O(n) แต่พอเป็น Doubly-linked List Order แต่ละตัวมี Pointer ชี้ไปตัวก่อนหน้าอยู่แล้ว เลยลบได้ใน O(1) โดยไม่ต้องไล่

#### 3. Candlestick Chart

เป็นอีกโครงสร้างหลักใน Market Data Publisher (MDP) พอหมดช่วงเวลาของแท่งหนึ่ง ก็สร้างแท่งใหม่ต่อท้าย Linked List
```java
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

**ปัญหาคือ** เราต้องเก็บกราฟของหุ้นหลายตัวในหลายช่วงเวลา ทำให้กิน Memory เยอะมาก แก้ได้ 2 ทาง
- จำกัดจำนวนแท่งที่อยู่ใน Memory ที่เหลือเขียนลง Disk
- ใช้ **Ring Buffer** เก็บแท่งเทียนแทน Linked List

> [!Note] Ring Buffer คืออะไร? 
> **Ring Buffer** (หรือ Circular Buffer) คือ Queue ขนาดคงที่ที่หัวต่อกับหาง Producer เขียนข้อมูลเข้าไปวนเรื่อยๆ ส่วน Consumer หนึ่งตัวหรือหลายตัวก็ดึงออกไป ข้อดีคือ...
> - **จองพื้นที่ไว้ล่วงหน้า** ไม่ต้องสร้างหรือคืน Object ระหว่างทำงาน
> - **Lock-free** หลาย Thread ใช้พร้อมกันได้โดยไม่ต้องแย่ง Lock
> - ใช้เทคนิคเสริมได้อีก เช่น **Padding** ให้ Sequence Number ไม่อยู่ใน Cache Line เดียวกับข้อมูลอื่น
> 
> ตลาดหุ้นใช้ Ring Buffer หลายที่มาก ทั้งใน MDP และ Sequencer ที่จะเห็นใน Section 3

---

## 3. เจาะลึกจุดสำคัญๆ

### 1. Performance

โจทย์ของเรื่อง Performance คือทำยังไงให้ Latency ของตลาดไม่ใช่แค่ต้องต่ำโดยเฉลี่ย แต่ต้อง constant ด้วย

เนื่องจาก Latency คือผลรวมของเวลาทำงานทุกขั้นบน Critical Path เพราะงั้นลดได้ 2 ทาง
1. **ลดจำนวนงานบน Critical Path**: Critical Path มีแค่ `gateway -> order manager -> sequencer -> matching engine` เท่านั้น ขนาด Logging ยังถูกเอาออกเลย
2. **ลดเวลาของแต่ละงาน**: ลดหรือตัดการใช้ Network และ Disk และลดเวลาประมวลผลของแต่ละงาน

ใน High-Level Design ทุก Component อยู่คนละ Server คุยกันผ่าน Network ซึ่ง Round-trip ประมาณ 500 Microsecond ต่อทอด หลายทอดรวมกันก็เป็นหลัก Millisecond และ Sequencer ยังต้องเขียน Event ลง Disk ซึ่งใช้เวลาเป็นสิบ Millisecond รวมแล้วเป็นหลักสิบ Millisecond ซึ่งเคยโอเคในยุคแรกๆ แต่ทุกวันนี้ไม่พอแล้ว

เพราะงั้นเราสามารถลด Latency ลงเหลือหลักสิบ **Microsecond** ด้วยการ **เอาทุกอย่างมาไว้บน Server เดียวกัน** แล้วให้ Component คุยกันผ่าน **mmap** แทน ตามแนวทางที่ 2 นั่นเอง

>[!Tip] mmap คืออะไร?
>mmap เป็น System Call ของ UNIX ที่ Map ไฟล์เข้ากับ Memory ของ Process ทำให้หลาย Process แชร์ Memory กันได้เร็วมาก และถ้าไฟล์นั้นอยู่ใน `/dev/shm` ซึ่งเป็น File System บน Memory ก็จะไม่แตะ Disk เลย
>
>ตลาดสมัยใหม่จะใช้ mmap สร้าง Message Bus ให้ Component บน Critical Path คุยกัน ส่ง Message ใช้เวลาไม่ถึง 1 Microsecond และพอใช้ mmap เป็น Event Store คู่กับ Event Sourcing ก็ได้ Microservice ที่ Latency ต่ำมากๆ ภายใน Server เดียว

![[stockexchange_low_latency_single_server_design.png]]

ทีนี้ก็จะเหลืออีกอย่างนึงคือทำยังไงให้ Latency มัน constant?

การซื้อขายทุกครั้งต้องใช้เวลาเกือบเท่ากัน ซึ่งเราสามารถวัดได้ด้วยการเช็ค Latency ที่ P99 หรือเข้มกว่านั้นคือ P99.99 โดยใช้เครื่องมืออย่าง `HdrHistogram` ช่วยคำนวณ

ถ้า Latency เหวี่ยงขึ้นลงเยอะ ต้องหาสาเหตุให้เจอ เช่น ใน Java มักเป็นเพราะ Safe Point อย่าง Stop-the-World Garbage Collection ของ HotSpot JVM ที่หยุดทุก Thread ชั่วคราว

### 2. Event Sourcing

พื้นฐานของ Event Sourcing อ่านได้ใน [[18 Digital Wallet#Event Sourcing|Digital Wallet]] 

แต่ถ้าสรุปสั้นๆ คือแทนที่จะเก็บแค่สถานะล่าสุดของ Order ก็เก็บทุก Event ที่ทำให้สถานะเปลี่ยน (`NewOrderEvent` → `OrderFilledEvent`) แล้วกู้สถานะกลับมาได้ด้วยการเล่น Event ซ้ำ

ทีนี้มาดูว่าตลาดหุ้นใช้ Event Sourcing คู่กับ mmap Event Store ยังไง
![[stockexchange_event_sourcing_design.png]]

1. External คุยกับตลาดผ่าน **FIX** Protocol
2. Gateway แปลง FIX เป็น **FIX over SBE (Simple Binary Encoding)** ที่เร็วและเล็กกว่า แล้วส่ง Order แต่ละตัวเป็น `NewOrderEvent` ผ่าน Event Store Client
3. Order Manager (ที่ฝังอยู่ใน Matching Engine) รับ `NewOrderEvent` จาก Event Store, Validate แล้วเพิ่มเข้า Order State ของตัวเอง จากนั้นส่งต่อไปที่ Matching Core
4. ถ้าจับคู่ได้ ก็สร้าง `OrderFilledEvent` ส่งเข้า Event Store
5. Component อื่นๆ เช่น MDP กับ Reporter ก็ Subscribe Event Store แล้วประมวลผลตามหน้าที่ของตัวเอง

ซึ่งคล้าย Pub-Sub ของ Kafka มาก ถ้าไม่ต้องการ Latency ต่ำขนาดนี้ จะใช้ Kafka ก็ยังได้

Design นี้ต่างจาก High-Level Design 2 จุด
1. **Order Manager ฝังอยู่ใน Component ที่ต้องใช้สถานะของ Order แทนที่จะเป็น Service กลาง** เช่นใน Reporter เพราะถ้าทุกตัวต้องไปถาม Order Manager ตัวกลาง Latency จะแย่
2. **Sequencer หายจาก Design ไปอยู่กับ Event Store** แล้วให้ Sequencer มีหน้าที่แค่เรียง Sequence ID ที่อยู่ใน Event

Event Store หนึ่งตัวต้องมี **Sequencer แค่ตัวเดียว (Single Writer)** เพราะถ้ามีหลายตัวก็จะแย่งกัน Lock เพื่อ Write ข้อมูล ทำให้เสียเวลา
![[stockexchange_sequencer_design.png]]

1. Component แต่ละตัว (เช่น Gateway, Matching Engine) เขียน Event ลง Ring Buffer ของตัวเอง
2. Sequencer ดึง Event จาก Ring Buffer ของแต่ละตัว แล้วแปะ Sequence ID
3. Sequencer เขียน Event ลง Event Store

และมี Sequencer สำรองไว้ **เผื่อ Main Sequencer ล่ม**

### 3. High Availability and Fault Tolerance

Service ที่เป็น Stateless อย่าง Client Gateway เราสามารถเพิ่ม Server เพื่อทำให้เกิด High Availability ได้อย่างง่ายดาย 

ส่วนที่เป็น Stateful อย่าง Order Manager กับ Matching Engine ต้อง Copy State ไปที่ตัวสำรองด้วย (อ่านเพิ่มเติมได้ใน [[08 Stateful VS Stateless|Stateful VS Stateless]]) ทีนี้มาไล่จากระดับเครื่องเดียวไปถึงหลาย Data Center

#### ขั้นที่ 1: Hot-Warm ใน Server เดียว

![[stockexchange_hotwarm.png]]

- **Hot Matching Engine**: ตัวหลัก รับ Event แล้วส่ง Event ออกไปที่ Event Store
- **Warm Matching Engine**: รับและประมวลผล Event เดียวกันเป๊ะ แต่ **ไม่ส่ง Event ออกไป**

พอตัว Hot ล่ม ตัว Warm ก็ขึ้นมาเป็นตัวหลักได้ทันที เพราะมี State ล่าสุดอยู่แล้ว ส่วนถ้าตัว Warm ล่มเอง พอรีสตาร์ทก็ Replay จาก Event Store ได้ และใช้ **Heartbeat** จาก Matching Engine ประกอบกับการ Monitor ปกติ เพื่อรู้ว่าตัว Hot มีปัญหา

#### ขั้นที่ 2: ขยายไปหลายเครื่อง

**ปัญหาคือ** Hot-Warm ใช้ได้แค่ใน Server เดียว ถ้า Server พังทั้งเครื่องก็จบ เลยต้องให้ทั้ง Server เป็น Hot หรือ Warm และ Replicate Event Store จากเครื่อง Hot ไปเครื่อง Warm ทุกเครื่อง ซึ่งใช้ **Reliable UDP** ส่ง Event ไปพร้อมกันได้เร็ว

และถ้าจะกันเหตุใหญ่อย่างแผ่นดินไหวหรือไฟดับทั้งพื้นที่ ก็ต้อง Replicate ไปหลาย Data Center ในหลายเมือง ซึ่งต้องตอบอีก 4 คำถาม

#### ขั้นที่ 3: "ล่ม" จริงๆ คืออะไร

ฟังดูง่ายแต่ไม่ง่าย เช่น...
- ระบบอาจส่ง **False Alarm** ทำให้สลับไปตัวสำรองโดยไม่จำเป็น
- ถ้าตัวหลักล่มเพราะ **Bug** ตัวสำรองที่รัน Code เดียวกันก็จะล่มด้วย Bug เดิมหลังสลับไป จนตัวสำรองล่มหมดทุกตัว

ทางแก้คือ ตอนเปิดระบบใหม่ๆ อาจต้องสลับด้วยมือก่อน พอมั่นใจแล้วค่อยทำให้อัตโนมัติ และใช้ **Chaos Engineering** (จงใจทำให้ระบบพังเพื่อหาจุดอ่อน) ช่วยหา Edge Case

#### ขั้นที่ 4: เลือก Leader ด้วย Raft

พอตัดสินใจสลับแล้ว จะให้ใครขึ้นมาแทน? ใช้ Algorithm เลือก Leader ที่พิสูจน์แล้ว เช่น **Raft**
![[stockexchange_raft_event_replication.png]]

ในภาพมี Cluster 5 Server แต่ละตัวมี Event Store ของตัวเอง Leader ส่ง Event ใหม่ไปให้ Follower ทุกตัวผ่าน RPC (`AppendEntries`) แล้ว Follower เก็บลง mmap Event Store ของตัวเอง ซึ่ง Raft ต้องได้เสียงอย่างน้อย **N/2 + 1** ถึงจะทำงานได้ เช่นถ้ามี 5 ตัวก็ต้องได้อย่างน้อย 3 เสียง

การเลือก Leader จะทำงานแบบนี้...
1. Leader ส่ง Heartbeat (`AppendEntries` ที่ไม่มีเนื้อหา) ให้ Follower ตลอด
2. ถ้า Follower ไม่ได้รับ Heartbeat นานเกินกำหนด ก็จะ **Election Timeout** แล้วเริ่มเลือกตั้งใหม่
3. Follower ตัวแรกที่ Timeout จะกลายเป็น **Candidate** แล้วขอเสียงจากตัวอื่น (`RequestVote`)
4. ถ้าได้เสียงเกินครึ่ง ก็เป็น Leader ใหม่ แต่ถ้า Candidate มี **Term** ต่ำกว่าตัวอื่น ก็เป็น Leader ไม่ได้
5. ถ้ามีหลายตัวเป็น Candidate พร้อมกัน เรียกว่า **Split Vote** ก็จะ Timeout แล้วเลือกตั้งใหม่

>[!Note] Term คืออะไร?
>**Term** คือช่วงเวลาใน Raft แต่ละ Term เริ่มด้วยการเลือกตั้ง แล้วตามด้วยการทำงานปกติ ถ้าเลือกตั้งแล้ว Split Vote ก็จบ Term นั้นแล้วขึ้น Term ใหม่
>
![[stockexchange_raft_term.png]]

### 4. Matching Algorithms

มาดูการจับคู่แบบ Pseudo-code กัน
```java
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);
}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }
    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

ไล่ทีละส่วน
- **`handleOrder`**: เช็คก่อนว่า Sequence ID ตรงกับลำดับที่รออยู่มั้ย ถ้าไม่ตรงก็ Error ทันที (นี่คือเหตุผลที่ Sequence ต้องเรียงต่อกัน) จากนั้น Validate แล้วแยกไปทาง `NEW` หรือ `CANCEL`
- **`handleNew`**: ถ้าเป็นคำสั่งซื้อก็ไปจับคู่กับฝั่งขาย ถ้าเป็นคำสั่งขายก็ไปจับคู่กับฝั่งซื้อ
- **`handleCancel`**: ถ้า Order ไม่อยู่ใน `orderMap` แล้ว แปลว่าถูกจับคู่ไปแล้ว ยกเลิกไม่ได้ ถ้ายังอยู่ก็ลบออกและตั้งสถานะเป็น `CANCELED`
- **`match`**: ไล่จับคู่กับ Order ในคิวราคาเดียวกันทีละตัว จนจำนวนที่เหลือเป็น 0 หรือคิวหมด

Pseudo-code นี้ใช้ **FIFO (First In First Out)** คือ Order ที่มาก่อนในระดับราคาเดียวกันได้จับคู่ก่อน

### 5. Market Data Publisher

#### Data Limitation

ข้อมูลแท่งเทียนรายวันหาฟรีได้จาก Google Finance แต่ L2/L3 แพงมาก ทำให้ Hedge Fund หลายเจ้าเลยเก็บข้อมูลเองผ่าน Real-time API ของตลาด 

MDP จึงมีการให้บริการในหลายระดับ เช่น User ทั่วไปดู L2 ได้ 5 ระดับ ถ้าอยากดู 10 ระดับต้องจ่ายเพิ่ม และเพราะ Memory ของ MDP มีจำกัด ก็ต้องจำกัดจำนวนแท่งเทียนที่เก็บไว้ด้วย โดยเก็บใน Ring Buffer แยกตามช่วงเวลา (1 นาที, 1 ชั่วโมง, 1 วัน)

![[stockexchange_mdp.png]]

#### Data Distribution Fairness

ในการซื้อขายหุ้น การได้ข้อมูลเร็วกว่าคนอื่นเหมือนมองเห็นอนาคต เพราะงั้นตลาดที่ถูกกำกับดูแลต้องการันตีว่า **ทุกคนได้ Market Data พร้อมกัน**

สมมติว่า MDP เก็บรายชื่อ Subscriber เรียงตามลำดับที่ต่อเข้ามา แล้วส่งให้คนแรกก่อนเสมอ ผลคือ Client ที่ฉลาดจะแย่งกันต่อเป็นคนแรกตอนตลาดเปิด ทางแก้มี 2 อย่าง
- **Multicast ด้วย Reliable UDP** - ส่งข้อมูลถึงหลายคนพร้อมกันในทีเดียว
- **สุ่มลำดับ Subscriber** ตอนต่อเข้ามา

การส่งข้อมูลบน Internet มี 3 แบบ
- **Unicast** - จาก 1 ต้นทางไป 1 ปลายทาง
- **Broadcast** - จาก 1 ต้นทางไปทั้ง Subnet
- **Multicast** - จาก 1 ต้นทางไปกลุ่มเครื่องที่อาจอยู่คนละ Subnet

Multicast นิยมใช้มากในตลาด เพราะถ้าตั้งผู้รับไว้ใน Multicast Group เดียวกัน ในทางทฤษฎีทุกคนจะได้ข้อมูลพร้อมกัน **ปัญหาคือ** UDP เชื่อถือไม่ได้ Datagram อาจไปไม่ถึงทุกคน เลยต้องมีกลไกส่งซ้ำมาช่วย

### 6. Network Security

- **แยก Service สาธารณะออกจาก Service ภายใน** เพื่อไม่ให้ DDoS กระทบลูกค้าที่สำคัญที่สุด ถ้าต้องให้ข้อมูลเดียวกัน ก็ทำสำเนาแบบอ่านอย่างเดียวไว้หลายชุด
- **ใช้ Cache** เก็บข้อมูลที่ไม่ค่อยเปลี่ยน Query ส่วนใหญ่ จะได้ไม่ไปถึง Database (อ่านเพิ่มเติมได้ใน [[06 Cache|Cache]])
- **ออกแบบ URL ให้ทนต่อ DDoS** เช่น URL `https://my.website.com/data?from=123&to=456` ผู้โจมตีสร้าง Request ไม่ซ้ำได้ไม่จำกัดแค่เปลี่ยนตัวเลข แต่ถ้าใช้ `https://my.website.com/data/recent` จะ Cache ที่ CDN ได้
- **มี Safelist / Blocklist**: Network Gateway หลายตัวมีให้อยู่แล้ว
- **Rate Limiting**: ใช้ป้องกัน DDoS

> [!Tip] ส่งท้าย 
> อ่านจบแล้วอาจรู้สึกว่า Design ที่ดีที่สุดของตลาดใหญ่ๆ คือเอาทุกอย่างไว้บน Server ยักษ์ตัวเดียว หรือแม้แต่ Process เดียว ซึ่งก็มีตลาดที่ทำแบบนั้นจริงๆ
> 
> แต่ช่วงหลังตลาด Crypto หลายเจ้าก็รันบน Cloud และโปรเจกต์ DeFi บางตัวใช้ **AMM (Automatic Market Making)** ที่ไม่ต้องมี Order Book เลยด้วยซ้ำ Cloud ทำให้คนเข้าสู่วงการนี้ได้ง่ายขึ้นมาก

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Stock Exchange ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---