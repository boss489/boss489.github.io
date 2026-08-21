---
title: "DynamoDB 실무 학습 Part 4 — 성능, 비용, Capacity, 운영"
permalink: /aws-backend/part-10/05-dynamodb/04-capacity-and-operations/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 4 — 성능, 비용, Capacity, 운영

## 목표

- RCU/WCU 계산의 핵심
- Query와 Get의 Capacity 차이
- FilterExpression의 함정
- 1MB 페이지
- On-Demand vs Provisioned
- Throttling / Retry
- Batch / Transaction
- Streams / TTL
- 운영 지표

---

## 1. RCU

Provisioned Capacity에서 Read Capacity Unit을 사용한다.

Strongly Consistent Read는 읽은 데이터의 크기를 4KB 단위로 계산한다.

Eventually Consistent Read는 같은 데이터에서 절반의 Read Capacity를 소비한다.

핵심:

```text
Strong Read
4KB 단위

Eventually Read
Strong의 절반 수준
```

---

## 2. 매우 중요한 계산 차이: GetItem과 Query

### GetItem / BatchGet의 Item 단위 사고

작은 Item 하나를 Get하면 Item 크기를 4KB 단위로 올림한다.

1KB Item 하나 Strong Read:

```text
→ 1 RCU
```

1KB Item을 개별 Get 100번:

```text
→ 대략 100 RCU
```

### Query

Query는 한 Query가 읽은 Item들의 **총 크기**를 기준으로 4KB 단위 Capacity를 계산한다.

1KB Item 100개가 Query 조건에 의해 읽혔다고 단순 가정:

```text
총 약 100KB
100 / 4 = 25
→ Strong Read 약 25 RCU 수준
```

즉:

> "Item마다 무조건 4KB 올림"이라고 Query까지 동일하게 이해하면 안 된다.

---

## 3. Item 크기가 큰 경우

10KB Item 하나 Strong Read:

```text
10KB
→ 4KB 단위 올림
→ 3 RCU
```

쓰기에서는 1KB 단위가 핵심이다.

---

## 4. WCU

일반적인 Write Capacity는 Item 크기를 1KB 단위로 올림한다.

```text
500B  → 1 WCU
1KB   → 1 WCU
1.5KB → 2 WCU
3KB   → 3 WCU
```

기억:

```text
Read  → 4KB 단위
Write → 1KB 단위
```

---

## 5. Query가 무조건 싸지는 않다

```text
PK = SHOP$21087
```

Partition에 50만 Item이 있고 모두 Query 대상이면 많은 데이터를 읽는다.

더 좋은 조건:

```sql
PK = :pk
AND begins_with(SK, :prefix)
```

핵심:

```text
Query 사용
+
읽는 범위를 KeyCondition으로 좁힘
```

---

## 6. FilterExpression의 함정

```text
KeyCondition
→ 먼저 Item 평가

FilterExpression
→ 그 후 결과 제거
```

예:

```text
읽은 후보 = 100,000
ACTIVE = 100
```

Filter로 100개만 반환되어도 Capacity는 Filter 적용 전 읽은 데이터의 영향을 받는다.


따라서 자주 사용하는 조회라면:

```text
SK
GSI
Sparse GSI
```

로 Access Pattern을 표현할 수 있는지 본다.

---

## 7. Count와 ScannedCount

Filter가 있을 때:

```text
ScannedCount
→ Filter 전에 평가된 수

Count
→ Filter 후 반환 수
```

```text
ScannedCount = 10000
Count = 10
```

이면 Key 설계 개선 신호일 수 있다.

---

## 8. Query/Scan의 1MB 페이지

한 번의 Query/Scan은 무한한 결과를 반환하지 않는다.

읽은 데이터가 약 1MB에 도달하면 페이지가 끊길 수 있다.

중요:

> Filter는 페이지를 읽은 뒤 적용된다.

따라서 첫 페이지에서 Filter 결과가 0건이어도:

```text
LastEvaluatedKey != null
```

일 수 있다.

"결과가 없으니 끝"이라고 판단하면 안 된다.

---

## 9. ReturnConsumedCapacity

학습/튜닝 시 실제 소비 Capacity를 보고 싶다면 API의 `ReturnConsumedCapacity` 옵션을 활용할 수 있다.

목적:

```text
이 Query가 실제로 얼마나 읽었는가?
GSI까지 어느 정도 Capacity를 사용했는가?
```

코드 리뷰에서 추측만 하지 말고 측정하는 습관을 들인다.

---

## 10. On-Demand

장점:

```text
Capacity 사전 설정 불필요
트래픽 변동 대응이 편함
운영 단순
```

적합 후보:

```text
초기 서비스
트래픽 예측 어려움
변동 폭 큼
```

---

## 11. Provisioned

```text
RCU / WCU
```

를 설정한다.

장점:

```text
예측 가능한 워크로드에서 비용/용량 관리
Auto Scaling 활용 가능
```

Capacity 부족 또는 트래픽 집중 시 Throttling을 고려해야 한다.

---

## 12. Throttling

원인 후보:

```text
Provisioned Capacity 부족
특정 Key 집중
갑작스러운 Traffic Spike
대량 Scan/Batch
비효율적인 Access Pattern
```

테이블 전체 사용률만 보지 않는다.

```text
평균은 낮은데 특정 Key만 뜨거운가?
```

를 확인한다.

---

## 13. Retry와 Exponential Backoff

Throttling/일시 오류에는 SDK Retry가 도움을 줄 수 있다.

```text
실패
→ 대기
→ Retry
→ 더 길게 대기
```

Jitter도 여러 Client의 동시 재시도를 완화한다.

하지만:

```text
잘못된 Hot Key 설계
→ Retry로 해결
```

은 근본 해결이 아니다.

---

## 14. BatchGetItem

정확한 Key 여러 개를 알고 있다.

```text
PRODUCT$100 / META
PRODUCT$101 / META
PRODUCT$102 / META
```

→ BatchGet 후보.

Query와 다르다.

```text
BatchGet
→ 정확한 Key 묶음

Query
→ 같은 PK 아래 범위
```

Batch 응답에는 `UnprocessedKeys`가 있을 수 있으므로 재시도를 고려한다.

---

## 15. BatchWriteItem

여러 Put/Delete를 묶는다.

중요:

```text
BatchWrite != Transaction
```

부분적으로 처리되지 않은 `UnprocessedItems`가 생길 수 있다.

원자성 목적이 아니다.

---

## 16. Transaction

다음이 반드시 함께 성공해야 한다.

```text
주문 생성
재고 감소
쿠폰 사용
```

→ `TransactWriteItems` 후보.

Transaction은 일반 Batch보다 강한 의미와 추가 비용/제약이 있으므로 정말 원자성이 필요한 곳에 사용한다.

---

## 17. DynamoDB Streams

변경 이벤트:

```text
DynamoDB Update
    ↓
Stream
    ↓
Lambda / Consumer
    ↓
후속 처리
```

활용:

```text
검색 Index 동기화
복제 데이터 갱신
Audit
이벤트 기반 후속 처리
```

Consumer는 다음을 고려한다.

```text
중복 처리
Idempotency
Retry
실패 처리
순서 요구사항
모니터링
```

---

## 18. TTL

Item에 만료 epoch seconds Attribute를 둔다.

```json
{
  "pk": "IDEMPOTENCY$abc",
  "expiresAt": 1787000000
}
```

활용:

```text
임시 데이터
세션성 데이터
Idempotency 기록
오래된 이벤트
```

주의:

> TTL은 정확한 시각의 예약 삭제가 아니다.

비즈니스적으로 만료 여부가 중요하면 Application에서도 시간을 검사한다.

---

## 19. 큰 Item

목록 API에 필요한 값:

```text
상품명
가격
썸네일
```

Item에는:

```text
상세 HTML
긴 설명
대량 Metadata
```

까지 들어 있어 매우 크다면 Access Pattern에 따라 Summary/Detail Item 분리를 검토한다.

ProjectionExpression은 응답/네트워크를 줄이는 데 도움을 줄 수 있지만, 이미 읽는 Item의 Capacity 비용을 필요한 Attribute 크기만큼 단순 축소하는 기능으로 이해하면 안 된다.

---

## 20. DynamoDB Item 크기

한 Item은 최대 400KB 제한이 있다.

이미지/대형 문서는 일반적으로:

```text
DynamoDB
→ Metadata / S3 Key

S3
→ 실제 Object
```

처럼 역할을 나눈다.

---

## 21. 운영 지표

CloudWatch에서 범주별로 본다.

```text
Consumed Capacity
Throttling
Latency
System Error
User Error
```

문제가 발생하면:

```text
1. Application Latency
2. DynamoDB Latency
3. Retry
4. Throttling
5. Scan 증가
6. Filter 효율
7. Hot Key
8. 최근 배포
```

순으로 추적해볼 수 있다.

---

## 22. 비용 리뷰 체크리스트

```text
Scan이 있는가?
Query 범위가 너무 넓은가?
Filter에 의존하는가?
Item이 너무 큰가?
Strong Read가 필요한가?
GSI가 과도한가?
Write가 여러 GSI를 갱신하는가?
Capacity Mode가 워크로드와 맞는가?
대량 작업이 반복되는가?
```

---

## Part 4 체크

다음 차이를 설명할 수 있어야 한다.

```text
1KB Item 100개를 개별 Get
vs
같은 100개를 한 Query로 읽기
```

그리고 Filter가 반환 건수는 줄여도 읽기 Capacity를 동일한 비율로 줄여주지는 않는 이유를 설명할 수 있어야 한다.

---

# 보강 — GSI 비용, Partition, Streams

## 23. GSI는 공짜 조회 기능이 아니다

Base Table Item을 Write할 때 GSI Key/Projected Attribute가 변경되면 Index도 갱신된다.

```text
Application Write
       ↓
Base Table
       ↓
GSI 갱신
```

따라서 GSI는:

```text
추가 Storage
추가 Write 비용
별도 Read 비용
운영 복잡도
```

를 만든다.

Access Pattern이 있다는 이유만으로 GSI를 무한히 추가하지 않는다.

Transaction 비용도 별도로 계산한다.

```text
TransactGetItems
4KB까지 2 RCU

TransactWriteItems
1KB까지 2 WCU
```

Transaction 안에서 GSI Key 또는 Projection이 바뀌면 Base Table 쓰기뿐 아니라 Index 갱신 비용도 함께 발생한다. 정말 함께 성공해야 하는 항목만 Transaction에 넣는다.

---

## 24. GSI Projection

GSI에 어떤 Attribute를 복사할지 정한다.

### KEYS_ONLY

```text
Index Key
+
Base Table Primary Key
```

위주.

### INCLUDE

필요한 Non-Key Attribute를 지정한다.

예:

```text
productName
price
thumbnail
```

### ALL

Base Item의 모든 Attribute를 Project한다.

판단:

```text
GSI Query 결과만으로 API 응답이 가능한가?
Base Table을 추가 조회해야 하는가?
Index Storage/Write 비용은?
```

---

## 25. Projection 설계 예

목록 API:

```json
{
  "productId": "100",
  "productName": "운동화",
  "price": 10000
}
```

GSI가 Product 목록 조회를 담당한다면:

```text
INCLUDE:
productName
price
```

를 고려할 수 있다.

반대로 상세 설명까지 모두 `ALL`로 넣는 것이 정말 필요한지 검토한다.

---

## 26. Physical Partition 사고

논리적으로는:

```text
Table
 ├ PK=A
 ├ PK=B
 └ PK=C
```

로 보이지만 DynamoDB 내부에서는 데이터를 Physical Partition들에 분산한다.

개념:

```text
Partition Key 값
      ↓
Hash
      ↓
Physical Partition 분산
```

따라서 좋은 Partition Key는 데이터뿐 아니라 요청 트래픽도 적절히 분산시키는 데 도움이 된다.

---

## 27. Adaptive Capacity

DynamoDB는 불균형한 Access Pattern을 어느 정도 흡수하기 위한 Adaptive Capacity 메커니즘을 제공한다.

하지만 다음처럼 이해하면 안 된다.

```text
PK = ALL

모든 Traffic을 하나의 Key에 집중
→ AWS가 알아서 완벽히 해결
```

Adaptive Capacity가 있어도 특정 Key에 극단적으로 트래픽이 몰리는 설계는 병목이 될 수 있다.

---

## 28. Hot Key 진단 질문

```text
어떤 PK가 가장 많이 호출되는가?
특정 대형 고객/Shop만 뜨거운가?
시간 Key 하나로 Write가 몰리는가?
GSI의 PK도 Hot하지 않은가?
Cache로 완화할 수 있는가?
Key 자체를 Sharding해야 하는가?
```

Base Table만 보지 말고 GSI의 Partition Key도 검토한다.

---

## 29. Streams의 전달 의미

Streams는 DynamoDB 변경을 후속 시스템으로 전달할 때 유용하다.

```text
DynamoDB
   ↓
Streams
   ↓
Consumer
   ↓
OpenSearch / Cache / 다른 저장소
```

하지만 이것을:

```text
정확히 한 번만 처리되는 완벽한 메시지 큐
```

라고 단순화하면 위험하다.

Consumer는 재시도와 중복 가능성을 고려해 Idempotent하게 설계하는 것이 중요하다.

---

## 30. Streams와 순서

DynamoDB Streams는 변경 레코드의 순서를 이해할 때 **Item/Shard 단위의 처리 특성**을 고려해야 한다.

전체 테이블의 모든 변경이 하나의 글로벌 순서로 처리된다고 가정하면 안 된다.

비즈니스가 전역 순서를 요구한다면 별도의 설계가 필요할 수 있다.

---

## 31. Idempotent Consumer

예:

```text
ORDER$1000 상태 변경
→ Stream
→ Search Index 갱신
```

Consumer가 같은 이벤트를 다시 처리해도:

```text
결과가 망가지지 않도록
```

설계한다.

예:

```text
최종 상태를 Upsert
Event ID 처리 기록
Version 비교
```

---

## 32. Streams 실패 처리

운영에서는 다음을 설계한다.

```text
Retry
실패 이벤트 격리
DLQ 또는 실패 목적지
Alarm
재처리 절차
Consumer Lag
```

이벤트 연결만 하고 실패 경로를 만들지 않으면 운영 시 복구가 어렵다.

---

## 33. GSI/Partition/Streams 최종 질문

```text
GSI를 추가하면 Write 비용은?
Projection은 무엇인가?
GSI PK가 Hot하지 않은가?
Adaptive Capacity에 과도하게 기대하지 않는가?
Stream Consumer는 중복에 안전한가?
실패 이벤트를 다시 처리할 수 있는가?
```
