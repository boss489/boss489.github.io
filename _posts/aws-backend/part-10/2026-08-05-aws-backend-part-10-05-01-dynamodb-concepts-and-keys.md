---
title: "DynamoDB 실무 학습 Part 1 — 핵심 개념과 Key"
permalink: /aws-backend/part-10/05-dynamodb/01-concepts-and-keys/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 1 — 핵심 개념과 Key

## 목표

이 Part가 끝나면 다음을 설명할 수 있어야 한다.

- DynamoDB의 Item, Attribute, Table
- Partition Key(PK)와 Sort Key(SK)
- `PK + SK`의 유일성
- GetItem, Query, Scan의 차이
- `KeyConditionExpression`
- `begins_with`, `between`
- GSI와 LSI의 기본 차이

---

## 1. DynamoDB는 RDB와 무엇이 다른가

RDB는 보통 데이터의 관계와 정규화에서 시작한다.

```text
SHOP
  ↓
TEMPLATE
  ↓
CORNER
```

DynamoDB는 먼저 **어떻게 조회할 것인가(Access Pattern)** 를 생각한다.

```text
"Shop 21087의 Template 886에 속한 Corner를 조회한다."
```

그 뒤 그 조회를 빠르게 만들도록 PK/SK를 설계한다.

---

## 2. Item과 Attribute

DynamoDB의 한 레코드는 Item이다.

```json
{
  "hk": "DSHOP$21087",
  "sk": "TMPL_SEQ$886+2982",
  "dcornId": "quick_menu_08"
}
```

`hk`, `sk`, `dcornId`가 Attribute다.

RDB의 Row와 비슷하지만 모든 Item이 동일한 Attribute를 가질 필요는 없다.

---

## 3. Partition Key

예:

```text
hk = DSHOP$21087
```

Partition Key는 데이터를 논리적으로 묶고 분산시키는 핵심 Key다.

같은 PK를 가진 Item은 같은 Item Collection에 속한다고 생각하면 이해하기 쉽다.

```text
DSHOP$21087
 ├ TMPL_SEQ$886+2981
 ├ TMPL_SEQ$886+2982
 └ TMPL_SEQ$887+3001
```

---

## 4. Sort Key

Sort Key는 같은 PK 내부의 Item을 구분하고 정렬한다.

```text
PK              SK
------------------------------------------------
DSHOP$21087     TMPL_SEQ$886+2981
DSHOP$21087     TMPL_SEQ$886+2982
DSHOP$21087     TMPL_SEQ$887+3001
```

중요:

> PK만 같다고 같은 Item이 아니다. PK + SK 조합이 Item을 식별한다.

---

## 5. PK와 SK를 둘 다 알아도 여러 건 나올까?

Primary Key가 `PK + SK`인 테이블에서 두 값을 **정확히 모두 지정**하면 최대 한 Item이다.

```text
DSHOP$21087 + TMPL_SEQ$886+2982
```

이 조합은 테이블에서 유일하다.

따라서 다음 조건은 결과가 0건 또는 1건이다.

```sql
hk = :hk AND sk = :sk
```

여러 건이 필요한 경우는 보통 SK를 범위 조건으로 사용한다.

```sql
hk = :hk
AND begins_with(sk, :prefix)
```

---

## 6. GetItem vs Query

### 정확한 PK + SK를 모두 안다

```text
PK = DSHOP$21087
SK = TMPL_SEQ$886+2982
```

→ GetItem 계열이 가장 직접적이다.

DynamoDBMapper:

```java
Dshop shop = dynamoDBMapper.load(
        Dshop.class,
        "DSHOP$21087",
        "TMPL_SEQ$886+2982"
);
```

### PK는 알고 SK 범위에서 여러 건을 원한다

```text
PK = DSHOP$21087
SK begins_with TMPL_SEQ$886+
```

→ Query가 적합하다.

---

## 7. 현재 코드 해석

```java
DynamoDBQueryExpression<Dshop> queryExpression =
        new DynamoDBQueryExpression<Dshop>()
                .withKeyConditionExpression("hk = :hk and sk = :sk")
                .withExpressionAttributeValues(
                        Map.of(
                                ":hk", new AttributeValue("DSHOP$21087"),
                                ":sk", new AttributeValue("TMPL_SEQ$886+2982")
                        )
                );
```

`":hk"`는 Java 변수명이 아니다.

Expression 안의 placeholder다.

```text
hk = :hk
     ↓
:hk → DSHOP$21087
```

```text
sk = :sk
     ↓
:sk → TMPL_SEQ$886+2982
```

---

## 8. KeyConditionExpression

Query의 핵심 조건이다.

Partition Key에는 기본적으로 equality가 필요하다.

```sql
hk = :hk
```

Sort Key에는 다음과 같은 조건을 사용할 수 있다.

```text
=
<
<=
>
>=
BETWEEN
begins_with
```

예:

```sql
hk = :hk
AND begins_with(sk, :sk)
```

---

## 9. begins_with

데이터:

```text
TMPL_SEQ$886+2981
TMPL_SEQ$886+2982
TMPL_SEQ$886+2983
TMPL_SEQ$887+3001
```

조건:

```sql
begins_with(sk, "TMPL_SEQ$886+")
```

결과:

```text
TMPL_SEQ$886+2981
TMPL_SEQ$886+2982
TMPL_SEQ$886+2983
```

이 때문에 SK 문자열 구조 자체가 데이터 모델의 일부다.

---

## 10. BETWEEN

시간이나 순번 범위 조회에도 SK를 활용할 수 있다.

```text
PK = ORDER$CUSTOMER$100
SK = 2026-08-01T10:00:00
SK = 2026-08-05T11:00:00
SK = 2026-08-20T12:00:00
```

```sql
PK = :pk
AND SK BETWEEN :from AND :to
```

---

## 11. Query와 Scan

### Query

```text
PK를 사용하여 특정 Item Collection을 조회
```

### Scan

```text
테이블 또는 인덱스의 데이터를 넓게 평가
```

온라인 API에서 Scan이 보이면 다음을 먼저 질문한다.

> 이 조회를 PK/SK 또는 GSI로 표현할 수 없는가?

Scan 자체가 금지된 기능은 아니다. 관리 배치, 마이그레이션, 검증 작업 등에서는 합리적일 수 있다.


---

## 12. GSI

기본 Key가 다음이라고 하자.

```text
PK = SHOP$21087
SK = CORNER$2982
```

그런데 다음 조회가 필요하다.

```text
CORNER$2982를 기준으로 Shop을 찾고 싶다.
```

Item에:

```text
GSI1PK = CORNER$2982
GSI1SK = SHOP$21087
```

를 두면 다른 조회 방향을 만들 수 있다.

GSI는 Base Table과 다른 Partition Key를 가질 수 있다.

중요한 제약:

```text
Base Table 쓰기 성공
≠
GSI Query에서 즉시 조회 가능
```

GSI는 Base Table 변경을 비동기로 반영하므로, 방금 생성한 주문·상태 변경처럼 즉시 최신값이 반드시 필요한 흐름은 GSI Query만으로 확인하지 않는다. 정확한 PK/SK로 Base Table을 읽거나, 짧은 지연을 허용하는 API인지 먼저 결정한다.

---

## 13. LSI

LSI(Local Secondary Index)는 Base Table과 **같은 Partition Key**를 공유하고 다른 Sort Key를 사용한다.

개념 비교:

```text
GSI
- PK 변경 가능
- SK도 별도 정의 가능
- 테이블 생성 후 추가 가능
- Eventually Consistent Read

LSI
- Base Table PK 공유
- 다른 SK
- 테이블 생성 시 정의
- Strongly Consistent Read 선택 가능
```

실무에서는 GSI를 더 자주 만나지만 LSI의 존재와 차이는 알아두는 것이 좋다.

---

## 14. 핵심 판단표

```text
정확한 PK만 있는 단순키 테이블
→ GetItem

정확한 PK + SK
→ GetItem / load

PK + SK 범위
→ Query

다른 조회 방향
→ GSI/LSI 검토

Key 없이 넓은 탐색
→ Scan
```

---

## 15. 연습 문제

데이터:

```text
PK = DSHOP$21087

SK
TMPL_SEQ$886+2981
TMPL_SEQ$886+2982
TMPL_SEQ$886+2983
TMPL_SEQ$887+3001
```

질문:

1. `886+2982` 한 건만 찾으려면?
2. Template 886 전체를 찾으려면?
3. `cornerSeq=2982`만 알고 Shop을 역조회하려면?

정답:

```text
1. GetItem/load
2. Query + begins_with
3. 기본 Key만으로 어렵다면 GSI Access Pattern 검토
```

---

## Part 1 체크

다음 문장을 설명할 수 있으면 통과다.

> DynamoDB에서 `PK + SK`는 Item을 유일하게 식별하고, Query는 PK를 기준으로 SK의 범위를 효율적으로 읽는다.
