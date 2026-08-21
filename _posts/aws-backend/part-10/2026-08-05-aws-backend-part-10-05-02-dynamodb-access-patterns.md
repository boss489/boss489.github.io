---
title: "DynamoDB 실무 학습 Part 2 — Access Pattern과 Single Table Design"
permalink: /aws-backend/part-10/05-dynamodb/02-access-patterns/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 2 — Access Pattern과 Single Table Design

## 목표

- Access Pattern부터 설계하는 이유
- Single Table Design
- 비정규화와 데이터 중복
- Composite Sort Key
- GSI를 통한 역방향 조회
- 일관성 모델
- Hot Key의 기초

---

## 1. Entity보다 Access Pattern을 먼저 쓴다

RDB식 출발:

```text
SHOP 테이블
TEMPLATE 테이블
CORNER 테이블
```

DynamoDB식 출발:

```text
AP1. Shop 전체 전시 조회
AP2. Template의 Corner 목록
AP3. Corner 하나 조회
AP4. Corner 상품 목록
AP5. Product 기준 역조회
```

DynamoDB에서는 이 목록이 사실상 데이터 모델 요구사항이다.

---

## 2. Single Table Design

한 DynamoDB Table 안에 여러 Entity Type을 저장할 수 있다.

```text
PK              SK                              TYPE
----------------------------------------------------------------
SHOP$21087      META                            SHOP
SHOP$21087      TEMPLATE$886                    TEMPLATE
SHOP$21087      TEMPLATE$886$CORNER$2981        CORNER
SHOP$21087      TEMPLATE$886$CORNER$2982        CORNER
```

목표는 "테이블을 무조건 하나만 쓰기"가 아니다.

> 같이 읽는 데이터를 Query하기 좋은 형태로 배치한다.


---

## 3. Item마다 Attribute가 달라도 된다

```json
{
  "pk": "SHOP$21087",
  "sk": "META",
  "shopName": "..."
}
```

```json
{
  "pk": "SHOP$21087",
  "sk": "TEMPLATE$886$CORNER$2982",
  "dcornId": "quick_menu_08",
  "status": "ACTIVE"
}
```

DynamoDB Table을 RDB Table과 1:1 대응시키지 않는다.

---

## 4. Composite Sort Key

```text
TEMPLATE$886$CORNER$2982
```

는 단순 문자열이 아니다.

```text
TEMPLATE
  ↓
886
  ↓
CORNER
  ↓
2982
```

라는 계층과 조회 Prefix를 표현한다.

Template 886의 Corner:

```sql
PK = SHOP$21087
AND begins_with(SK, "TEMPLATE$886$CORNER$")
```

---

## 5. Key 순서가 중요한 이유

다음 두 SK를 비교한다.

```text
TEMPLATE$886$CORNER$2982
CORNER$2982$TEMPLATE$886
```

첫 번째는:

```text
Template → Corner
```

목록 조회에 유리하다.

두 번째는:

```text
Corner → Template
```

방향의 Prefix에는 유리할 수 있다.

Key 순서는 Access Pattern을 반영한다.

---

## 6. JOIN 대신 비정규화

RDB:

```text
CORNER_PRODUCT
      ↓ JOIN
PRODUCT
```

DynamoDB에서는 조회가 매우 잦다면 필요한 상품 정보를 관계 Item에 중복 저장할 수 있다.

```text
PK = CORNER$2982
SK = PRODUCT$100

productName = "운동화"
price = 10000
```

원본 Product Item이 별도로 있어도 화면에 필요한 일부 데이터를 복제할 수 있다.

---

## 7. 중복 데이터의 Trade-off

장점:

```text
읽기 단순
JOIN 불필요
네트워크 왕복 감소
```

단점:

```text
쓰기 복잡
복제 데이터 동기화 필요
일시적 불일치 가능
```

즉:

```text
쓰기 복잡성 ↑
읽기 효율 ↑
```

을 교환하는 것이다.

---

## 8. GSI로 역방향 Access Pattern 만들기

기본:

```text
PK = CORNER$2982
SK = PRODUCT$100
```

요구:

```text
PRODUCT$100이 어떤 Corner에 있는가?
```

GSI:

```text
GSI1PK = PRODUCT$100
GSI1SK = CORNER$2982
```

Base Table:

```text
Corner → Product
```

GSI:

```text
Product → Corner
```

---

## 9. Strong vs Eventually Consistent

### Eventually Consistent Read

쓰기 직후 잠시 이전 상태가 보일 가능성을 허용한다.

### Strongly Consistent Read

성공한 최신 쓰기를 반영한 읽기가 필요한 경우 사용한다.

중요:

```text
Base Table / LSI
→ Strong Read 선택 가능

GSI
→ Eventually Consistent Read만 지원
```

따라서:

> 쓰기 직후 반드시 최신 상태를 읽어야 하는 조회를 GSI에 의존해도 되는가?

를 설계 때 물어야 한다.

---

## 10. Hot Key

좋지 않은 트래픽 분포:

```text
SHOP$100      10 req/s
SHOP$200      12 req/s
SHOP$21087    50,000 req/s
```

PK가 존재한다고 자동으로 트래픽이 균등 분산되는 것은 아니다.

Partition Key는:

```text
데이터 그룹화
+
트래픽 분산
```

둘을 고려해야 한다.

---

## 11. 모든 데이터를 하나의 PK에 넣으면 안 되는 이유

```text
PK = SHOP$21087
```

아래에 상품까지 수백만 개가 계속 쌓인다면:

```text
조회 범위 증가
Item Collection 증가
특정 Shop 트래픽 집중
```

문제가 생길 수 있다.

Corner 상품 API가 독립적으로 호출된다면:

```text
PK = CORNER$2982
```

처럼 Aggregate를 분리하는 것도 후보가 된다.

---

## 12. Transaction이 필요한 중복과 아닌 중복

주문과 재고:

```text
주문 생성
+
재고 감소
```

둘 중 하나만 성공하면 안 된다.

→ Transaction 후보.

검색용 복제 데이터:

```text
Product 수정
→ Corner의 상품명 복제
```

몇 초 늦게 반영되어도 괜찮다면 이벤트 기반 Eventually Consistent 구조도 가능하다.

판단 기준:

```text
같은 요청에서 모두 성공해야 함
→ Transaction

실패해도 재시도·보상으로 복구 가능하고, 잠시 불일치를 허용함
→ 이벤트 기반 복제
```

Transaction은 무결성을 위한 도구이고, 검색·목록용 복제는 읽기 모델을 위한 도구다. 둘을 같은 문제로 취급하지 않는다.

---

## 13. 설계 연습

요구:

```text
1. Shop의 Template 조회
2. Template의 Corner 조회
3. Corner 한 건 조회
4. Product 기준 Corner 역조회
```

후보:

```text
PK = SHOP$21087
SK = TEMPLATE$886
SK = TEMPLATE$886$CORNER$2982
```

역조회:

```text
GSI1PK = PRODUCT$100
GSI1SK = CORNER$2982
```

중요한 것은 이 Key가 "예뻐서"가 아니라 요구한 Query를 지원하기 때문에 선택된다는 점이다.

---

## Part 2 체크

다음 질문에 답할 수 있어야 한다.

> 왜 DynamoDB에서는 데이터 중복을 허용하는가?

답:

> Access Pattern을 빠르게 처리하기 위해 읽기 시 JOIN 대신 필요한 데이터를 미리 함께 저장할 수 있기 때문이다. 대신 쓰기와 일관성 관리 비용을 감수한다.
