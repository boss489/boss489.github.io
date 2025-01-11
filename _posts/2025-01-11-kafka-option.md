---
title: "카프카 max.poll.interval.ms"
categories:
  - kafka
tags:
  - kafka
last_modified_at: 2025-01-11T06:31:52-07:00
---

## kafka max.poll.interval.ms

Kafka는 데이터 스트리밍 처리에 최적화된 분산형 메시징 플랫폼입니다. 이 글에서는 Kafka Consumer 설정 중 하나인 `max.poll.interval.ms`의 동작 방식과 활용 방법을 자세히 알아보겠습니다.

## `max.poll.interval.ms`란?
`max.poll.interval.ms`는 Kafka Consumer가 메시지를 가져오기 위해 `poll()` 메서드를 호출해야 하는 최대 간격을 지정합니다. 이 설정은 Consumer의 비정상적인 동작을 감지하고, 그룹의 안정성을 유지하기 위한 중요한 매개변수입니다.

- **기본값**: 300,000ms (5분)
- **작동 원리**: Consumer가 지정된 시간 내에 `poll()`을 호출하지 않으면, Consumer 그룹에서 제거되고, 해당 파티션은 다른 Consumer에게 재할당됩니다.

## 주요 사용 사례
1. **장애 감지**
   Consumer가 비정상적으로 작동하거나 중단된 경우, 설정된 시간 내에 `poll()`을 호출하지 않으면 자동으로 그룹에서 제외됩니다.
2. **긴 메시지 처리**
   데이터 처리 시간이 긴 작업이 필요한 경우, `max.poll.interval.ms` 값을 늘려야 합니다.(처리시간내에 처리가 되지 않을경우 리밸런싱후 동일 메시지를 중복 컨슈밍 할 수 있음.!!)


## 기본 설정 예시
Java 코드에서 `max.poll.interval.ms`를 설정하는 방법은 다음과 같습니다:

```java
Properties props = new Properties();
props.put("max.poll.interval.ms", "300000");  // 5분
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
```

기본값은 5분으로 설정되며, Consumer는 이 시간 안에 `poll()`을 호출해야 합니다.

## 긴 처리 시간을 다루는 방법
긴 작업을 처리할 때, `max.poll.interval.ms`의 기본값을 초과하면 Consumer는 실패로 간주됩니다. 이를 방지하려면 다음과 같은 전략을 사용할 수 있습니다:

1. **값 조정**
   처리 시간에 맞춰 `max.poll.interval.ms` 값을 늘립니다.

2. **Offset 수동 커밋**
   긴 작업 중에도 메시지 중복 처리를 방지하려면 수동으로 Offset을 커밋하는 방식을 사용합니다:

   ```java
   consumer.commitSync(Collections.singletonMap(
       new TopicPartition(record.topic(), record.partition()),
       new OffsetAndMetadata(record.offset() + 1)
   ));
   ```

3. **병렬 처리**
   메시지 처리를 병렬화하여 전체 처리 시간을 단축할 수 있습니다.

## 결론
`max.poll.interval.ms`는 Kafka Consumer 설정에서 중요한 역할을 합니다. 이를 적절히 조정하면 장애 감지 및 안정적인 데이터 처리가 가능합니다. 긴 처리 작업이 필요한 경우에는 Offset 관리와 설정 값을 조정하여 Consumer 그룹의 안정성을 유지하세요.
