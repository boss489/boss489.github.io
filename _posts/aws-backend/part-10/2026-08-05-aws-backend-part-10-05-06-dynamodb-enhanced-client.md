---
title: "DynamoDB 실무 학습 Part 6 — AWS SDK v2 Enhanced Client"
permalink: /aws-backend/part-10/05-dynamodb/06-enhanced-client/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# DynamoDB 실무 학습 Part 6 — AWS SDK v2 Enhanced Client

## 목표

- DynamoDBMapper와 Enhanced Client 비교
- `DynamoDbTable`
- `QueryConditional`
- GSI Query
- Pagination
- Condition Expression
- Batch/Transaction
- Async Enhanced Client
- v1 → v2 Migration

---

## 1. SDK 세대

```text
AWS SDK for Java v1
→ DynamoDBMapper

AWS SDK for Java v2
→ DynamoDbClient
→ DynamoDbEnhancedClient
```

SDK가 달라져도:

```text
PK
SK
Query
GSI
Pagination
Consistency
```

는 그대로다.

---

## 2. Bean

```java
@DynamoDbBean
public class Dshop {

    private String hk;
    private String sk;
    private String dcornId;

    @DynamoDbPartitionKey
    public String getHk() {
        return hk;
    }

    @DynamoDbSortKey
    public String getSk() {
        return sk;
    }

    public String getDcornId() {
        return dcornId;
    }

    // setters
}
```

---

## 3. DynamoDbTable

```java
DynamoDbTable<Dshop> table =
        enhancedClient.table(
                "dshop",
                TableSchema.fromBean(Dshop.class)
        );
```

---

## 4. GetItem

```java
Key key = Key.builder()
        .partitionValue(
                DshopKey.hk("21087")
        )
        .sortValue(
                DshopKey.sk(886, 2982)
        )
        .build();

Dshop shop = table.getItem(key);
```

Mapper의 `load()`와 같은 Access Pattern이다.

---

## 5. QueryConditional.keyEqualTo

PK의 전체 Item:

```java
QueryConditional condition =
        QueryConditional.keyEqualTo(
                Key.builder()
                        .partitionValue(
                                DshopKey.hk("21087")
                        )
                        .build()
        );

PageIterable<Dshop> pages =
        table.query(r -> r
                .queryConditional(condition)
        );
```

---

## 6. sortBeginsWith

SDK v1 Expression:

```java
"hk = :hk AND begins_with(sk, :sk)"
```

Enhanced Client에서는 QueryConditional로 표현할 수 있다.

```java
QueryConditional condition =
        QueryConditional.sortBeginsWith(
                Key.builder()
                        .partitionValue(
                                DshopKey.hk("21087")
                        )
                        .sortValue(
                                DshopKey.templatePrefix(886)
                        )
                        .build()
        );
```

핵심:

```text
문법 변경
≠
데이터 모델 변경
```

---

## 7. 다른 Sort 조건

Enhanced Client의 QueryConditional은 다음 계열의 Query를 객체로 표현한다.

```text
keyEqualTo
sortBeginsWith
sortBetween
sortGreaterThan
sortGreaterThanOrEqualTo
sortLessThan
sortLessThanOrEqualTo
```

예:

```java
QueryConditional condition =
        QueryConditional.sortBetween(
                Key.builder()
                        .partitionValue("CUSTOMER$100")
                        .sortValue("2026-08-01")
                        .build(),
                Key.builder()
                        .partitionValue("CUSTOMER$100")
                        .sortValue("2026-08-31")
                        .build()
        );
```

---

## 8. QueryEnhancedRequest

```java
QueryEnhancedRequest request =
        QueryEnhancedRequest.builder()
                .queryConditional(condition)
                .scanIndexForward(false)
                .limit(20)
                .build();
```

주의:

`limit`은 "최종적으로 사용자에게 정확히 20개의 Filter 결과를 보장"하는 의미로 단순 이해하면 안 된다. DynamoDB의 평가/페이지/Filter 순서를 이해해야 한다.

---

## 9. Pagination

```java
PageIterable<Dshop> pages =
        table.query(request);

for (Page<Dshop> page : pages) {
    List<Dshop> items = page.items();

    Map<String, AttributeValue> lastKey =
            page.lastEvaluatedKey();
}
```

`lastEvaluatedKey`를 API Cursor로 인코딩하여 다음 요청에 사용할 수 있다.

---

## 10. GSI

Bean에 Secondary Partition/Sort Key를 매핑하고:

```java
DynamoDbIndex<Dshop> index =
        table.index("gsi1");
```

Index에서 Query한다.

GSI에서는 Strongly Consistent Read를 요청하지 않는다.

---

## 11. putItem / updateItem / deleteItem

```java
table.putItem(item);
table.updateItem(item);
table.deleteItem(key);
```

하지만 단순 API 호출보다 중요한 것은:

```text
중복 생성 방지 조건이 필요한가?
동시 수정 조건이 필요한가?
```

이다.

---

## 12. Condition Expression

Enhanced Client 요청에서도 조건 Expression을 조합할 수 있다.

개념:

```text
attribute_not_exists(pk)
status = :expected
version = :version
```

사용 목적:

```text
중복 생성 방지
상태 전이 보호
Optimistic Concurrency
```

---

## 13. Batch

Enhanced Client는 여러 Table/Key를 묶은 Batch API도 제공한다.

구분:

```text
BatchGet
→ 정확한 여러 Key 조회

BatchWrite
→ 여러 Put/Delete
```

Unprocessed Key/Item 가능성을 항상 고려한다.

---

## 14. Transaction

Enhanced Client의 Transaction API를 사용하여:

```text
주문 Put
재고 Update
쿠폰 Update
```

를 원자적으로 묶을 수 있다.

중요:

```text
Batch
→ 처리 효율

Transaction
→ 원자성
```

---

## 15. Async Enhanced Client

비동기 Client:

```text
DynamoDbEnhancedAsyncClient
DynamoDbAsyncTable
```

Reactive/비동기 Application에서는 동기 Client를 Event Loop에서 직접 호출하여 Blocking시키지 않도록 주의한다.

WebFlux를 사용한다면:

```text
동기 DynamoDB Client
→ Blocking I/O 가능

Async DynamoDB Client
→ 비동기 흐름과 더 자연스럽게 결합 가능
```

단, SDK가 Async라고 해서 Backpressure/메모리/재시도 문제가 자동 해결되는 것은 아니다.

---

## 16. Repository Boundary

```java
public interface DshopRepository {

    Optional<Dshop> findCorner(
            String shopNo,
            long templateSeq,
            long cornerSeq
    );

    List<Dshop> findCornersByTemplate(
            String shopNo,
            long templateSeq
    );
}
```

Service에는:

```text
QueryConditional
DynamoDbTable
AttributeValue
```

을 노출하지 않는다.

---

## 17. v1 → v2 Migration

추천 순서:

```text
1. Mapper 타입이 Service까지 새는지 확인
2. Repository Interface 정리
3. Key Builder 분리
4. Integration Test 확보
5. Enhanced Client 구현 추가
6. Get/Query/GSI 결과 비교
7. Pagination 비교
8. Write Condition 비교
9. 호출부 전환
10. v1 구현 제거
```

---

## 18. Testcontainers / Local Integration Test

DynamoDB 저장소 코드는 Mock만으로는 다음을 충분히 검증하기 어렵다.

```text
Key Mapping
Annotation
Expression
GSI
Serialization
Pagination
```

로컬 DynamoDB 호환 환경 또는 Testcontainers 기반 Integration Test를 두면 Repository 계약 검증에 유용하다.

실제 AWS 환경에서는 추가로:

```text
IAM
실제 Index
환경 변수
Capacity/설정
```

을 확인한다.

---

## Part 6 체크

다음 변환을 설명할 수 있어야 한다.

```text
DynamoDBMapper.load()
→ DynamoDbTable.getItem()

KeyConditionExpression begins_with
→ QueryConditional.sortBeginsWith()

PaginatedQueryList
→ PageIterable / PagePublisher
```

그리고 SDK 변경과 Repository 설계는 별개의 문제라는 것을 이해해야 한다.

---

# 보강 — REST Cursor Pagination 완성 예제

## 19. DynamoDB에 Offset Pagination을 억지로 만들지 않는다

RDB에서 흔한 API:

```text
GET /corners?page=100&size=20
```

DynamoDB Query는 기본적으로:

```text
100페이지로 바로 점프
```

하는 Offset 구조가 아니다.

DynamoDB가 제공하는 자연스러운 방식:

```text
현재 Page
→ LastEvaluatedKey
→ 다음 Query의 ExclusiveStartKey
```

따라서 Cursor Pagination이 잘 맞는다.


---

## 20. API 형태

첫 요청:

```text
GET /shops/21087/templates/886/corners?size=20
```

응답:

```json
{
  "items": [
    {
      "cornerId": "2981"
    },
    {
      "cornerId": "2982"
    }
  ],
  "nextCursor": "eyJoayI6Ii4uLiJ9"
}
```

다음 요청:

```text
GET /shops/21087/templates/886/corners
    ?size=20
    &cursor=eyJoayI6Ii4uLiJ9
```

---

## 21. Repository Page 객체

```java
public record SliceResult<T>(
        List<T> items,
        String nextCursor
) {
}
```

Repository:

```java
SliceResult<Dshop> findCorners(
        String shopNo,
        long templateSeq,
        int size,
        String cursor
);
```

Service는 `LastEvaluatedKey` 같은 AWS SDK 타입을 알 필요가 없다.

---

## 22. Cursor 내부 값

Cursor 내부에는 다음 DynamoDB Key 정보를 직렬화할 수 있다.

```json
{
  "hk": "DSHOP$21087",
  "sk": "TMPL_SEQ$886+2982"
}
```

이를 Base64 URL-safe 형태 등으로 인코딩하여 외부에는 opaque cursor로 제공한다.

중요:

> Cursor를 사용자가 직접 조립해야 하는 공개 데이터 구조처럼 취급하지 않는다.

필요하다면 서명/검증을 추가한다.

Cursor에는 조회 순서를 바꾸는 값도 함께 묶어야 한다.

```text
cursor의 시작 Key
+
정렬 방향 / 필터 / 조회 주체
```

예를 들어 `status=PAID`에 발급한 Cursor를 `status=CANCELLED` 요청에 재사용하면 안 된다. 사용자별 조회라면 Cursor가 해당 사용자와 일치하는지도 검증하고, 장기간 재사용을 막아야 하면 발급 시각 또는 만료 시각을 포함한다.

---

## 23. Enhanced Client에서 시작 Key

개념:

```java
QueryEnhancedRequest.Builder builder =
        QueryEnhancedRequest.builder()
                .queryConditional(condition)
                .limit(size);

if (cursor != null) {
    Map<String, AttributeValue> key =
            cursorCodec.decode(cursor);

    builder.exclusiveStartKey(key);
}
```

Query:

```java
PageIterable<Dshop> pages =
        table.query(builder.build());
```

한 Page를 가져온다.

```java
Page<Dshop> page =
        pages.stream()
                .findFirst()
                .orElseThrow();
```

---

## 24. 다음 Cursor 생성

```java
Map<String, AttributeValue> lastKey =
        page.lastEvaluatedKey();

String nextCursor =
        lastKey == null || lastKey.isEmpty()
                ? null
                : cursorCodec.encode(lastKey);
```

응답:

```java
return new SliceResult<>(
        page.items(),
        nextCursor
);
```

---

## 25. Filter가 있으면 주의

요청:

```text
size = 20
```

이라고 해도 Filter가 있다면 DynamoDB가 20개를 평가한 뒤 일부를 제거할 수 있다.

즉:

```text
limit = 20
≠
반드시 사용자 Item 20개 반환
```

이다.

20개의 최종 결과를 반드시 채우려면 여러 DynamoDB Page를 읽는 Application 로직이 필요할 수 있고, 그만큼 비용과 Latency가 증가한다.

가장 좋은 해결은 핵심 필터 조건을 가능한 한 Key/GSI Access Pattern으로 표현하는 것이다.

---

## 26. Cursor API 체크

```text
LastEvaluatedKey를 그대로 외부에 노출할 것인가?
Cursor 위변조가 문제인가?
Filter 때문에 빈 Page가 가능한가?
Cursor 사용 중 데이터가 변경되면 허용 가능한가?
정렬 방향이 Cursor와 일치하는가?
```

DynamoDB Pagination은 단순 UI 페이지 번호보다 **연속 조회 Cursor**로 이해하는 것이 좋다.
