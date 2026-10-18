---
title: 19 Real-time Gaming Leaderboard
tags:
  - system
  - systemdesign
  - real-time_gaming_leaderboard
  - redis
  - database_partition
  - scatter_gather
---
วันนี้เราจะมาทำระบบ Leaderboard System ที่แสดงผลแบบ Real-Time กัน หรือก็คือเห็นคะแนนขยับกันต่อหน้าต่อตา ลำดับเปลี่ยนแปลงกันตลอดนั่นแหละ จะเป็นไง ไปดูกัน...

![[leaderboard_overview.png]]

---

## 1. ถามรายละเอียดของ Scope งาน

Q1: เริ่มต้นก่อนว่า... เราคิดคะแนนกันยังไงอ่ะ?
A1: ก็เวลาเราชนะ เราก็ได้แต้มเนอะ ก็เอาแต้มที่ได้ไปรวมกับแต้มที่คนๆ นั้นมี

Q2: Users ทุกคนอยู่บน Leaderboard นี้เลยมั้ย
A2: อยู่หมดๆ

Q3: แล้ว...มีช่วงเวลากำกับใน Leaderboard มั้ยนะ?
A3: มีนะ กำหนดให้ Leaderboard จะ reset ทุกๆ เดือนละกัน

Q4: ผมสามารถ focus แค่การแสดง top 10 ให้ Users ทุกคนเห็นได้มั้ยครับ หรือผมควร focus อะไรเพิ่มเติมมั้ยครับ?
A4: โอเคนะ ขอแค่ top10 ที่ตำแหน่งของแต่ละคนถูกต้องก็น่าจะโอเคละ อ่อ แต่ว่าลำดับของแต่ละคนก็ต้องถูกด้วยนะ ถึงจะไม่ได้เอามาแสดงให้คนอื่นเห็นก็เถอะ

Q5: Users มีเยอะแค่ไหนเหรอครับ ต่อ 1 tournament?
A5: ราวๆ 5 million DAU ละก็ 25 million MAU อ่ะๆ

Q6: แล้วจำนวน Matches มีเยอะแค่ไหนเหรอครับ ต่อ 1 tournament?
A6: แต่ละคนจะได้เล่นคนละ 10 Matches ต่อวันอ่ะ

Q7: สมมติว่ามี Users 2 คนที่แต้มเท่ากัน เราจะ tie-break ยังไงเหรอครับ?
A7: ถือว่า rank ของทั้ง 2 คนเท่ากันได้เลย

---

## 2. Sketch ภาพระบบคร่าวๆ

### API Design

1. **POST** /v1/scores - API สำหรับ update แต้มของ user โดยจะมีแค่ Game Service เรียกเท่านั้น Client เรียกตรงๆ ไม่ได้

| Field   | Description            |
| ------- | ---------------------- |
| user_id | user ที่จะได้แต้มเพิ่ม |
| points  | จำนวนแต้มที่จะได้เพิ่ม |

2. **GET** /v1/scores - API สำหรับดึงรายชื่อ และคะแนนของ Top10

```swagger
{
	"data": [
		{
			"user_id": "user1",
			"user_name": "alice",
			"rank": 1,
			"score": 12345
		},
		{
			"user_id": "user2",
			"user_name": "bob",
			"rank": 2,
			"score": 12000
		}
	]
}
```

3. **GET** /v1/scores/:user_id - API สำหรับดึงคะแนนและ Rank ของ User คนนั้น

```swagger
{
	"data": {
		"user_id": "user5",
		"score": 1000,
		"rank": 6
	}
}
```

### High-Level Architecture

ภาพรวมระบบก็จะเป็นประมาณนี้
![[leaderboard_high-level_architecture.png]]

1. เวลาที่มี User ชนะ Match ก็แปลว่า User คนนั้นจะได้แต้ม ระบบก็จะเรียกใช้งาน Game Service
2. Game Service ที่ได้รับการ trigger เมื่อ Match จบก็จะ trigger Leaderboard Service เพื่อ update คะแนนของ User ที่ชนะ
3. Leaderboard Service update score ของ User ที่ชนะ
4. ระบบจะแสดง Leaderboard ที่แสดง Top10 ของ Tournament นั้น รวมถึง Rank ของ User ผ่านการเรียกใช้งาน Leaderboard Service

แล้วทำไมเราต้องมี Game Service นะ? - หลักๆ ก็ประมาณว่า...
1. จัดการเรื่อง Match
2. Trigger การ update คะแนน ซึ่งดีกว่าการให้ Client trigger Leaderboard Service โดยตรง เพราะจะกลายเป็นว่าการ Set Score จะอยู่ที่ Client ทำให้เสี่ยงต่อ Man-in-the-middle ได้

![[leaderboard_game_triggers_event.png]]

และเราสามารถเพิ่ม Queue ระหว่าง Game Service กับ Service อื่นๆ ที่ Game Service trigger เป็นเหมือนกับ Pub/Sub อ่ะ เช่น
- Leaderboard Service เพื่อ update Leaderboard
- Analytic Service เพื่อดูสถิติการแข่งขัน
- Notification Service เพื่อสร้าง Notification
แต่ว่าก็แล้วแต่การใช้งานนะ

### Data Model

หัวใจของระบบนี้อยู่ที่ Leaderboard ว่าจะเก็บคะแนนไว้ที่ไหนให้เรียงลำดับได้เร็ว มันก็มีหลายแบบที่ทำได้นะ เดี๋ยวเราจะเริ่มไล่จากทางที่ง่ายที่สุดก่อนละกัน
#### Relational Database

ถ้า User ไม่เยอะ ก็ใช้ตาราง `leaderboard` ที่มีแค่ `user_id` กับ `score` ได้เลย เมื่อมี User ที่ชนะ Match ก็ `UPDATE` บวกแต้มเข้าไป 
```sql
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```
```sql
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

หรือถ้าอยากรู้ลำดับก็ `ORDER BY score DESC` ได้เลย
```sql
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

![[leaderboard_relational_database_schema.png]]

![[leaderboard_user_wins_match.png]]

แต่ปัญหาคือ...การจะรู้ว่า User คนหนึ่งอยู่อันดับเท่าไหร่ ต้องเรียงทุกคนทั้งตารางก่อน พอมีหลายล้านแถวและคะแนนเปลี่ยนตลอดเวลา 

เพราะงั้น Query นี้จะใช้เวลาหลายวินาที ซึ่งไม่ใช่ Real-time แล้ว จะใส่ `LIMIT 10` ช่วยก็ได้แค่ตอนดึง Top 10 แต่ตอนหา Rank ของคนที่ไม่ได้อยู่ข้างบนก็ยังต้องไล่ทั้งตารางอยู่ดี เพราะงั้นก็ต้องลองใช้ตัวอื่นๆ ดู

#### Redis Sorted Set

Redis มี Data Type ชื่อ **Sorted Set** ที่เหมาะกับ Leaderboard พอดี เป็น Set ที่สมาชิกแต่ละตัวผูกกับคะแนน และ **เรียงลำดับให้เองอัตโนมัติทุกครั้งที่เพิ่มหรือแก้คะแนน** ไม่ต้องมาสั่งเรียงทีหลัง

![[leaderboard_redis_sort_set.png]]

การที่ Redis Sorted Set สามารถทำแบบนี้ได้ เป็นเพราะข้างใน Redis Sorted Set ใช้โครงสร้าง 2 อย่างที่ทำงานคู่กัน คือ Hash Table ไว้จับคู่ User กับคะแนน และ **Skip List** ไว้เรียงคะแนน

>[!Tip] What is Skip List?
>Skip List คือ  Linked List ที่เรียงลำดับแล้ว แต่ซ้อน Index ไว้ด้านบนหลายชั้น แต่ละชั้นกระโดดข้ามทีละหลาย Node ทำให้หาข้อมูลได้เร็วคล้าย Binary Search แทนที่จะไล่ทีละตัว ทั้งการเพิ่ม แก้ และค้นหาจึงใช้เวลาแค่ O(log n) - ก็ตามตัวแหละ List ที่เราสามารถ Skip ได้หลายค่าที่ไม่เกี่ยวข้องกับที่เราจะหา
>
>![[leaderboard_skip_list.png]]
>
>อย่างเช่นในภาพนี้ การจะหาค่า 45 ใน List
>- วิธีปกติคือ ไล่จาก Node แรกไปยัง Node สุดท้าย
>- พอเรามี Level 1 Index ก็ทำให้เรา Skip ไปได้ครึ่งนึง
>- Level 2 Index ก็ทำให้เรา Skip ไปได้อีกครึ่งนึง

คำสั่งที่ใช้ในระบบนี้ จะมี 4 ตัว

1. User 'mary1934' ชนะได้ n แต้ม
```redis
ZINCRBY leaderboard_feb_2021 n 'mary1934'
```

2. ดึง Top 10 เรียงตาม Score คือ Index 0 ถึง Index 9
```redis
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

3. ดึง Rank ของ User 'mary1934'
```redis
ZREVRANK leaderboard_feb_2021 'mary1934'
```

4. ดึง User ที่อยู่ใน Rank 357-365
```redis
ZREVRANGE leaderboard_feb_2021 357 365
```

แต่การใช้ Redis ก็ยังต้องมีเรื่องที่ต้องระวังอยู่ คือ Redis อาจจะล่มได้ เพราะมีการใช้งานจาก User จำนวนมาก แต่ปกติจะตั้ง Read Replica ไว้ ถ้าตัวหลักล่มก็เลื่อน Replica ขึ้นมาแทน 

นอกจากนี้เราก็ยังสามารถเก็บข้อมูลใน MySQL ควบคู่ไว้ด้วย 2 ตาราง คือ 
- users เก็บข้อมูล User 
- points เก็บทุกครั้งที่ User ชนะพร้อม Timestamp 

ทำให้ถ้า Redis เจ๊ง ก็ไล่ข้อมูลใน point แล้วยิง ZINCRBY ใหม่ทีละรายการ สร้าง Leaderboard คืนมาได้

---

## 3. เจาะลึกส่วนที่สำคัญๆ

### Use Cloud Provider or Not Use

เราสามารถใช้ Cloud Provider หรือไม่ใช้ก็ได้ กรณีที่ไม่ใช้ก็จะมี Architecture ประมาณนี้...
![[leaderboard_no_cloud_provider.png]]

ก็คือเรามี User Profile กับ User Point Schema อยู่ใน MySQL ละก็เราจะมี Redis Cache ที่จัดการเรื่อง Leaderboard และ User Profile ที่เป็น Top10

แต่ว่าถ้าเราจะใช้ Cloud Provider ก็สามารถทำได้เหมือนกัน ผ่านการทำ Lambda Function แล้วใช้ Lambda Service ของ AWS

| APIs                    | Lambda function            |
| ----------------------- | -------------------------- |
| GET /v1/scores<br>      | LeaderboardFetchTop10      |
| GET /v1/scores/:user_id | LeaderboardFetchPlayerRank |
| POST /v1/scores         | LeaderboardUpdateScore     |

หน้าตา Architecture ก็จะเป็นประมาณนี้...
![[leaderboard_with_cloud_provider_update_point.png]]
![[leaderboard_with_cloud_provider_get_leaderboard.png]]

### Redis Scaling

ถ้าวันหนึ่ง User โตขึ้น 100 เท่า...

สมมติว่า ID ของ User แต่ละคนเป็น string ความยาว 24 และคะแนนเป็นเลข 16 bits หรือ 2 bytes แปลว่า User แต่ละคนจะใช้เนื้อที่ 26 bytes เพราะงั้นการที่มี User ราวๆ 25 million MAU ก็แปลว่า Storage จะต้องใช้เนื้อที่ราวๆ 650 MB เลย

เพราะงั้นใน Case นี้ Storage ก็จะต้องใช้เนื้อที่ถึง 65 GB ต่อเดือนเลย ซึ่งเครื่องเดียวมันรับไม่ไหวหรอก เพราะงั้นก็ต้องมีการแบ่งข้อมูลกันเก็บ ซึ่งเราจะแบ่งได้ 2 แบบ คือ...

1. Fixed Partitioning - แบ่งตามช่วงคะแนน เช่น Shard แรกเก็บคนที่มี 1-100 แต้ม Shard ถัดไปเก็บ 101-200 ไปเรื่อยๆ

ข้อดีของแบบนี้คือ...
- การดึง Top10 ทำได้ง่าย ก็ไปเช็ค Set สุดท้าย ละดึงมา 10 คนที่เยอะสุดก็พอ
- การหาอันดับของ User ก็ทำได้ง่ายเหมือนกัน ผ่านการเช็คใน Shard ของตัวเองว่าเป็นลำดับที่เท่าไหร่ แล้วบวกจำนวนคนทั้งหมดใน Shard ที่คะแนนสูงกว่า
- แต่การ insert หรือ update ข้อมูลจะจุกจิกหน่อย คืออาจจะต้องมีการย้าย Shard อ่ะนะ

![[leaderboard_fixed_partition.png]]

2. Hashed Partitioning - ใช้ Redis Cluster ที่แบ่งข้อมูลให้อัตโนมัติด้วยการ Hash user_id แล้วกระจายข้อมูลลงหลาย Node

ข้อดีของแบบนี้คือ...
- การ insert หรือ update ข้อมูลจะทำได้ง่าย เพราะข้อมูลอยู่ใน Shard เดิม
- การดึง Top10 และการหาอันดับของ User จะยากกว่า เพราะข้อมูลไม่ได้เรียงตามคะแนน

![[leaderboard_hashed_partition.png]]

### Alternative Data Model: NoSQL

จะว่าอย่างงั้นอย่างงี้ก็เถอะ งานนี้ NoSQL ก็ทำได้นะ อย่างการใช้ DynamoDB โดยใช้ `game_name#{year-month}` เป็น Partition Key และใช้ Score เป็น Sort Key

![[leaderboard_dynamodb_partitionkey_sortkey.png]]

ปัญหาคือ...ข้อมูลทั้งเดือนจะไปกองอยู่ใน Partition เดียว เพราะมี Partition Key ที่เหมือนกัน ทำให้ Partition นั้นกลายเป็น Hot Partition ที่มี Load หนักอยู่ตัวเดียว 

ซึ่งเรามีทางแก้คือต่อเลข Partition ท้าย Key เป็น `game_name#{year-month}#p{user_id % N}` เพื่อกระจายข้อมูลไปหลาย Partition เทคนิคนี้เรียกว่า `Write Sharding`

![[leaderboard_dynamodb_write_sharding.png]]

แต่การทำแบบนี้ก็ต้องแลกมาด้วยการที่ตอนดึง Top 10 ต้องทำ `Scatter-Gather` คือ Scatter ข้อมูลหลายมีหลายๆ Partition และ Gather ข้อมูล Top10 จากแต่ละ Partition มา แล้วตัดเหลือแค่ Final Top10 
ซึ่งทำให้หา Rank ตรงๆ ไม่ได้ แต่บอกเป็น Percentile ได้ เช่น "คุณอยู่ Top 10%" ซึ่งในชีวิตจริงอาจดีกว่าบอกว่าอยู่อันดับที่ 1,200,001 ด้วยซ้ำ

![[leaderboard_dynamodb_scatter_gather.png]]

---

เพียงเท่านี้ Design System Interview เกี่ยวกับ Real-time Gaming Leaderboard ก็น่าจะผ่านพ้นภัยได้ด้วยดีละ...

-> BadLuckZ

---
