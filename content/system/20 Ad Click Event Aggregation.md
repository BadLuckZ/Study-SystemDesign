---
title: 20 Ad Click Event Aggregation
tags:
  - system
  - systemdesign
  - ad_click_event_aggregation
  - real_time_bidding
  - map_reduce
  - kappa
  - watermark
  - hotspot
  - snapshot
  - reconciliation
  - delivery_sementics
---
ไม่ว่าจะ platform ไหนๆ มันก็จะมี ads เต็มไปหมด จะเป็น Banner เล็กๆ ใน Platform หรือว่าเป็น Video เลย กระทั่ง Platform ยอดนิยมยาก Google, Facebook, Tiktok ก็ยังปรากฎ Ads Area เลย เพราะมันก็เป็นแหล่งที่ทำเงินได้เยอะมากๆ อ่ะนะ จากจำนวนการกด Click ของ User ที่ผ่านไปมาอ่ะแหละ แล้วระบบแบบนี้มันทำยังไงกันนะ...

>[!Note] Real-Time Bidding (RTB)  
>Real-Time Bidding คือ กระบวนการซื้อขายพื้นที่โฆษณาแบบอัตโนมัติ 
>- ฝั่งคนที่ต้องการจะลงโฆษณาก็จะเสนอราคาของพื้นที่โฆษณาผ่าน DSP (Demand Side Platform) 
>- ฝั่งเจ้าของพื้นที่โฆษณาก็จะเสนอขายพื้นที่โฆษณาผ่าน SSP (Supply Side Platform) 
>
>ถ้า match กัน ระบบก็จะพามาจับคู่กันที่ Ad Exchange ซึ่งกระบวนการทั้งหมดนี้จบภายในไม่ถึง 1 วินาที
>
>![[adclick_rtb.png]]

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: ข้อมูลที่เข้ามามีหน้าตาเป็นไงเหรอครับ?
A1: เป็น Log File ที่กระจายตาม Server ต่างๆ โดยแต่ละ Event ก็มี ad_id, click_timestamp, user_id, ip ละก็ country อ่ะๆ

Q2: มีข้อมูลที่เข้ามาเยอะแค่ไหนเหรอครับ?
A2: ก็... 1 billion clicks ต่อวันอ่ะ ละก็จะโตขึ้นเรื่อยๆ ปีละ 30%

Q3: มี Query แบบไหนที่ใช้เป็นหลักบ้างเหรอครับ?
A3: มีหลักๆ 3 แบบ
- จำนวน clicks ที่กด ad หนึ่งในเวลา M นาทีที่ผ่านมา
- Top100 ads ที่ถูกคลิกเยอะสุดใน M นาทีที่ผ่านมา
- กรอง ads ด้วย ip, user_id, country ได้

Q4: มี Edge Case อะไรบ้างเหรอครับ?
A4: Event เข้า Log File ช้า, Event ซ้ำ, ระบบล่ม ไรงี้ๆ

Q5: แล้วระบบนี้นี่ต้องเร็วแค่ไหนเหรอครับ
A5: อาจจะต่างจาก RTB นะ คือมันช้าได้แหละเพราะเป็น Analysis เนอะ

### Non-Functional Requirement

- Correctness: ผลการนับ Event ต้องถูกต้อง เพราะมีผลต่อการคิดเงิน รวมถึงสามารถจัดการ Event ที่อาจจะเข้ามาซ้ำได้
- Robustness: ถ้ามีส่วนของระบบที่ล่มไป ระบบต้องไม่พังทั้งหมด ล่มแค่ส่วนนั้น ไม่ลามไปส่วนอื่นๆ
- Latency: ระยะเวลาในการแสดงผลตั้งแต่การคลิกควรใช้เวลาไม่นาน

---

## 2. Sketch ภาพระบบคร่าวๆ

### API Design

คือ... User ที่ใช้ API นี้ไม่ใช่ประชาชนคนทั่วไป แต่เป็นพวก Data Scientist หรือคนลง Ads เพราะงั้น... การ filter ข้อมูลสามารถใส่มาเป็น Query Parameter ได้เลย ก็จะมีหลักๆ 2 เส้น

1. **GET** /v1/ads/:ad_id/aggregated_count - ดึงจำนวน Click ของ Ad นั้นๆ ออกมา โดยจะมี parameters ต่อไปนี้

| Field  | Description                                                                       |
| ------ | --------------------------------------------------------------------------------- |
| from   | Start Time                                                                        |
| to     | End Time                                                                          |
| filter | รหัสการกรอง เช่น 001 = เอาแค่เฉพาะ country = USA (จะมีรหัสอะไรบ้างก็ไปตกลงกันเอง) |
แล้วจะได้ออกมาเป็น List ของ Ads ที่มีจำนวน Click ของแต่ละ Ad ติดมาด้วย

2. **GET** /v1/ads/popular_ads - ดึง Top N Ads ที่ถูกคลิกเยอะสุดใน M นาทีที่ผ่านมา

| Field  | Description                 |
| ------ | --------------------------- |
| count  | จำนวน Ads Top N ที่จะดึงมา  |
| window | M นาทีล่าสุดย้อนจากปัจจุบัน |
| filter | รหัสการกรอง                 |
แล้วจะได้ออกมาเป็น List ของ Ads ที่เป็น Top N นั่นเอง

### Data Model

ข้อมูลในระบบนี้จะมี 2 แบบหลักๆ

1. Raw Data หรือก็คือ Click Event เช่น...

| ad_id | click_timestamp     | user_id | ip            | country |
| ----- | ------------------- | ------- | ------------- | ------- |
| ad001 | 2021-01-01 00:00:01 | user1   | 207.148.22.22 | usa     |
| ad001 | 2021-01-01 00:00:20 | user1   | 207.148.22.22 | usa     |
| ad001 | 2021-01-01 00:01:05 | user1   | 207.148.22.22 | usa     |
| ad002 | 2021-01-01 00:01:05 | user1   | 207.148.22.22 | usa     |

2. Aggregated Data หรือก็คือผลรวมที่มีการนับในทุกๆ นาที ละก็มี filter_id ที่บ่งบอกรหัสการกรองเอาไว้ ละก็ group ข้อมูลที่รอดจากการ filter ไปด้วย (ad_id, click_minute) เช่น...

| ad_id | click_minute    | filter_id | counter |
| ----- | --------------- | --------- | ------- |
| ad001 | 20210101_000000 | 0001      | 2       |
| ad001 | 20210101_000100 | 0001      | 1       |
| ad002 | 20210101_000100 | 0001      | 1       |

โดยที่ filter_id ก็จะประมาณนี้
- 0001 เอามาแค่เฉพาะ country ที่เป็น usa
- 0002 เอามาแค่เฉพาะ ip 207.148.22.22

ที่ต้องมี filter_id ไว้ด้วยก็เพื่อทำให้การ query ข้อมูลทำได้สะดวกขึ้นอ่ะนะ

แล้วเราควรจะใช้ Data แบบไหนดี... คำตอบคือ เก็บทั้งคู่ เพราะอันนึงคือ Raw Data เป็น Source of Truth ที่เรามี อีกอันนึงอย่าง Aggregated Data ก็ช่วยทำให้การ Query ข้อมูลทำได้เร็วขึ้น

### Database Choice

ด้วยความที่ระบบนี้มีการเขียนข้อมูลเยอะมาก จากการที่ Clicking Ads เยอะ ทำให้เป็นการสร้าง Record ใหม่เข้า Log File เยอะ ทำให้ Raw Data มีขนาดใหญ่ตามไปด้วย 

ไหนยังจะการที่ต้อง Query ข้อมูลเพื่อสร้าง Aggregated Data อย่างสม่ำเสมอด้วย อันนี้นี่ต้องทำทั้ง Read และ Write ละเป็นการ Read และ Write ที่เกิดขึ้นบ่อยมาก ไม่ต่างกับ Write ของ Raw Data เลย

เพราะงั้น Choice ที่เหมาะสมจึงเป็น NoSQL อย่าง Cassandra ที่มีการ optimized มาสำหรับการ Write Data เยอะๆ อยู่แล้ว รวมถึงการ Query ข้อมูลตามเวลาก็ทำได้เร็วเหมือนกัน

### High-Level Design

![[adclick_aggregation_flow.png]]

คือ Flow ของการสร้าง Aggregated Data จะเป็นประมาณว่า...
1. เรามี Log Watcher ที่ดูเรื่องการ Click Ads พอมีการ Click ก็ส่ง Data ใหม่ไปให้กับ Data Aggregation Service
2. Data Aggregation Service นำ Data ที่ได้มา filter และ group ร่วมกับ Raw Data แล้วใส่ใน Aggregated Data
3. UI ก็จะมาดึงข้อมูลจาก Aggregated Data ไปแสดงผล

แต่ว่า...ถ้ามันมีแค่นี้เลย มันจะมีปัญหาตอนที่มีการ Click Ads เยอะๆ เพราะจะทำให้มีการ Push ข้อมูลเข้ามาหา Data Aggregation Service เยอะจนอาจจะทำให้ระบบล่มไปได้ หรือต่อให้ไม่ล่ม ก็จะมี Aggregation Data เข้าสู่ Database เยอะมากๆ จนอาจจะทำให้ Part การ Write เข้า Database ล่มไปได้เหมือนกัน

ซึ่งเราแก้ได้ด้วยการใช้ Message Queue ให้มันทำงานแบบ Asynchronous ไป และทำให้เราสามารถ Scale ส่วนที่เป็นการ Write เพื่อรองรับการ Click Ads เยอะๆ ได้ (อ่านเพิ่มได้ใน [[14 Synchronous VS Asynchronous|Syn vs Asyn]])

![[adclick_high-level_design.png]]

1. Log Watcher ส่ง Event คลิกเข้า Message Queue แรก
2. Database Writer ดึง Event ไปเก็บเป็น Raw Data
3. Aggregation Service ดึง Event ไปนับทุกนาที แล้วส่งผลเข้า Message Queue ที่สอง
4. Database Writer อีกตัวดึงผลไปเก็บเป็น Aggregated Data
5. Query Service ดึงจาก Aggregated Data ไปแสดงบน Dashboard

ทีนี้... แล้ว Data Aggregation Service มันทำงานอะไรยังไงนะ แล้วทำไมถึงไม่ให้ Data Aggregation Service เขียนเข้า Database โดยตรงเลย ทำไมต้องมี Message Queue มาคั่นด้วย มาดูกัน

### Aggregation Service

![[adclick_aggregation_mapreduce.png]]

Aggregation Service ใช้แนวคิดที่เหมือนการทำ Big Data คือ MapReduce ก็คือแตกงานออกเป็น Node เล็กๆ จากนั้นแต่ละ Node ก็ส่งให้ Node กลางเพื่อรวมของไว้ด้วยกัน หลักๆ ก็จะมี Node 3 ประเภท
- **Map Node**: Node ที่คอยอ่าน Event ที่เข้ามา จากนั้นก็ Filter แล้วส่งไปหา Aggregation Node ตาม Pattern ที่กำหนด เช่น Event ที่มี ad_id %2 = 0 ไปที่ Agg Node 1 ที่เหลือไป Agg Node 2 เป็นต้น
- **Aggregation Node**: Node ที่คอยนับว่า ad_id นี้มีการ Click กี่ครั้งในแต่ละนาที เมื่อครบนาทีแล้วก็จะส่งผลไปให้ Reducer Node
- **Reduce Node**: Node ที่รวมผลลัพธ์จาก Aggregation Node ไว้ด้วยกันแล้วส่งต่อ

หน้าตาจริงๆ ของมันจะเป็นประมาณนี้
![[adclick_aggregation_services.png]]

#### Usecase1: นับจำนวน Clicks ของแต่ละ Ads
Map แยกตาม ad_id แล้วแต่ละ Aggregate Node นับของตัวเองได้เลย
![[adclick_aggregate_click_counter.png]]

#### Usecase2: ดึง Top N Ads ที่มีการ Click สูงสุดใน M นาทีที่ผ่านมา
แต่ละ Aggregate Node หา Top N ของตัวเอง แล้ว Reduce Node รวมทุกชุดมาเลือก Top N สุดท้ายอีกที เช่นมี 3 Node แต่ละ Node ส่ง Top 3 มา Reduce ก็เลือก Top 3 จาก 9 ตัวนั้น
![[adclick_aggregate_topn_ads.png]]

#### Usecase3: Filter Data
เพราะเรามี Aggregated Data อยู่แล้ว เราก็นับแยกตามเงื่อนไขไว้ล่วงหน้าเลย เช่น นับแยกตามประเทศ พอมี Query ขอเฉพาะ USA ก็ดึงผลที่นับไว้แล้วได้ทันที วิธีนี้เรียกว่า Star Schema ข้อเสียคือถ้ามีเงื่อนไขเยอะ จำนวนแถวจะเพิ่มขึ้นตามมหาศาล 
![[adclick_aggregate_filter_result.png]]

---

## 3. เจาะลึกจุดที่สำคัญๆ

### Streaming VS Batching

![[adclick_system_types.png]]

อะไรคือ Streaming กับ Batching?
- Streaming คือระบบที่ออกแบบมาเพื่อประมวลผลทันทีที่ข้อมูลเข้ามา ข้อมูลไหลเข้ามาไม่มีที่สิ้นสุด
- Batching คือระบบที่ออกแบบมาเพื่อรอข้อมูลก้อนใหญ่ครบแล้วค่อยประมวลผลทีเดียว

เราสามารถทำให้ระบบของเรามีทั้ง 2 ทางแยกกันได้ โดยเราจะเรียกว่า Lambda Architecture แต่ว่าก็มีข้อเสียคือต้องดูแล code 2 ชุด 

เพราะงั้นเราเลยมี Kappa Architecture ที่ตัด Batch Layer ออกไปเลย แล้วใช้ Streaming Engine ตัวเดียวทำทั้งงานปกติ และงานนับข้อมูลเก่าใหม่ ตามภาพ Architecture ด้านล่าง

![[adclick_lambda_kappa.png]]

จะเห็นว่าในภาพของ Kappa มีแค่ Streaming Layer เนอะ แล้ว Batching Layer อ่ะอยู่ไหน?

คำตอบคือ...ไม่มีแล้ว เพราะ Kappa มองว่าข้อมูลเก่าก็คือ Stream เหมือนกัน แค่เป็น Stream ที่มีจุดเริ่มและจุดจบชัดเจน เวลาต้องนับใหม่ก็แค่ดึง Raw Data มา Replay ผ่าน Streaming Engine ตัวเดิม ทำให้ใช้ Logic และ Codebase ชุดเดียว ไม่ต้องดูแล 2 ระบบเหมือน Lambda

ในระบบเรา การ Replay จะเป็นประมาณนี้...

![[adclick_recalculation_service.png]]

1. Recalculation Service ดึง Raw Data ช่วงที่ต้องนับใหม่ออกมาจาก Raw Data DB
2. ส่งข้อมูลเข้า Data Aggregation Service ที่เป็นโค้ดชุดเดียวกัน แต่แยก Instance ไว้ใช้ Recalculate อย่างเดียว เพื่อไม่ให้กระทบงาน Real-time
3. ผลลัพธ์ถูกส่งเข้า Message Queue ตัวที่ 2 แล้วอัปเดตลง Aggregation DB เหมือนงานปกติเลย

### Time and Watermark

timestamp ที่ใช้นับในระะบนี้มีให้เลือก 2 แบบ
1. Event Time (ตอน User คลิก)	
2. Processing Time (ตอน Server ประมวลผล)

แต่ละแบบก็มีข้อดีข้อเสียของมันอ่ะนะ...

|         | Event Time                                             | Processing Time                                  |
| ------- | ------------------------------------------------------ | ------------------------------------------------ |
| ข้อดี   | แม่นกว่า รู้ว่า User Click เมื่อไหร่จริงๆ              | มีความน่าเชื่อถือ เพราะเป็นเวลาของ Server        |
| ข้อเสีย | เวลามาจาก Client เพราะงั้นมันก็อาจจะผิด หรือถูกปลอมได้ | ถ้า Event มาช้ามาก จะถูกนับผิดนาที (Late Events) |

ด้วยระบบนี้ที่ความแม่นยำสำคัญที่สุด จึงเลือก Event Time แต่ก็ยังจะเจอปัญหาว่า Event ที่มาช้าเลยช่วงนาทีของตัวเองไปนิดเดียวจะไม่ถูกนับ (ไม่ถูกนับในที่นี้คือไม่ใช่ว่าไปช่วงใหม่นะ คือหายไปเลยเหมือนไม่มี Event นี้เกิดขึ้นมาก่อน)

ทางแก้คือใช้หลักการ **Watermark** คือการยืดเวลาปิดของแต่ละ Window ออกไปอีกนิด เช่น 15 วินาที เพื่อให้ Event ที่มาช้าไม่มากยังถูกนับใน Window ที่ถูกต้อง ตามภาพนี้

![[adclick_watermark.png]]

ในภาพ Event 2 เกิดขึ้นช่วงท้ายๆ ของ Window 1 แต่ด้วย Delay ระหว่างทาง ทำให้กว่าจะมาถึง Aggregation Service ก็เลยเวลาปิดของ Window 1 ไปแล้ว ใน Case ปกติ Event 2 จะตกหล่นไปเลย เพราะ Window 1 ปิดไปแล้ว ส่วน Window 2 ก็ไม่นับให้ เพราะ Event Time ของมันไม่ได้อยู่ในช่วงของ Window 2

แต่พอมี Watermark ยืดเวลาปิดของ Window 1 ออกไปอีก 15 วินาที Event 2 ก็มาทันพอดี เลยยังถูกนับใน Window 1 ได้

แต่ว่าก็ต้องเลือก Watermark ดีๆ นะ...
- Watermark ยาว - รับ Event ที่มาช้ามากๆ ได้ แม่นยำขึ้น แต่ผลลัพธ์ก็ออกช้าลงตาม
- Watermark สั้น - ผลลัพธ์ออกเร็ว แต่ Event ที่มาช้าก็หลุดได้ง่ายขึ้น

ส่วน Event ที่มาช้าเกินกว่า Watermark ไปเยอะๆ ก็ไม่ต้องออกแบบอะไรซับซ้อน เพราะเกิดน้อยมาก ปล่อยให้ Reconciliation ตอนสิ้นวันมาเก็บตกแทน

### Aggregation Window

ระบบเราสามารถใช้ช่วงเวลานับ 2 แบบ เพื่อ 2 Purposes ที่ต่างกัน
1. Tumbling Window: แบ่งเวลาเป็นช่วงเท่าๆ กัน และไม่ทับกัน เช่น ทุกๆ 1 นาที ซึ่งเหมาะกับการนับคลิกทุกนาที
2. Sliding Window: ช่วงเวลาที่เลื่อนไปเรื่อยๆ และอาจจะทับกันได้ เช่น ช่วง 3 นาทีล่าสุดที่คำนวณใหม่ทุกนาที เหมาะกับ Top N ใน M นาทีล่าสุด

![[adclick_windows.png]]

### Delivery Guarantees

ก่อนหน้านี้เราพูดถึงเรื่องวิธีการลดโอกาส Event หายไปละ เราจะพูดกันต่อเรื่องของปัญหาที่สำคัญสุดคือทำยังไงให้ Event มันไม่ซ้ำกัน

ใน Kafka มี 3 ระดับให้เลือก คือ At-most-once, At-least-once และ Exactly-once 

ในระบบทั่วๆ ไปจะใช้ At-least-once เพราะถึงต่อให้มีข้อมูลซ้ำนิดหน่อยก็ไม่เป็นไร แต่ระบบนี้มันไม่ใช่แบบนั้น เพราะไม่กี่เปอร์เซ็นต์มันสามารถหมายถึงเงินหลักล้านได้เลย เพราะงั้นจึงต้องเป็น Exactly-once

แต่ก่อนหน้านั้น เราต้องมาดูก่อนว่า Event ซ้ำเกิดได้ยังไง?
![[adclick_duplicated_events.png]]

1. Aggregator ดึง Event ช่วง Offset 100 ถึง 110 จาก Kafka มานับ
2. Aggregator ส่งผลที่นับได้ไปให้ปลายทาง และปลายทางตอบ Ack กลับมาเรียบร้อย
3. Aggregator กำลังจะบอก Kafka ว่าทำถึง 110 แล้ว **แต่ล่มไปก่อน** 
4. พอสร้าง Aggregator ตัวใหม่ขึ้นมาก็เริ่มจาก 100 ใหม่ ทำให้ Event ช่วง Offset 100-110 ถูกนับซ้ำ

ถ้าแก้ด้วยการบันทึก Offset ลงที่เก็บภายนอก เช่น HDFS หรือ S3 ก่อนส่งผล ก็จะกลายเป็นปัญหากลับด้านกันจากเดิม ตามภาพนี้
![[adclick_duplicated_events_offset.png]]

ก็คือถ้า Aggregator ล่มหลังบันทึก Offset แต่ยังไม่ทันส่งผลกลับไปให้ Kafka ก็จะทำให้ Aggregator ตัวใหม่จะเห็นว่าทำถึง 110 แล้ว ทำให้ Event offset 100-110 ไม่ถูกนับเลย

แล้วถ้าสลับไปบันทึก Offset หลังปลายทางตอบ Ack กับ Kafka แทนล่ะ? ก็ยังไม่รอดอยู่ดี เพราะถ้า Aggregator ล่มตรงระหว่างได้ Ack กับบันทึก Offset ก็วนกลับไปนับ Event ซ้ำเหมือนเดิม

สรุปคือ ไม่ว่าจะทำอะไรก่อนหลัง ก็มีจังหวะที่ล่มแล้วพังได้เสมอ 

ทางแก้ที่ถูกคือครอบการส่งผล ACK ให้ Kafka กับการบันทึก Offset ไว้ใน **Distributed Transaction** เดียวกัน (ก็คือมองว่ามันเป็น Transaction เดียวกันนั่นแหละว่าง่ายๆ) มีแค่สำเร็จทั้งคู่หรือล้มทั้งคู่ ซึ่ง Message Queue ตัวที่สองที่ใส่ไว้ตั้งแต่ต้น คือสิ่งที่ทำให้ทำแบบนี้ได้

![[adclick_duplicated_events_distributed_transaction.png]]

### Scale System

#### Message Queue
การ scale Message Queue ต้องดู 3 ส่วน
1. **Producer** (ตัวที่เขียน Event เข้า Queue เช่น Log Watcher) - เพิ่มได้ตามสบาย ไม่มีข้อจำกัดอะไร
2. **Consumer** (ตัวที่ดึง Event ไปใช้ เช่น Aggregation Service) - เวลาเพิ่ม Consumer เข้าไปใน Consumer Group ระบบจะ Rebalance คือกระจาย Partition ให้ Consumer แต่ละตัวใหม่ ตามภาพนี้
![[adclick_scale_message_queue.png]]

แต่ถ้ามี Consumer เยอะๆ การ Rebalance อาจใช้เวลาหลายนาที เพราะงั้นถ้าจะเพิ่ม Consumer ก็ควรทำช่วงที่คนใช้น้อยๆ

3. **Broker** (Server ของ Message Queue เอง เช่น Kafka Broker) - มีเรื่องที่ต้องคิด 3 อย่าง
	- **Hashing Key** - ใช้ `ad_id` เป็น Key ในการแบ่ง Partition เพื่อให้คลิกของโฆษณาตัวเดียวกันอยู่ Partition เดียวกัน Aggregation Service จะได้ดึง Event ของ `ad_id` นั้นได้ครบจาก Partition เดียว
	- **จำนวน Partition** - ควรจองไว้เยอะๆ ตั้งแต่แรก เพราะถ้ามาเพิ่มทีหลัง `ad_id` เดิมอาจถูกย้ายไปอยู่ Partition อื่น ทำให้ Event ของโฆษณาตัวเดียวกันกระจายอยู่หลาย Partition
	- **แยก Topic** - ถ้า Topic เดียวไม่พอ ก็แยกตามภูมิภาค (`topic_asia`, `topic_europe`) หรือประเภทโฆษณา (`topic_web_ads`, `topic_mobile_ads`) ได้ ข้อดีคือได้ Throughput เพิ่ม และ Rebalance เร็วขึ้นเพราะ Consumer ต่อ Topic น้อยลง แต่ก็แลกมาด้วยระบบที่ซับซ้อนและดูแลยากขึ้น

#### Aggregation Service
เราสามารถ scale Aggregation Service ได้ผ่านการเพิ่ม Node ได้ตรงๆ หรือจะเอาระบบไปรันบนระบบจัดการทรัพยากรอย่าง Hadoop YARN ที่เพิ่มกำลังประมวลผลได้ตามต้องการ
![[adclick_scale_aggregation_service.png]]

#### Database
ด้วยความที่เราเลือก Cassandra ตัวมันสามารถ horizontal scale ได้เองด้วยแนวคิดคล้าย [[03 Consistent Hashing|Consistent Hashing]] อย่างการเพิ่ม Node แล้วกระจายข้อมูลใหม่ให้อัตโนมัติ
![[adclick_scale_horizontal_scale_database.png]]

### Hotspot

ในความเป็นจริง มันก็จะมีโฆษณาของบริษัทใหญ่ๆ ที่ทุ่มงบหลักล้าน ซึ่ง Ads เหล่านั้นก็มักจะถูกคลิกเยอะกว่าตัวอื่นๆ อ่ะนะ เพราะงั้น Aggregate Node ที่รับ `ad_id` นั้นจะโหลดหนักอยู่ตัวเดียวได้ 

ทางแก้คือให้ Node นั้นขอ Resources เพิ่มได้ แล้วแบ่ง Event ไปให้ Node ใหม่ช่วยนับ เสร็จแล้วรวมผลกลับมาที่ Node เดิม ตามภาพนี้เลย

![[adclick_hotspot.png]]

### Fault Tolerance

ถ้า Aggregation Node ล่มไป ผลที่นับค้างไว้ใน Memory ก็จะหายหมด จะดึง Event จาก Kafka มานับใหม่ตั้งแต่ต้นก็ทำได้แหละ แต่ช้าเกินไป 

เพราะงั้นเราจะทำ **Snapshot** ไว้เป็นระยะ โดยเก็บทั้ง Offset ล่าสุดและผลที่นับค้างไว้ เช่น Top N ของ M นาทีล่าสุด
![[adclick_snapshot.png]]
![[adclick_snapshot.png]]

พอ Node ล่มก็เปิด Node ใหม่ขึ้นมา โหลด Snapshot ล่าสุด แล้วดึงเฉพาะ Event ที่มาหลัง Snapshot จาก Kafka มานับต่อได้เลยนั่นเอง

### Monitoring

ในระบบนี้ สิ่งที่ควรเฝ้าดูก็จะมี...
1. Latency: แปะ Timestamp ไว้ทุกจุดที่ Event ผ่าน แล้วดูว่าช้าตรงไหน
2. Records Lag ของ Kafka: ถ้าตัวเลขจำนวน Event ค้างรอนับเพิ่มขึ้นอย่างรวดเร็ว แปลว่าต้องเพิ่ม Aggregate Node
3. Aggregate Node Resources เช่น CPU, Disk, Memory

### Reconciliation 

เผื่อลืม reconciliation บ้านๆ คือการ double-check ว่าข้อมูลมีตวามถูกต้องมั้ย เหมือนใน [[15 Payment System#2. Reconciliation|Payment System]] จะมี Third Party ที่ check เรื่องเงินเข้าออก อย่าง PSP (Payment Service Provider) เราก็สามารถนำ Transaction ในระบบของเราไปเทียบกับ  

แต่อย่างระบบนี้มันทำแบบนั้นไม่ได้ สิ่งที่ทำได้คือทุกสิ้นวัน รัน Batch Job นับใหม่จาก Raw Data แล้วเทียบกับผลที่ได้กับ Aggregation Data ว่าตรงกันหรือไม่ 

ถ้าต้องการความแม่นยำมากกว่านี้ก็เทียบถี่ขึ้น เช่นทุกชั่วโมง ผลอาจไม่ตรงกันเป๊ะ เพราะ Event บางตัวมาช้าเกิน Watermark แต่ว่าก็พอช่วยจับความผิดพลาดใหญ่ๆ ได้

![[adclick_final_design.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Ad Click Event Aggregation ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
