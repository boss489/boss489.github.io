---
title: "DynamoDB 실무 학습 Part 7 — 종합 설계, 코드 리뷰, 실전 문제"
permalink: /aws-backend/part-10/05-dynamodb/07-practice/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 7 — 종합 설계, 코드 리뷰, 실전 문제

## 목표

최종 Part에서는 설명을 읽는 것보다 직접 판단한다.

```text
요구사항
→ Access Pattern
→ PK/SK
→ GSI
→ Java Repository
→ 성능
→ 비용
→ 동시성
→ 운영
```

---

# 실전 문제 1 — Dshop 전시

## 요구사항

```text
AP1. Shop 전체 전시 구조
AP2. Template별 Corner 목록
AP3. Corner 한 건
AP4. Corner별 Product
AP5. Product 기준 Corner 역조회
AP6. ACTIVE Corner 조회
```

먼저 직접 설계한 뒤 아래 모범 사고를 본다.

---

## 모범 사고

Shop 구조:

```text
PK = SHOP$21087

SK
META
TEMPLATE$886
TEMPLATE$886$CORNER$2981
TEMPLATE$886$CORNER$2982
```

AP2:

```text
PK = SHOP$21087
begins_with(
    SK,
    TEMPLATE$886$CORNER$
)
```

AP3:

```text
GetItem
PK = SHOP$21087
SK = TEMPLATE$886$CORNER$2982
```

Corner 상품이 독립적으로 크다면:

```text
PK = CORNER$2982
SK = PRODUCT$100
```

처럼 별도 Item Collection 후보.

Product 역조회:

```text
GSI1PK = PRODUCT$100
GSI1SK = CORNER$2982
```

ACTIVE만 필요하면 Sparse GSI 후보.

---

# 실전 문제 2 — 주문/재고

## 요구사항

```text
AP1. orderId로 주문
AP2. customerId로 최근 주문
AP3. FAILED 주문
AP4. 상품별 주문 역조회
AP5. 주문 생성 + 재고 감소는 함께 성공
```

---

## 모범 사고

원본:

```text
PK = ORDER$1000
SK = META
```

고객 주문:

```text
GSI1PK = CUSTOMER$100
GSI1SK = 20260821T090000$ORDER$1000
```

FAILED:

```text
GSI2PK = FAILED
GSI2SK = 20260821T090000$ORDER$1000
```

실패 주문에만 GSI2 Key를 넣으면 Sparse GSI가 된다.

상품 관계:

```text
PK = ORDER$1000
SK = PRODUCT$500

GSI3PK = PRODUCT$500
GSI3SK = ORDER$1000
```

주문 + 재고:

```text
TransactWriteItems
```

검토.

---

# 실전 문제 3 — 대규모 이벤트

## 요구사항

```text
초당 매우 많은 Event
Device별 최근 Event 조회
최근 7일 조회
오래된 데이터 자동 정리
```

잘못된 설계:

```text
PK = TODAY
```

하루 Write가 한 Key로 집중될 수 있다.

후보:

```text
PK = DEVICE$100$20260821
SK = 09:00:00.123$EVENT$abc
```

최근 7일:

```text
7개 날짜 Partition Query
```

오래된 데이터:

```text
TTL
```

Device 하나 자체가 Hot하면 추가 Sharding을 검토한다.

---

# 코드 리뷰 문제 1

```java
DynamoDBScanExpression expression =
        new DynamoDBScanExpression()
                .withFilterExpression(
                        "shopNo = :shopNo"
                );
```

질문:

```text
shopNo 조회가 온라인 핵심 Access Pattern인가?
그렇다면 왜 PK/GSI가 아닌가?
Scan 데이터 규모는?
```

---

# 코드 리뷰 문제 2

```java
.withKeyConditionExpression("hk = :hk")
.withFilterExpression("#status = :active")
```

Partition:

```text
500,000 Items
ACTIVE = 20
```

리뷰:

```text
Filter로 20개만 반환해도
많은 데이터를 읽을 수 있다.

ACTIVE가 중요하면
Key/GSI/Sparse GSI 검토.
```

---

# 코드 리뷰 문제 3

```java
public List<Dshop> find(
        DynamoDBQueryExpression<Dshop> expression
) {
    return mapper.query(
            Dshop.class,
            expression
    );
}
```

문제:

```text
호출자가 DynamoDB 문법을 알아야 함
Repository 추상화가 약함
SDK Migration 영향 확대
```

개선:

```java
findCornersByTemplate(
    shopNo,
    templateSeq
)
```

---

# 코드 리뷰 문제 4

```java
query:
hk = :hk AND sk = :sk
```

두 값을 정확히 알고 있다.

질문:

```text
결과가 최대 한 건인가?
그렇다면 load/GetItem이 의도를 더 잘 표현하는가?
```

---

# 코드 리뷰 문제 5 — 동시성

```java
Product p = repository.find(id);

p.setStock(
    p.getStock() - 1
);

repository.save(p);
```

문제:

```text
Lost Update
재고 음수
동시 주문
```

후보:

```text
Atomic Update
Condition Expression
Transaction
Optimistic Lock
```

---

# 코드 리뷰 문제 6 — Pagination

```java
List<Dshop> result = mapper.query(...);

if (result.isEmpty()) {
    return Collections.emptyList();
}
```

Filter가 있고 첫 DynamoDB Page의 결과가 0건이라고 해서 전체 Query가 끝난 것은 아닐 수 있다.

확인:

```text
LastEvaluatedKey
1MB Page
Filter 적용 순서
SDK Lazy Pagination
```

---

# 최종 설계 절차

새 요구사항을 받으면 다음 순서로 간다.

```text
1. Access Pattern 작성
2. 각 조회에서 이미 알고 있는 값 확인
3. 한 건 / 목록 구분
4. PK 후보
5. SK 계층/정렬
6. Get vs Query
7. 역방향 → GSI
8. 일부 상태 → Sparse GSI
9. Many-to-Many → 관계 Item/GSI
10. 데이터 증가량
11. Hot Key
12. Item Size
13. Consistency
14. Pagination
15. Write 동시성
16. Transaction 필요성
17. 비용/Capacity
18. Repository API
19. Integration Test
20. 운영 지표
```

---

# 최종 면접/코드리뷰 질문 15개

1. PK와 SK를 둘 다 정확히 알면 결과는 몇 건인가?
2. GetItem과 Query는 언제 구분하는가?
3. Query와 Scan의 핵심 차이는?
4. FilterExpression이 Read 비용을 크게 줄이지 않는 이유는?
5. GSI와 LSI의 차이는?
6. GSI에서 Strong Read가 가능한가?
7. Sparse GSI란?
8. Hot Partition은 왜 생기는가?
9. Write Sharding의 Trade-off는?
10. BatchWrite와 Transaction의 차이는?
11. Lost Update를 DynamoDB에서 어떻게 막을 수 있는가?
12. TTL은 정확한 예약 삭제인가?
13. Query의 Pagination은 무엇으로 이어가는가?
14. Repository에 SDK 타입을 숨기는 이유는?
15. DynamoDB 모델링에서 Entity보다 Access Pattern을 먼저 보는 이유는?

---

# 최종 답안 핵심

```text
DynamoDB를 잘한다
≠ SDK 메서드를 많이 외운다

DynamoDB를 잘한다
=
Access Pattern을 파악하고
PK/SK/GSI로 표현하며
비용·동시성·운영까지 판단한다
```

처음 봤던 코드:

```java
.withKeyConditionExpression(
        "hk = :hk and sk = :sk"
)
```

를 이제는 다음 관점으로 읽어야 한다.

```text
왜 hk가 DSHOP$21087인가?
왜 sk가 TMPL_SEQ$886+2982인가?
886과 2982는 각각 무엇인가?
이 Key 순서가 어떤 Query를 지원하는가?
정확한 Key인데 왜 Query인가?
Prefix Query도 존재하는가?
Partition은 얼마나 커지는가?
GSI는 어떤 역조회 때문에 존재하는가?
Filter로 낭비하고 있지는 않은가?
Hot Key 가능성은?
Repository가 이 구현을 감추는가?
```

이 질문들이 자연스럽게 나오면 실무 DynamoDB 코드를 읽고 설계할 기반이 갖춰진 것이다.

---

# Final Challenge — 대규모 커머스 주문 시스템

## 조건

```text
회원: 1,000만
상품: 100만
일 주문: 500만
```

요구사항:

```text
AP1. orderId로 주문 상세 조회
AP2. 회원의 최근 주문 20개
AP3. 회원의 최근 30일 주문
AP4. productId가 포함된 주문 역조회
AP5. FAILED 주문만 운영자가 시간순 조회
AP6. 주문 생성과 재고 차감은 함께 성공
AP7. 주문 상태는 허용된 이전 상태에서만 변경
AP8. 주문 변경을 검색 시스템에 반영
AP9. API는 Cursor Pagination 사용
```

아래 답을 보기 전에 직접 설계한다.

```text
Base PK =
Base SK =

GSI1 =
GSI2 =
GSI3 =

GetItem =
Query =
Sparse GSI =
Transaction =
Conditional Update =
Streams =
Pagination =
```

---

# 모범 설계 사고

## 1. 주문 원본

orderId 직접 조회가 매우 중요하다.

```text
PK = ORDER$1000
SK = META
```

AP1:

```text
GetItem
```

---

## 2. 회원 최근 주문

주문 Item:

```text
GSI1PK = CUSTOMER$100
GSI1SK = 20260821T090000$ORDER$1000
```

AP2:

```text
GSI1 Query
scanIndexForward = false
limit = 20
```

AP3 최근 30일:

```text
GSI1PK = CUSTOMER$100

GSI1SK BETWEEN
20260722...
AND
20260821...
```

시간 문자열 포맷은 lexical order가 시간 순과 일치하도록 만든다.

---

## 3. 상품별 주문 역조회

Order Product 관계 Item:

```text
PK = ORDER$1000
SK = PRODUCT$500
```

그리고:

```text
GSI2PK = PRODUCT$500
GSI2SK = 20260821T090000$ORDER$1000
```

AP4:

```text
GSI2 Query
```

이것은 Inverted Index 역할을 한다.

---

## 4. FAILED 주문

FAILED 상태 Item에만:

```text
GSI3PK = FAILED_ORDER
GSI3SK = 20260821T090000$ORDER$1000
```

를 넣는다.

성공/완료 주문에는 `GSI3PK`를 제거한다.

→ Sparse GSI.

운영 조회가 글로벌하게 매우 많다면 `FAILED_ORDER` 하나가 Hot Key가 되지 않는지도 검토한다.

예:

```text
FAILED_ORDER$20260821
```

처럼 시간 Bucket을 두는 선택지도 있다.

---

## 5. 주문 + 재고

다음은 함께 성공해야 한다.

```text
Order Put
Stock Update
```

후보:

```text
TransactWriteItems
```

재고 Update에는:

```text
stock >= :quantity
```

Condition을 함께 사용한다.

클라이언트 재시도까지 고려하면 재고 조건만으로는 중복 주문을 막을 수 없다. 요청마다 `idempotencyKey`를 받고 같은 Transaction에 최초 처리 기록을 넣는다.

```text
ConditionCheck / Put
attribute_not_exists(IDEMPOTENCY$<key>)

Order Put
Stock Update (stock >= quantity)
```

같은 Key로 재시도했을 때 조건이 실패하면 재고를 다시 차감하지 말고, 기존 처리 결과를 조회해 같은 응답을 반환한다.

---

## 6. 주문 상태 전이

예:

```text
PAID → SHIPPING
```

아무 상태에서나 SHIPPING으로 바뀌면 안 된다.

```text
UpdateExpression
SET #status = :shipping

ConditionExpression
#status = :paid
```

Condition 실패는:

```text
이미 취소됨
이미 배송 처리됨
```

등의 도메인 상태로 해석한다.

---

## 7. 검색 시스템 동기화

```text
DynamoDB
   ↓
Streams
   ↓
Consumer
   ↓
Search
```

Consumer 요구사항:

```text
Idempotent
Retry
실패 격리
모니터링
재처리
```

검색 시스템의 반영은 주문 Transaction과 같은 순간에 반드시 완료되어야 하는지, Eventually Consistent여도 되는지 구분한다.


---

## 8. Cursor Pagination

회원 최근 주문:

```text
GSI1 Query
```

응답의:

```text
LastEvaluatedKey
```

를 Cursor로 인코딩한다.

다음 요청:

```text
ExclusiveStartKey
```

로 사용한다.

```text
page=100
```

형식의 임의 Offset Jump를 기본 설계로 삼지 않는다.

---

# 잘못된 설계 예

## 잘못된 예 1

```text
PK = ORDER_DATE$20260821
```

일 주문 500만 건이 하나의 날짜 Key로 집중될 수 있다.

---

## 잘못된 예 2

```text
전체 주문 Scan
Filter customerId = 100
```

회원 주문은 핵심 Access Pattern이므로 GSI 등의 Key로 표현해야 한다.

---

## 잘못된 예 3

```text
전체 FAILED 조회를
status Filter로 처리
```

실패 주문이 매우 적고 자주 조회된다면 Sparse GSI가 더 자연스럽다.

---

## 잘못된 예 4

```java
stock = getStock();
save(stock - 1);
```

동시 주문에서 Lost Update/재고 문제가 생길 수 있다.

---

## 잘못된 예 5

```text
DynamoDB 저장 성공
→ Search API 동기 호출
→ Search 실패
→ 주문 API 전체 실패
```

검색 반영이 비즈니스 Transaction과 같은 원자성을 정말 요구하는지 검토한다. 많은 경우 Stream 기반 비동기 동기화가 더 적합할 수 있다.

---

# Final Challenge 통과 기준

다음 연결이 자연스러우면 된다.

```text
orderId
→ GetItem

customerId + time
→ GSI Query

productId → order
→ Inverted Index

FAILED only
→ Sparse GSI

재고
→ Atomic/Conditional Update

주문 + 재고
→ Transaction

검색 동기화
→ Streams

목록 다음 페이지
→ LastEvaluatedKey / Cursor

대규모 Traffic
→ Hot Key / Partition 분산 검토
```

여기까지 이해하면 단순 DynamoDB API 사용을 넘어 실무 데이터 모델과 코드 리뷰를 할 수 있는 단계다.
