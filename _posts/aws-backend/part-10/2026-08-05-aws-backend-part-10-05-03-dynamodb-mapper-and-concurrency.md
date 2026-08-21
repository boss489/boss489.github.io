---
title: "DynamoDB 실무 학습 Part 3 — DynamoDBMapper, Repository, 쓰기 동시성"
permalink: /aws-backend/part-10/05-dynamodb/03-mapper-and-concurrency/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 3 — DynamoDBMapper, Repository, 쓰기 동시성

## 목표

- `@DynamoDBHashKey`, `@DynamoDBRangeKey`
- `load()`와 `query()`
- Repository 추상화
- GSI 매핑
- Pagination
- Conditional Write
- Atomic Update
- Optimistic Lock

---

## 1. DynamoDBMapper Entity

```java
@DynamoDBTable(tableName = "dshop")
public class Dshop {

    private String hk;
    private String sk;
    private String dcornId;

    @DynamoDBHashKey(attributeName = "hk")
    public String getHk() {
        return hk;
    }

    @DynamoDBRangeKey(attributeName = "sk")
    public String getSk() {
        return sk;
    }

    @DynamoDBAttribute(attributeName = "dcornId")
    public String getDcornId() {
        return dcornId;
    }
}
```

```text
@DynamoDBHashKey
→ Partition Key

@DynamoDBRangeKey
→ Sort Key
```

---

## 2. load()

정확한 Key:

```java
Dshop shop = dynamoDBMapper.load(
        Dshop.class,
        "DSHOP$21087",
        "TMPL_SEQ$886+2982"
);
```

결과:

```text
존재 → 객체
없음 → null
```

Repository:

```java
public Optional<Dshop> findCorner(
        String shopNo,
        long tmplSeq,
        long cornerSeq
) {
    return Optional.ofNullable(
            dynamoDBMapper.load(
                    Dshop.class,
                    DshopKey.hk(shopNo),
                    DshopKey.sk(tmplSeq, cornerSeq)
            )
    );
}
```

---

## 3. Key 생성 규칙 분리

좋지 않은 호출부:

```java
"DSHOP$" + shopNo
"TMPL_SEQ$" + tmplSeq + "+" + cornerSeq
```

개선:

```java
public final class DshopKey {

    public static String hk(String shopNo) {
        return "DSHOP$" + shopNo;
    }

    public static String sk(
            long tmplSeq,
            long cornerSeq
    ) {
        return "TMPL_SEQ$"
                + tmplSeq
                + "+"
                + cornerSeq;
    }

    public static String templatePrefix(long tmplSeq) {
        return "TMPL_SEQ$" + tmplSeq + "+";
    }
}
```

Service가 DynamoDB 문자열 포맷을 알 필요가 없게 한다.

---

## 4. query()

Template 886의 여러 Corner:

```java
Map<String, AttributeValue> eav = Map.of(
        ":hk", new AttributeValue().withS(
                DshopKey.hk("21087")
        ),
        ":sk", new AttributeValue().withS(
                DshopKey.templatePrefix(886)
        )
);

DynamoDBQueryExpression<Dshop> expression =
        new DynamoDBQueryExpression<Dshop>()
                .withKeyConditionExpression(
                        "hk = :hk AND begins_with(sk, :sk)"
                )
                .withExpressionAttributeValues(eav);

List<Dshop> shops =
        dynamoDBMapper.query(
                Dshop.class,
                expression
        );
```

---

## 5. Repository는 Access Pattern을 표현한다

피하고 싶은 API:

```java
repository.query(
    DynamoDBQueryExpression<Dshop> expression
);
```

좋은 방향:

```java
findCorner(shopNo, tmplSeq, cornerSeq)
findCornersByTemplate(shopNo, tmplSeq)
findAllByShop(shopNo)
```

Service:

```java
List<Dshop> corners =
        repository.findCornersByTemplate(
                "21087",
                886
        );
```

---

## 6. GSI 매핑

```java
@DynamoDBIndexHashKey(
        globalSecondaryIndexName = "gsi1",
        attributeName = "gsi1pk"
)
public String getGsi1pk() {
    return gsi1pk;
}

@DynamoDBIndexRangeKey(
        globalSecondaryIndexName = "gsi1",
        attributeName = "gsi1sk"
)
public String getGsi1sk() {
    return gsi1sk;
}
```

GSI Query에서는:

```java
.withIndexName("gsi1")
.withConsistentRead(false)
```

를 기억한다.

GSI는 Strongly Consistent Read를 지원하지 않는다.

---

## 7. Pagination

DynamoDB Query는 한 번의 응답에 무한한 데이터를 담지 않는다.

한 Query/Scan 응답은 읽은 데이터 기준 최대 약 1MB 단위로 페이지가 나뉜다.

개념:

```text
Page 1
 ↓
LastEvaluatedKey
 ↓
Page 2 요청의 ExclusiveStartKey
```

API Cursor로 변환할 수도 있다.

```json
{
  "items": [],
  "nextCursor": "..."
}
```

Offset Pagination과 다른 점을 이해해야 한다.

---

## 8. DynamoDBMapper의 PaginatedQueryList

`query()`의 반환값은 일반 `ArrayList`처럼 생각하면 위험하다.

추가 페이지를 지연 로딩할 수 있다.

```java
List<Dshop> result = mapper.query(...);
```

다음 동작이 추가 요청을 유발할 수 있다.

```java
result.size();
전체 순회
```

대량 데이터에서는 페이지 동작을 명시적으로 이해한다.

---

## 9. ExpressionAttributeNames

Attribute 이름이 DynamoDB 예약어와 충돌하거나 특수한 이름이면 이름 placeholder를 사용한다.

예:

```text
status
name
size
```

등은 문맥에 따라 Expression 작성 시 충돌 가능성을 고려한다.

```java
.withExpressionAttributeNames(
    Map.of("#st", "status")
)
.withFilterExpression("#st = :active")
```

값 placeholder:

```text
:active
```

이름 placeholder:

```text
#st
```

역할이 다르다.

---

## 10. save()를 INSERT라고 생각하지 않는다

같은 PK/SK로 저장하면 두 Item이 생기지 않는다.

```text
PK + SK는 유일
```

기존 Item을 변경하는 효과가 생길 수 있다.

기본 `SaveBehavior.UPDATE`는 매핑한 Attribute만 갱신하고, 매핑하지 않은 기존 Attribute는 보존한다. 다만 매핑한 Attribute에 `null`을 넣으면 해당 Attribute가 제거될 수 있다.

```text
부분 수정
→ null 의미와 SaveBehavior를 확인

전체 덮어쓰기
→ CLOBBER는 미매핑 Attribute와 Version 조건까지 무시할 수 있으므로 신중히 사용
```

따라서:

> 없을 때만 생성

이 필요하면 조건부 쓰기를 검토한다.

---

## 11. Conditional Write

개념:

```text
attribute_not_exists(pk)
```

조건이 참일 때만 Put.

활용:

```text
중복 주문 방지
Idempotency Key
최초 생성 보장
상태 전이 보호
```

예:

```text
status가 PENDING일 때만 PROCESSING으로 변경
```

조건부 Update를 사용하면 경쟁 상태를 줄일 수 있다.

---

## 12. Lost Update

현재:

```text
stock = 10
```

A와 B가 동시에 읽는다.

```text
A → 10
B → 10
```

각자 1 감소 후:

```text
A → 9 저장
B → 9 저장
```

기대값 8이 아니라 9가 된다.

---

## 13. Atomic Update

단순 숫자 증감은 Read → 계산 → Save보다 DynamoDB에서 원자적으로 변경하는 방식을 우선 검토한다.

개념:

```text
SET stock = stock - :one
```

그리고:

```text
stock >= :one
```

같은 Condition을 결합할 수 있다.

---

## 14. Optimistic Lock

```java
@DynamoDBVersionAttribute
public Long getVersion() {
    return version;
}
```

A/B가 version 1을 읽고 A가 먼저 저장하면 version이 증가한다.

B가 오래된 version으로 저장하면 충돌을 감지할 수 있다.

```text
Pessimistic:
미리 잠금

Optimistic:
저장 시 충돌 확인
```

---

## 15. Repository Integration Test

```java
@Test
void 특정_코너를_조회한다() {

    Dshop shop = repository
            .findCorner(
                    "21087",
                    886,
                    2982
            )
            .orElseThrow();

    assertThat(shop.getDcornId())
            .isEqualTo("quick_menu_08");
}
```

여기서는 실제:

```text
Annotation
PK/SK
Query
Index
Serialization
```

을 검증한다.

Service Unit Test에서는 DynamoDB SDK를 몰라도 된다.

---

## 16. 원래 코드 리팩터링

원래:

```java
.withKeyConditionExpression(
    "hk = :hk and sk = :sk"
)
```

PK와 SK가 정확하므로 결과는 최대 한 건이다.

더 의도가 명확한 형태:

```java
repository.findCorner(
        "21087",
        886,
        2982
);
```

내부:

```java
mapper.load(...)
```

---

## Part 3 체크

다음 기준을 기억한다.

```text
정확한 PK + SK
→ load

PK + SK 범위
→ query

SDK 문법
→ Repository 내부

동시 수정
→ Atomic / Condition / Optimistic Lock 검토
```

---

# 보강 — Conditional Update 실전

## 17. PutItem과 UpdateItem의 차이

개념적으로:

```text
PutItem
→ Item 전체를 새 값으로 Put

UpdateItem
→ 특정 Attribute를 표현식으로 변경
```

동시성이나 부분 변경이 중요하다면 `UpdateItem`의 표현식이 매우 강력하다.

---

## 18. UpdateExpression

대표 연산:

```text
SET
REMOVE
ADD
DELETE
```

예:

```text
SET status = :processing
```

여러 값:

```text
SET status = :processing,
    updatedAt = :now
```

Attribute 제거:

```text
REMOVE temporaryValue
```

숫자/Set 갱신에서는 `ADD`가 사용될 수 있지만, 표현식의 의미와 데이터 타입을 정확히 이해하고 선택한다.

---

## 19. attribute_not_exists

중복 생성 방지:

```text
ConditionExpression

attribute_not_exists(pk)
```

의미:

```text
해당 Key의 Item이 아직 없을 때만 Put
```

활용:

```text
주문 중복 생성 방지
Idempotency Key 최초 등록
한 번만 생성되어야 하는 작업
```

---

## 20. 상태 전이 보호

요구사항:

```text
PENDING → PROCESSING
```

PENDING 상태일 때만 변경하고 싶다.

```text
UpdateExpression
SET #status = :processing

ConditionExpression
#status = :pending
```

두 요청이 동시에 상태를 변경하려 해도 Condition을 통과한 요청만 성공하도록 만들 수 있다.

---

## 21. 재고 차감

피하고 싶은 방식:

```java
Product product = repository.find(id);
product.setStock(product.getStock() - 1);
repository.save(product);
```


더 나은 후보:

```text
UpdateExpression
SET stock = stock - :one

ConditionExpression
stock >= :one
```

핵심:

```text
Read
→ Java 계산
→ Write
```

사이의 경쟁 구간을 없애고 DynamoDB에서 조건과 갱신을 원자적으로 수행한다.

---

## 22. if_not_exists

Attribute가 없으면 기본값을 사용해야 할 수 있다.

개념:

```text
SET viewCount =
    if_not_exists(viewCount, :zero) + :one
```

활용:

```text
Counter 초기화 + 증가
Optional Attribute 기본값
```

---

## 23. ReturnValues

Update 후 변경 결과가 필요하면 ReturnValues를 검토한다.

예:

```text
ALL_NEW
UPDATED_NEW
ALL_OLD
UPDATED_OLD
NONE
```

재고 감소 후 최신 재고를 다시 Get하는 대신 Update 결과를 활용할 수 있는지 본다.

---

## 24. ConditionalCheckFailedException

Condition이 거짓이면 단순 시스템 장애가 아니라 **비즈니스 경쟁 조건의 정상적인 결과**일 수 있다.

예:

```text
stock >= 1 실패
→ 품절

status = PENDING 실패
→ 이미 다른 요청이 처리
```

따라서 무조건 500으로 처리하지 않고 도메인 의미로 변환할 수 있다.

```java
throw new SoldOutException();
```

또는:

```java
throw new AlreadyProcessedException();
```

---

## 25. 선택 기준

```text
단순 숫자 증감
→ Atomic Update

특정 상태일 때만 변경
→ ConditionExpression

여러 Item을 함께 원자적으로
→ Transaction

객체 전체의 동시 수정 충돌
→ Optimistic Lock 후보
```

Conditional Write는 DynamoDB 실무에서 매우 중요한 기능이다.
