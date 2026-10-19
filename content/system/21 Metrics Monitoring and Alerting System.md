---
title: 21 Metrics Monitoring and Alerting System
tags:
  - systemdesign
  - system
  - metrics_monitoring_and_alerting_system
  - time_series_database
  - service_discovery
  - message_queue
---
วันนี้เราจะมาทำระบบ Metrics Monitoring and Alerting กัน เป็นระบบที่คอย Check Health ของ Server ทั้งหมดในบริษัท เช่น CPU, Memory, Request ต่อวินาที แล้วเอามาแสดงเป็นกราฟ ละก็ถ้ามีอะไรผิดปกติก็ส่ง Alert ไปบอกคนดูแลทันที ประมาณพวก Datadog, Prometheus, Grafana อ่ะ จะเป็นไง มาดูกัน...

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: ระบบนี้เราทำให้ใครใช้เหรอครับ? ใช้กันเองภายในบริษัทใหญ่ๆ แบบ Facebook หรือ Google หรือว่าเป็น SaaS ขายคนอื่นแบบ Datadog ครับ? 
A1: ใช้กันเองภายในบริษัท

Q2: แล้วเราจะเก็บ Metrics อะไรบ้างเหรอครับ?
A2: เอาเป็น Operational Metrics ละกัน เช่น CPU Load, Memory Usage, Disk Space หรือระดับสูงขึ้นมาหน่อยก็ได้ เช่น จำนวน Request ต่อวินาทีของ Service หรือจำนวน Server ที่ยังรันอยู่ใน Pool แต่ Business Metrics อย่างยอดขายไม่ต้องนะ

Q3: แล้ว Infrastructure ที่ต้อง Monitor ใหญ่ขนาดไหนเหรอครับ? 
A3: มี User 100 million DAU, มี Server Pool 1,000 Pool แต่ละ Pool มี 100 เครื่อง

Q4: ต้องเก็บข้อมูลไว้นานแค่ไหนเหรอครับ? 
A4: 1 ปีละกัน

Q5: แล้วผมลดความละเอียดของข้อมูลเก่าๆ ได้มั้ยครับ? 
A5: ได้นะ ข้อมูลใหม่เก็บแบบละเอียดไว้ 7 วัน หลังจากนั้นยุบเหลือความละเอียด 1 นาทีเก็บไว้ 30 วัน เกิน 30 วันก็ยุบเหลือความละเอียด 1 ชั่วโมงได้เลย

Q6: แล้ว Alert ต้องส่งผ่านช่องทางไหนบ้างเหรอครับ? 
A6: Email, โทรศัพท์, PagerDuty หรือ Webhook ก็ได้

Q7: ต้องเก็บ Log อย่าง Error Log หรือ Access Log ด้วยมั้ยครับ? 
A7: ไม่ต้อง

Q8: แล้วต้องทำ Distributed Tracing ด้วยมั้ยครับ? 
A8: ไม่ต้องเหมือนกัน

> [!Tip] Log Monitoring กับ Distributed Tracing คืออะไร?
> 
> - **Log Monitoring** คือ การเก็บและค้นหา Log ข้อความ เช่น Error Log
> - **Distributed Tracing** คือ การไล่เช็คตาม Request ว่าวิ่งผ่าน Service ไหนบ้าง ช้าที่จุดไหน
> 
> ทั้ง 2 อย่างเป็นคนละเรื่องกับ Metrics ที่เป็นตัวเลขตามเวลา เลยตัดออกจาก Scope

### Functional Requirement

- ระบบเก็บ Metrics ของ Server ได้หลายประเภท เช่น CPU Usage, Request Count, Memory Usage, จำนวน Message ใน Queue
- ระบบส่ง Alert ได้เมื่อ Metrics ผิดปกติ ผ่าน Email, โทรศัพท์, PagerDuty และ Webhook
- ระบบแสดง Metrics เป็นกราฟบน Dashboard ได้
- ระบบเก็บข้อมูลได้ 1 ปี ตาม Retention Policy
    - 7 วันแรก: เก็บแบบละเอียด (Raw)
    - 8-30 วัน: ยุบเหลือความละเอียด 1 นาที
    - นอกเหนือจากนั้น ถึง 1 ปี: ยุบเหลือความละเอียด 1 ชั่วโมง

### Non-Functional Requirement

- Scalability - รองรับ Metrics และ Alert ที่เพิ่มขึ้นเรื่อยๆ ได้
- Low Latency - Query สำหรับ Dashboard และ Alert ต้องตอบได้เร็ว
- Reliability - ห้ามพลาด Alert สำคัญ
- Flexibility - ต้องเปลี่ยนเครื่องมือใหม่เข้ามาได้ง่าย และไม่กระทบ Pipeline เดิมที่ใช้งานอยู่

---

## 2. Sketch ภาพระบบคร่าวๆ

ด้วยความไม่คุ้นชินว่า Metrics Monitoring and Alert System คืออะไร เราจะมาดู Fundamental เล็กๆ น้อยๆ ก่อนละกัน

![[metric_maa_fundamental.png]]

Metric Monitoring and Alert System หลักๆ จะประกอบด้วย 5 ส่วน
1. Data Collection: gather metric data เข้าสู่ระบบ
2. Data Transmission: แปลง metric data ที่รวบรวมมาให้อยู่ใน format ที่ system ของเราสามารถนำไปใช้งานต่อได้
3. Data Storage: เก็บ metric data ที่เข้าสู่ระบบมา
4. Alerting: วิเคราะห์ metric data และแจ้งเตือน dev เมื่อเกิดความผิดปกติ
5. Visualization: แสดง metric  data ออกมาเป็น Chart ต่างๆ เพื่อวิเคราะห์ และตอบคำถามเชิง business

### Data Model

Metrics จะถูกเก็บในรูปแบบ **Time Series** โดย Time Series แต่ละเส้นจะมี label บอกว่า Metric อะไร ของ Device อะไร เช่น ถ้าเราอยากรู้ว่า **CPU Load ของ Server `i631` ที่อยู่ใน Production ตอน 20:00 เป็นเท่าไหร่** ก็ไปดูที่เส้น CPU Load i631 ละก็เล็งเวลา 20:00 ไว้ ก็จะเห็นว่าเป็น 0.29 นั่นเอง

![[metric_maa_cpu_load_sample.png]]

แต่ละจุดในเส้นจะเก็บเป็นข้อมูลที่มีลักษณะแบบนี้...

| Field     | Value                               |
| --------- | ----------------------------------- |
| Metric    | cpu.load                            |
| Label     | host:i631, env:prod, region:us-east |
| Timestamp | 1613707265                          |
| Value     | 0.29                                |

ถ้าคำถามซับซ้อนขึ้นอีกหน่อย เช่น **CPU Load เฉลี่ยของ Web Server ทุกตัวใน Region us-west ช่วง 10 นาทีล่าสุดเป็นเท่าไหร่** ระบบก็จะไปดึงทุกบรรทัดที่ชื่อ Metric เป็น `CPU.load` และมี Label `region` = `us-west` ออกมา หน้าตาที่ได้ก็จะเป็นประมาณนี้

```txt
CPU.load host=webserver01,region=us-west 1613707265 50
CPU.load host=webserver01,region=us-west 1613707265 62
CPU.load host=webserver02,region=us-west 1613707265 43
CPU.load host=webserver02,region=us-west 1613707265 53
...
CPU.load host=webserver01,region=us-west 1613707265 76
CPU.load host=webserver01,region=us-west 1613707265 83
```

แล้วก็เอาตัวเลขท้ายบรรทัดมาเฉลี่ยกัน ซึ่งรูปแบบบรรทัดแบบนี้เรียกว่า **Line Protocol** เป็น Format ที่เครื่องมือ Monitoring หลายตัวใช้ เช่น Prometheus และ OpenTSDB

### Data Access Pattern

![[metric_maa_data_access_pattern.png]]

จากภาพ แกน Y แต่ละแถวคือ Time Series หนึ่งเส้น ส่วนแกน X คือเวลา จะเห็นว่า...
- **Write หนัก** เพราะมี Metrics เป็น 10 ล้านตัวที่ถูกเขียนเข้ามาพร้อมๆ กันตลอด และหลายตัวก็ถูกเก็บถี่มาก
- **Read มาเป็นพักๆ** เพราะคนที่อ่านคือ Visualization กับ Alerting ซึ่งจะอ่านเยอะเป็นช่วงๆ เช่น ตอนมีคนเปิด Dashboard หลายคนพร้อมกัน หรือตอนที่ Alert Rule หลายตัวรันพร้อมกัน

เพราะงั้นที่เก็บข้อมูลของเราต้องรับ Write หนักได้ตลอด และรับ Read ที่พุ่งขึ้นมาเป็นพักๆ ได้ด้วย

### Data Storage

Data Storage ถือเป็นหัวใจของระบบนี้เลย ทีนี้เราจะเลือกอะไรดี เริ่มจากตัวที่คุ้นเคยก่อนละกัน (อ่านพื้นฐานเพิ่มเติมได้ใน [[01 Database|Database]])

**Relational Database อย่าง MySQL** - ในทางทฤษฎีก็เก็บ Time Series ได้นะ **ปัญหาคือ**...
- Query ที่ใช้บ่อยกับ Time Series เช่น Moving Average ในช่วงเวลาที่เลื่อนไปเรื่อยๆ เขียนเป็น SQL ได้ยาก และอ่านยากมาก
- ถ้าจะค้นหาตาม Label ได้เร็ว ก็ต้องสร้าง Index ให้ทุก Label
- รับ Write หนักตลอดเวลาได้ไม่ดี ต่อให้จะ config หนักแค่ไหนก็อาจจะยังไม่พอที่จะ scale

ถัดมาก็เป็น... **NoSQL อย่าง Cassandra หรือ Bigtable** - ทำได้เหมือนกัน แต่ต้องเข้าใจไส้ในของ Database ตัวนั้นลึกมาก ถึงจะออกแบบ Schema ให้เก็บและ Query Time Series ได้ดี ในเมื่อมี Database ที่ทำมาเพื่องานนี้โดยเฉพาะอยู่แล้ว ก็ไม่คุ้มที่จะทำเองอ่ะนะ 

-> แล้วมันคืออะไรอ่ะที่บอกว่าเป็น Database ที่ทำมาเพื่องานนี้โดยเฉพาะ --> Time-Series Database นั่นเอง

**Time-Series Database** - เป็น Database ที่ออกแบบมาเพื่อ Time Series โดยตรง ทำให้ใช้ Server น้อยกว่ามากในการรับข้อมูลปริมาณเท่ากัน มี Query Language ของตัวเองที่เขียนง่ายกว่า SQL และบางตัวมีฟีเจอร์จัดการ Retention กับการยุบข้อมูลมาให้เลย ตัวอย่างของ Time-Series Database ก็เช่น...
- **OpenTSDB** - เป็น Distributed แต่สร้างบน Hadoop และ HBase ทำให้ต้องดูแล Cluster เพิ่มอีก
- **MetricsDB** ของ Twitter และ **Timestream** ของ Amazon
- **InfluxDB** และ **Prometheus** - 2 ตัวที่นิยมที่สุด ทั้งคู่ใช้ In-memory Cache คู่กับ On-disk Storage ทนทานและเร็ว อย่าง InfluxDB ที่มี 8 Cores กับ RAM 32 GB ก็รับ Write ได้มากกว่า 250,000 ครั้งต่อวินาทีแล้ว

เพราะงั้นเราจะเลือกใช้ **Time-Series Database** อย่าง InfluxDB ในการเก็บข้อมูล

>[!Tip] Label ควรมี Cardinality ต่ำ 
>Time-Series Database ที่ดีจะสร้าง Index บน Label ให้ เพื่อให้ค้นหาและ Aggregate ตาม Label ได้เร็ว สำคัญกับงาน Visualization 
>
>แต่ Label แต่ละตัวควรมี **Cardinality ต่ำ** ก็คือ...มีค่าที่เป็นไปได้ไม่เยอะ เช่น `region` ที่มีแค่ไม่กี่ค่า เพราะถ้าเอาค่าที่ไม่ซ้ำกันเลยอย่าง `user_id` มาเป็น Label จำนวน Time Series จะบวมจนระบบรับไม่ไหว

### High-Level Design

ประกอบร่างทั้งหมดเข้าด้วยกันก็จะได้ประมาณนี้
![[metric_maa_high-level_design.png]]

1. **Metrics Source** - แหล่งที่มาของ Metrics เช่น Application Server, SQL Database, Message Queue
2. **Metrics Collector** - เก็บ Metrics จาก Metrics Source แล้วเขียนลง Time-Series Database
3. **Time-Series Database** - เก็บ Metrics เป็น Time Series มี Query Interface สำหรับวิเคราะห์ข้อมูลปริมาณมาก และมี Index บน Label ให้ค้นหาได้เร็ว
4. **Query Service** - ตัวกลางที่ช่วยให้ Query ข้อมูลจาก Time-Series Database ได้ง่ายขึ้น ถ้าเลือก Database ดีๆ ส่วนนี้จะบางมาก หรือใช้ Query Interface ของ Database แทนไปเลยก็ได้
5. **Alerting System** - ยิง Query ไปเช็ค Metrics แล้วส่ง Alert ไปตามช่องทางต่างๆ
6. **Visualization System** - ยิง Query ไปดึง Metrics มาแสดงเป็นกราฟ

---

## 3. เจาะลึกจุดที่สำคัญๆ

### Metric Collection

การเก็บ Metrics จะมี 2 แบบให้เลือก คือ **Pull** กับ **Push** ซึ่งเป็นเรื่องที่เถียงกันตลอดว่าแบบไหนดีกว่า และก็ไม่มีคำตอบที่ถูกที่สุด ไปดูทีละแบบกัน

#### 1. Pull Model

Pull คือการที่ Metrics Collector วิ่งไปดึง Metrics จาก Server แต่ละเครื่องเองเป็นระยะๆ ผ่าน HTTP

![[metric_maa_pull_model.png]]

**ปัญหาคือ** Metrics Collector ต้องรู้ให้ได้ว่ามี Server อะไรบ้างที่ต้องไปดึง วิธีที่ง่ายสุดคือเขียนรายชื่อ DNS หรือ IP ของทุก Server ไว้ในไฟล์ Config แต่ในบริษัทใหญ่ที่ Server ถูกเพิ่มหรือลดอยู่ตลอด ไฟล์นี้จะดูแลยากมาก และเสี่ยงที่จะลืมเพิ่ม Server ใหม่ ทำให้ไม่ได้เก็บ Metrics ของมัน

เพราะงั้นเราจะใช้ **Service Discovery** อย่าง etcd หรือ Zookeeper มาช่วย คือให้ Server แต่ละตัวไปลงทะเบียนไว้ที่ Service Discovery เอง พร้อม Config ว่าต้องเก็บ Metrics เมื่อไหร่และที่ไหน

เพราะงั้น Flow ของ Pull Model ก็จะเป็นประมาณนี้...
1. Metrics Collector ดึงข้อมูลของ Server ที่ต้องเก็บจาก Service Discovery เช่น IP Address, ความถี่ในการดึง, Timeout และ Retry
2. Metrics Collector ยิง HTTP ไปดึง Metrics ที่ Endpoint ที่ตกลงกันไว้ เช่น `/metrics` ซึ่งฝั่ง Server ต้องลง Client Library เพื่อเปิด Endpoint นี้
3. Metrics Collector ลงทะเบียนรอรับการแจ้งเตือนจาก Service Discovery ไว้ด้วย พอมี Server เพิ่มหรือลด Service Discovery ก็จะแจ้งมาเอง หรือจะให้ Metrics Collector คอยถามเป็นระยะๆ ก็ได้

ทีนี้พอเราใช้ Service Discovery ก็จะมีปัญหาใหม่ตามมา คือ Server ที่ลงทะเบียนใน Service Discovery มีเยอะมากจน Metrics Collector ตัวเดียวดึงไม่ไหวแน่ๆ ก็ต้องมีหลายตัว 

**ปัญหาคือ** ถ้าใช้ Collector หลายตัวไปดึง Server เครื่องเดียวกัน ข้อมูลก็จะซ้ำ ต้องมีวิธีแบ่งงานกัน

เราสามารถใช้ **Consistent Hashing** ได้ (อ่านเพิ่มเติมได้ใน [[03 Consistent Hashing|Consistent Hashing]]) โดยวาง Collector แต่ละตัวไว้บน Hash Ring แล้วเอาชื่อ Server ที่ลงทะเบียนใน Service Discovery มา Hash ลงบน Ring เหมือนกัน Server ตกช่วงไหนก็ให้ Collector ที่ดูแลช่วงนั้นเป็นคนดึง ทำให้ Server หนึ่งเครื่องมี Collector ดูแลแค่ตัวเดียวเสมอ

![[metric_maa_consistent_hashing.png]]

อย่างในภาพ มี Collector 4 ตัว กับ Server 6 เครื่อง Collector 2 จะดูแลแค่ Server 1 กับ Server 5 เท่านั้น

#### 2. Push Model

Push คือกลับด้านกัน ให้ Server แต่ละเครื่องเป็นฝ่ายส่ง Metrics มาหา Metrics Collector เอง
![[metric_maa_push_model.png]]

โดยปกติจะมีการลง **Collection Agent** ไว้ในทุก Server ที่ต้อง Monitor (ก็คือลงในฝั่ง Metric Source) ซึ่งเป็นโปรแกรมที่รันค้างไว้ คอยเก็บ Metrics จาก Service ในเครื่อง แล้ว Push ไปหา Metrics Collector เป็นระยะๆ -> เราสามารถเขียน Agent ให้ aggregate ข้อมูลในเบื้องต้นก่อนส่งมาให้เราได้ด้วย เช่น Counter ที่นับทุกวินาที ก็รวมเป็นยอดต่อนาทีแล้วค่อยส่งทีเดียว ช่วยลดปริมาณข้อมูลที่ส่งไปได้เยอะ

แล้วถ้า Traffic เยอะจน Metrics Collector ปฏิเสธข้อมูลล่ะ? 

Collection Agent ก็สามารถเก็บข้อมูลพักไว้ในเครื่องก่อน (เช่น เขียนลง Disk) แล้วค่อยส่งใหม่ทีหลังได้

**ปัญหาคือ** ถ้า Server อยู่ใน Auto-scaling Group ที่ถูกเปิดปิดบ่อยๆ ข้อมูลที่พักไว้ก็จะหายไปพร้อมกับ Server ที่ถูกปิด

เพราะงั้นทางที่ดีคือทำให้ Metrics Collector ไม่ตามหลังตั้งแต่แรก ด้วยการทำเป็น Auto-scaling Cluster ที่มี [[02 Load Balancer|Load Balancer]] อยู่ข้างหน้า และให้ Scale ขึ้นลงตาม CPU Load ของ Collector

![[metric_maa_push_load_balancer.png]]

แล้วเราควรเลือกแบบไหนดี?

| Topic                      | Pull                                                                          | Push                                                                          |
| -------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Debug ง่าย                 | เปิด `/metrics` ดูค่าได้ตลอดเวลา จากเครื่องตัวเองก็ได้                        | ถ้า Collector ไม่ได้รับ Metrics ก็ไม่รู้ว่าเป็นที่ Server หรือที่ Network     |
| Health Check               | ถ้า Server ไม่ตอบตอนไปดึง ก็รู้ทันทีว่า Server ล่ม                            | ถ้าไม่ได้รับ Metrics ก็อาจเป็นแค่ปัญหา Network ก็ได้ ไม่ได้สื่อถึง Server ล่ม |
| Job อายุสั้น               | Batch Job บางตัวจบก่อนจะถูกดึง แก้ได้ด้วยการเพิ่ม Push Gateway                | Job ส่งเองก่อนจบได้เลย                                                        |
| Firewall / Network ซับซ้อน | Collector ต้องเข้าถึง Endpoint ของทุก Server ได้ ยุ่งยากถ้ามีหลาย Data Center | ถ้ามี Load Balancer กับ Auto-scaling ก็รับข้อมูลจากที่ไหนก็ได้                |
| ความน่าเชื่อถือของข้อมูล   | Server ที่ดึงถูกกำหนดไว้ใน Config ล่วงหน้า ข้อมูลจึงมาจาก Server จริงแน่นอน   | ใครก็ Push เข้ามาได้ ต้องแก้ด้วย Whitelist หรือการยืนยันตัวตน                 |

ในชีวิตจริง บริษัทใหญ่ๆ มักต้องรองรับทั้ง 2 แบบ โดยเฉพาะเมื่อมี Serverless เข้ามา ที่เราไม่สามารถลง Collection Agent ไว้ในเครื่องได้ตั้งแต่แรก

### Scale the Metrics Transmission Pipeline
 
ไม่ว่าจะเลือก Pull หรือ Push ตัว Metrics Collector ก็จะเป็น Cluster ที่ Auto-scale ได้อยู่แล้ว 

**ปัญหาคือ** ถ้า Time-Series Database ล่มหรือรับไม่ทัน Metrics ที่ Collector ส่งมาก็จะหายไปเลย
เพราะงั้นเราจะเพิ่ม Queue อย่าง Kafka คั่นไว้ตรงกลาง เพื่อเก็บ Metrics เอาไว้ และทำให้สามารถทำงานแบบ Asynchronous ได้ (อ่านเพิ่มเติมได้ใน [[09 Message Queue|Message Queue]])
![[metric_maa_add_kafka.png]] 
1. Metrics Collector ส่ง Metrics เข้า Kafka
2. Consumer หรือ Stream Processing อย่าง Apache Storm, Flink, Spark ดึง Metrics จาก Kafka มาประมวลผล
3. Consumer เขียน Metrics ลง Time-Series Database

ข้อดีของการใส่ Kafka คือ...
- **Kafka เชื่อถือได้และ Scale ได้ดี** เป็น Messaging Platform ที่ใช้กันแพร่หลาย
- **แยกการเก็บกับการประมวลผลออกจากกัน** แต่ละฝั่ง Scale แยกกันได้
- **ข้อมูลไม่หายตอน Database ล่ม** เพราะ Kafka เก็บข้อมูลค้างไว้ ทำให้สามารถรอ Database กลับมาแล้วค่อยเขียนต่อได้

#### Scale ผ่าน Kafka
 
เราใช้กลไก Partition ของ Kafka ช่วย Scale ได้หลายทาง
- **ตั้งจำนวน Partition ตาม Throughput ที่ต้องการ**
- **แบ่ง Partition ตามชื่อ Metric** ทำให้ Metric เดียวกันอยู่ Partition เดียวกัน Consumer ก็ Aggregate ตามชื่อ Metric ได้ง่าย
- **แบ่งย่อยลงไปอีกตาม Label**
- **จัดกลุ่มและจัดลำดับความสำคัญของ Metrics** ให้ Metrics สำคัญถูกประมวลผลก่อน

![[metric_maa_scale_kafka.png]]

#### Aggregate ที่ไหนดี?
 
Metrics สามารถเกิดการ Aggregate (รวมข้อมูล เช่น หาผลรวมหรือค่าเฉลี่ย) ได้ 3 จุด
1. **ที่ Collection Agent (ฝั่ง Client)** - ทำได้แค่ Logic ง่ายๆ เช่น รวม Counter ทุก 1 นาทีก่อนส่ง
2. **ที่ Ingestion Pipeline (ก่อนเขียนลง Storage)** - ต้องใช้ Stream Processing อย่าง Flink ข้อดีคือ Write ลง Database น้อยลงมาก เพราะเขียนแค่ผลที่คำนวณแล้ว แต่ต้องจัดการ Event ที่มาช้า และเสียความละเอียดกับความยืดหยุ่นไป เพราะไม่ได้เก็บ Raw Data ไว้
3. **ที่ฝั่ง Query (หลังเขียนลง Storage)** - เก็บ Raw Data ไว้ทั้งหมด แล้วค่อย Aggregate ตอน Query ไม่เสียข้อมูลเลย แต่ Query จะช้ากว่า เพราะต้องคำนวณจากข้อมูลทั้งหมดทุกครั้ง

### Query Service
 
Query Service คือ Cluster ของ Query Server ที่คอยรับ Request จาก Visualization กับ Alerting แล้วไป Query Time-Series Database ให้ 

ข้อดีคือมันแยก Database ออกจากฝั่ง Client ทำให้ถ้าวันหนึ่งอยากเปลี่ยน Database หรือเปลี่ยนระบบ Visualization / Alerting ก็ทำได้โดยไม่กระทบกัน

#### Cache Layer
เพื่อลด Load ของ Time-Series Database และทำให้ Query เร็วขึ้น เราสามารถเพิ่ม Cache ไว้เก็บผลลัพธ์ของ Query ได้ (อ่านเพิ่มเติมได้ใน [[06 Cache|Cache]]) เช่น ถ้ามีคนเปิด Dashboard เดียวกัน 50 คน ก็ Query Database แค่ครั้งเดียว อีก 49 ครั้งดึงจาก Cache
![[metric_maa_cache_layer.png]]
 
#### Time-Series Database Query Language
Time-Series Database ดังๆ อย่าง Prometheus กับ InfluxDB ไม่ได้ใช้ SQL แต่มี Query Language ของตัวเอง เหตุผลหลักคือ SQL เขียน Query สำหรับ Time Series ได้ยากมาก

อย่างการหา Moving Average ถ้าเขียนด้วย SQL จะหน้าตาประมาณนี้
```sql
SELECT id, temp, AVG(temp) OVER (
	PARTITION BY group_nr ORDER BY time_read
) AS rolling_avg
FROM (
	SELECT id, temp, time_read, interval_group, 
		id - row_number() OVER (
			PARTITION BY interval_group ORDER BY time_read
		) AS group_nr
	FROM (
	    SELECT id, time_read, temp,
	    "epoch"::timestamp + "900 seconds"::interval * (
		    EXTRACT(epoch FROM time_read)::int4 / 900
		) AS interval_group FROM readings
	) t1
) t2
ORDER BY time_read;
```
 
แต่ถ้าเขียนด้วย **Flux** ซึ่งเป็นภาษาของ InfluxDB จะสั้นและอ่านง่ายกว่ามาก
```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### Storage Layer
 
#### เลือก Time-Series Database ให้ดี
จากงานวิจัยของ Facebook พบว่าอย่างน้อย **85%** ของ Query ทั้งหมด ถามหาข้อมูลที่เกิดขึ้นใน **26 ชั่วโมงล่าสุด** เพราะงั้นถ้าเลือก Time-Series Database ที่ออกแบบมาให้ข้อมูลล่าสุดเข้าถึงได้เร็ว (เช่น เก็บข้อมูลใหม่ไว้ใน Memory) ก็จะช่วยให้ระบบโดยรวมเร็วขึ้นมาก
 
#### ประหยัดพื้นที่
ข้อมูลที่ต้องเก็บมีเยอะมาก เพราะงั้นต้องหาทางลดขนาดลง ซึ่งทำได้ 3 วิธี
 
**1. Data Encoding และ Compression** ![[metric_maa_data_encoding.png]]
 
ถ้าเก็บ Timestamp แบบเต็มๆ แต่ละตัวต้องใช้ 32 bits แต่จะเห็นว่า 1610087371 กับ 1610087381 ห่างกันแค่ 10 วินาที ซึ่งเก็บได้ในแค่ 4 bits เพราะงั้นแทนที่จะเก็บค่าเต็มทุกตัว เราเก็บค่าแรกแบบเต็ม แล้วตัวถัดๆ ไปเก็บแค่ส่วนต่าง (Delta) จากตัวก่อนหน้า
 
```txt
เก็บแบบเต็ม: 1610087371, 1610087381, 1610087391, 1610087400, 1610087411
เก็บแบบ Delta: 1610087371, 10, 10, 9, 11
```
 
**2. Downsampling**
Downsampling คือการแปลงข้อมูลความละเอียดสูงให้เป็นความละเอียดต่ำ เพื่อลดพื้นที่ ซึ่งตรงกับ Retention Policy ที่คุยกันไว้ตอนแรก
- 7 วัน: ไม่ยุบ
- 30 วัน: ยุบเหลือความละเอียด 1 นาที
- 1 ปี: ยุบเหลือความละเอียด 1 ชั่วโมง

ลองดูตัวอย่างการยุบข้อมูลที่เก็บทุก 10 วินาที ให้เหลือทุก 30 วินาที โดยใช้ค่าเฉลี่ย...

ข้อมูลเดิม (ทุก 10 วินาที)

| metric | timestamp            | hostname | metric_value |
| ------ | -------------------- | -------- | ------------ |
| cpu    | 2021-10-24T19:00:00Z | host-a   | 10           |
| cpu    | 2021-10-24T19:00:10Z | host-a   | 16           |
| cpu    | 2021-10-24T19:00:20Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 30           |
| cpu    | 2021-10-24T19:00:40Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:50Z | host-a   | 30           |
 
หลังยุบ (ทุก 30 วินาที)

| metric | timestamp            | hostname | metric_value (avg)      |
| ------ | -------------------- | -------- | ----------------------- |
| cpu    | 2021-10-24T19:00:00Z | host-a   | 15.33 (= (10+16+20)/3)  |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 26.67 (= (30+20+30)/3)  |
 
จาก 6 แถวเหลือ 2 แถว ประหยัดไป 3 เท่า ซึ่งถ้ายุบจากทุก 10 วินาทีเป็นทุก 1 ชั่วโมง ก็จะประหยัดได้ถึง 360 เท่าเลย
 
**3. Cold Storage**
Cold Storage คือที่เก็บข้อมูลที่แทบไม่มีใครเรียกใช้แล้ว ราคาถูกกว่า Storage ปกติมาก ข้อมูลเก่าๆ ที่ถูกยุบแล้วก็ย้ายไปไว้ตรงนี้ได้
 
### Alerting System
 
ทีนี้มาดูระบบ Alert กันว่าทำงานยังไง
![[metric_maa_alerting_system.png]]
 
1. **โหลด Config File เข้า Cache** - Alert Rule จะถูกเขียนเป็นไฟล์ Config เก็บไว้บน Disk เช่น Rule นี้บอกว่า ถ้า Instance ไหนติดต่อไม่ได้นานเกิน 5 นาที ให้ส่ง Alert ระดับ `page`
	```yaml
	- name: instance_down
	  rules:
	  # Alert for any instance that is unreachable for >5 minutes.
	  - alert: instance_down
	    expr: up == 0
	    for: 5m
	    labels:
	      severity: page
	```
 
2. **Alert Manager ดึง Config จาก Cache**
3. **Alert Manager ยิง Query ไปที่ Query Service ตามรอบที่กำหนดใน Rule** ถ้าค่าที่ได้เกิน Threshold ก็สร้าง Alert Event ขึ้นมา โดย Alert Manager มีหน้าที่เพิ่มเติมอีก
	- **Filter, Merge และ Dedupe Alert** - เช่น ถ้า Instance 1 มี Disk Usage เกิน 90% ติดกัน 3 ครั้งในเวลาสั้นๆ ก็รวมเป็น Alert เดียว ไม่ต้องส่งไป 3 รอบ
	![[metric_maa_merge_alert.png]]

	- **Access Control** - จำกัดให้เฉพาะคนที่มีสิทธิ์เท่านั้นที่จัดการ Alert ได้ ป้องกันความผิดพลาดจากคนและเพื่อความปลอดภัย
	- **Retry** - เช็คสถานะของ Alert และทำให้มั่นใจว่า Notification ถูกส่งออกไปอย่างน้อย 1 ครั้ง
 
4. **Alert Store** - เป็น Key-Value Database อย่าง Cassandra ที่เก็บสถานะของทุก Alert ได้แก่ `inactive`, `pending`, `firing` และ `resolved` เพื่อให้มั่นใจว่า Notification ถูกส่งอย่างน้อย 1 ครั้ง
5. **Alert ที่ผ่านเงื่อนไขถูกส่งเข้า Kafka**
6. **Alert Consumer ดึง Alert Event จาก Kafka**
7. **Alert Consumer ส่ง Notification ไปตามช่องทางต่างๆ** เช่น Email, Text Message, PagerDuty หรือ HTTP Endpoint (อ่านเพิ่มเติมเรื่องการส่ง Notification ได้ใน [[06 Notification System|Notification System]] และเรื่อง HTTP Endpoint ได้ใน [[15 Webhook|Webhook]])

>[!Tip] ทำ Alerting System เอง หรือซื้อดี?
>ในตลาดมีระบบ Alert ระดับ Industry ให้ใช้เยอะมาก ส่วนใหญ่ต่อกับ Time-Series Database ดังๆ และช่องทางอย่าง Email, PagerDuty ได้อยู่แล้ว ในชีวิตจริงแทบไม่มีเหตุผลที่จะทำเอง แต่ถ้าสัมภาษณ์ตำแหน่ง Senior ก็ควรอธิบายได้ว่าทำไมถึงเลือกแบบนั้น


### Visualization System
 
Visualization สร้างอยู่บนชั้นของข้อมูล ทำหน้าที่แสดง Metrics บน Dashboard ในช่วงเวลาต่างๆ และแสดง Alert บน Alert Dashboard เช่น จำนวน Request ของแต่ละ Server, Memory / CPU, เวลาโหลดหน้าเว็บ, Traffic เข้าออก และจำนวน Login
![[metric_,maa_grafana_ui.png]]
 
การทำ Visualization ให้ดีนั้นยากมาก เพราะงั้นเหตุผลที่จะใช้ของสำเร็จรูปจึงหนักแน่นมาก อย่าง **Grafana** ที่ต่อกับ Time-Series Database ดังๆ ได้เกือบทุกตัว
 
พอรวมทุกอย่างเข้าด้วยกัน Design สุดท้ายก็จะเป็นประมาณนี้
![[metric_maa_final_design.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Metrics Monitoring and Alerting System ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---