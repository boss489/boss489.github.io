---
title: "DynamoDB 실무 학습 Part 5 — 고급 데이터 모델링 패턴"
permalink: /aws-backend/part-10/05-dynamodb/05-advanced-modeling/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 5 — 고급 데이터 모델링 패턴

## 목표

- Sparse GSI
- Inverted Index
- Adjacency List
- Write Sharding
- Time-Series
- GSI Overloading
- 실제 요구사항 → Key 설계

---

## 1. Sparse GSI

일부 Item만 Index에 넣는다.

```text
정상 주문
PK = ORDER$100

실패 주문
PK = ORDER$101
GSI1PK = FAILED_ORDER
GSI1SK = 20260821$ORDER$101
```

`GSI1PK`가 없는 정상 주문은 GSI에 나타나지 않는다.

---

## 2. Sparse GSI vs Filter

전체:

```text
1,000,000 Orders
```

실패:

```text
100 Orders
```

Filter:

```text
많은 Item 평가
→ FAILED만 반환
```

Sparse GSI:

```text
FAILED Item만 Index
→ GSI Query
```

실패 주문 조회가 핵심 Access Pattern이면 Sparse GSI가 훨씬 자연스러울 수 있다.


---

## 3. Inverted Index

기본:

```text
CUSTOMER$100
 ├ ORDER$1
 └ ORDER$2
```

새 요구:

```text
PRODUCT$500이 포함된 주문을 찾고 싶다.
```

관계 Item에:

```text
GSI1PK = PRODUCT$500
GSI1SK = ORDER$1
```

를 두면 역방향 조회가 가능하다.

---

## 4. Adjacency List

Many-to-Many:

```text
USER ↔ GROUP
```

```text
PK          SK
USER$100    GROUP$A
USER$100    GROUP$B
USER$200    GROUP$A
```

반대 조회:

```text
GSI_PK = GROUP$A
GSI_SK = USER$100
```

관계 자체에도:

```text
role
joinedAt
status
```

를 저장할 수 있다.

---

## 5. Write Sharding

Hot Write:

```text
PK = EVENT$20260821
```

모든 Event가 한 Key로 몰린다.

분산:

```text
EVENT$20260821#0
EVENT$20260821#1
...
EVENT$20260821#9
```

쓰기:

```java
int shard = Math.floorMod(
        eventId.hashCode(),
        10
);
```

Trade-off:

```text
쓰기 분산 ↑
전체 읽기 복잡도 ↑
```

전체 이벤트를 조회하거나 집계해야 하면 `#0`부터 `#9`까지 모든 shard를 Query한 뒤 애플리케이션에서 합쳐야 한다. 그래서 shard 수는 시작 시 고정하고, 같은 `eventId`는 항상 같은 shard로 가도록 결정한다.

```text
쓰기 API
→ eventId hash로 shard 1개 선택

전체 조회 API
→ 모든 shard 병렬 Query + 결과 병합
```

---

## 6. Time-Series

Device 이벤트:

```text
PK = DEVICE$100$20260821
SK = 09:00:00$EVENT$001
```

다음 날:

```text
PK = DEVICE$100$20260822
```

시간 범위를 Partition에 포함하면 무한 성장과 Hot Partition을 줄이는 데 도움이 될 수 있다.

여러 날짜 조회는 여러 Query가 필요하다.

---

## 7. 시간 Sort Key

정렬 가능한 문자열:

```text
2026-08-21T09:00:00
2026-08-21T10:00:00
```

날짜를 문자열 SK로 쓸 때 lexical order가 시간 순서와 일치하도록 포맷한다.

---

## 8. GSI Overloading

같은 GSI를 여러 Entity Type에서 재사용한다.

```text
Order:
GSI1PK = CUSTOMER$100
GSI1SK = ORDER$20260821$1000

Product:
GSI1PK = CATEGORY$SHOES
GSI1SK = PRODUCT$500
```

장점:

```text
Index 재사용
```

단점:

```text
복잡도 증가
문서화 필수
```

---

## 9. 실전 설계 1 — 주문

요구:

```text
AP1. customerId로 최근 주문
AP2. orderId로 주문 직접 조회
AP3. FAILED 주문 조회
AP4. productId로 주문 역조회
```

한 가지 후보:

주문 원본:

```text
PK = ORDER$1000
SK = META
```

고객 최근 주문 GSI:

```text
GSI1PK = CUSTOMER$100
GSI1SK = 20260821T090000$ORDER$1000
```

실패 주문 Sparse GSI:

```text
GSI2PK = FAILED
GSI2SK = 20260821T090000$ORDER$1000
```

Product 관계 Item:

```text
PK = ORDER$1000
SK = PRODUCT$500

GSI3PK = PRODUCT$500
GSI3SK = ORDER$1000
```

정답은 하나가 아니다. Access Pattern과 데이터 규모를 만족하는지 평가한다.

---

## 10. 실전 설계 2 — Dshop

요구:

```text
AP1. Shop 전체
AP2. Template 886의 Corner
AP3. Corner 2982 직접 조회
AP4. Product 기준 Corner 역조회
```

후보:

```text
PK = SHOP$21087
SK = TEMPLATE$886
SK = TEMPLATE$886$CORNER$2982
```

AP2:

```text
begins_with(SK, "TEMPLATE$886$CORNER$")
```

AP3:

```text
GetItem
PK = SHOP$21087
SK = TEMPLATE$886$CORNER$2982
```

AP4:

```text
GSI_PK = PRODUCT$100
GSI_SK = CORNER$2982
```

---

## 11. 실전 설계 3 — 대량 이벤트

요구:

```text
하루 수천만 이벤트 Write
Device별 최근 이벤트
날짜별 전체 분석은 Offline
```

잘못된 후보:

```text
PK = 20260821
```

모든 Write가 하루 Key에 집중된다.

개선 후보:

```text
PK = DEVICE$100$20260821
SK = TIMESTAMP$EVENT_ID
```

또는 Device 자체도 Hot하다면 추가 Sharding을 검토한다.

Offline 분석은 DynamoDB 온라인 Access Pattern과 별도 데이터 파이프라인/S3 등을 고려할 수 있다.

---

## 12. 고급 패턴 선택 순서

```text
일부 Item만 빠르게?
→ Sparse GSI

반대 방향 조회?
→ Inverted Index

Many-to-Many?
→ Adjacency List

특정 Key Write 폭증?
→ Write Sharding

시간에 따라 무한 증가?
→ Time Partition

GSI 수가 많아짐?
→ Overloading 가능성 검토
```

---

## 13. 과도한 Single Table Design

다음은 경고 신호다.

```text
Key Prefix가 수십 개
GSI가 무엇을 의미하는지 팀원이 설명 못함
한 변경이 너무 많은 Entity에 영향
독립 도메인이 억지로 한 Table에 묶임
```

Single Table은 목표가 아니라 최적화 수단이다.

---

## Part 5 체크

새 요구사항을 보면 먼저 다음을 적는다.

```text
무엇을 알고 조회하는가?
한 건인가 목록인가?
어떤 순서로 정렬하는가?
반대 방향 조회가 있는가?
일부 상태만 조회하는가?
어느 Key에 트래픽이 몰리는가?
```
