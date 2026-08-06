---
title: "리액티브 카프카 pollTimeout과 max.poll.interval.ms"
permalink: /kafka/reactive-kafka-poll-timeout/
categories:
  - kafka
tags:
  - kafka
  - reactive
  - spring
last_modified_at: 2026-06-17T09:00:00+09:00
---

# 리액티브 카프카 pollTimeout과 max.poll.interval.ms

Reactive Kafka를 사용할 때 가장 자주 혼동되는 설정 중 하나가 `pollTimeout`과 Kafka Consumer의 `max.poll.interval.ms`이다. 이름이 비슷해서 같은 범주의 옵션처럼 보이지만, 실제로는 제어하는 대상이 완전히 다르다.

- `pollTimeout`: 내부 `poll()` 호출이 데이터를 기다리는 시간
- `max.poll.interval.ms`: 다음 `poll()` 호출까지 허용되는 최대 간격

이 둘을 잘못 이해하면 처리 지연, 불필요한 리밸런싱, 중복 소비 같은 운영 이슈로 이어질 수 있다.

## 핵심 차이

### `pollTimeout`
Reactive Kafka의 `ReceiverOptions#pollTimeout`은 Kafka 브로커에서 레코드를 가져오기 위해 `poll()`이 얼마나 오래 대기할지를 결정한다.

```java
ReceiverOptions.<String, String>create(getConsumerProps())
    .subscription(Collections.singleton(consumerProp.getTopicName()))
    .withKeyDeserializer(new StringDeserializer())
    .withValueDeserializer(deserializer)
    .pollTimeout(Duration.ofMillis(2000));
```

이 값이 크면 빈 토픽에서 불필요한 CPU 사용을 줄일 수 있고, 너무 작으면 잦은 `poll()` 호출로 오버헤드가 늘어날 수 있다. 다만 이 값이 크다고 해서 Consumer 그룹에서 제외되는 것은 아니다.

### `max.poll.interval.ms`
반면 `max.poll.interval.ms`는 Consumer가 정상적으로 일하고 있는지 판단하기 위한 Kafka Consumer Group 레벨의 안전장치다.

- 기본값: `300000`ms
- 의미: 이전 `poll()` 이후 다음 `poll()`이 일정 시간 안에 호출되지 않으면 해당 Consumer를 비정상으로 간주
- 결과: Consumer Group에서 제외되고 파티션이 다른 Consumer에게 재할당됨

즉 `pollTimeout`은 "한 번의 poll이 얼마나 기다릴지"이고, `max.poll.interval.ms`는 "다음 poll이 언제까지 다시 호출되어야 하는지"다.

## 왜 운영 장애로 이어질 수 있는가

이름만 보고 `pollTimeout`을 늘리면 긴 처리 시간을 커버할 수 있다고 생각하기 쉽다. 하지만 실제 장애는 보통 메시지 처리 시간이 길어져서 다음 `poll()` 호출이 늦어질 때 발생한다.

예를 들어 다음과 같은 상황을 생각해볼 수 있다.

1. Consumer가 대량 메시지를 한 번에 가져온다.
2. 각 메시지를 외부 API 호출 또는 DB 작업과 함께 오래 처리한다.
3. 처리하는 동안 다음 `poll()`이 지연된다.
4. `max.poll.interval.ms`를 초과한다.
5. 리밸런싱이 발생하고, 아직 커밋되지 않은 메시지가 다시 전달된다.

이 경우 문제의 원인은 `pollTimeout`이 아니라 처리 시간과 `max.poll.interval.ms`의 불균형이다.

## 테스트용 Producer 예시

아래 코드는 토픽에 테스트 메시지를 대량 적재할 때 사용할 수 있다.

```java
public static void main(String[] args) {
    publishMessages("test", 9999);
}

public static void publishMessages(String topic, int messageCount) {
    Properties props = new Properties();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

    KafkaProducer<String, String> producer = new KafkaProducer<>(props);

    for (int i = 0; i < messageCount; i++) {
        String key = "key-" + i;
        String value = "message-" + i;
        producer.send(new ProducerRecord<>(topic, key, value), (metadata, exception) -> {
            if (exception != null) {
                System.err.printf("Error publishing message: %s%n", exception.getMessage());
            } else {
                System.out.printf(
                    "Message published to topic %s, partition %d, offset %d%n",
                    metadata.topic(),
                    metadata.partition(),
                    metadata.offset()
                );
            }
        });
    }

    producer.close();
}
```

## 테스트용 Consumer 예시

병렬 처리 구조에서는 처리량은 높일 수 있지만, ack 시점과 처리 시간 관리가 훨씬 중요해진다.

```java
public void ackParallel() {
    Flux.defer(() -> KafkaReceiver.create(receiverOptions).receive())
        .parallel()
        .flatMap(this::processMessage)
        .subscribe(record -> record.receiverOffset().acknowledge());
}
```

이 구조에서 주의할 점은 다음과 같다.

- 병렬 처리 중 일부 작업이 지연되면 ack 시점이 늦어질 수 있다.
- 처리 파이프라인이 막히면 다음 `poll()` 호출 주기에도 영향을 줄 수 있다.
- 파티션 순서 보장이 필요한 경우 무분별한 병렬화는 오히려 문제를 만든다.

## 실무에서 확인해야 할 포인트

### 1. 처리 시간과 `max.poll.interval.ms`의 관계
- 메시지 한 건의 최대 처리 시간
- 한 번에 가져오는 레코드 수(`max.poll.records`)
- 외부 시스템 지연 시간
- 재시도 로직 포함 여부

이 값들을 합쳐서 최악의 경우 다음 `poll()`이 얼마나 늦어질 수 있는지 계산해야 한다.

### 2. `pollTimeout`의 적절한 사용
- 너무 짧으면 불필요한 polling 오버헤드 증가
- 너무 길면 종료나 상태 변화 감지가 둔해질 수 있음
- 일반적으로 수백 ms에서 수 초 사이로 조정

`pollTimeout`은 성능 튜닝 항목이지, 장애 회피용 타임아웃은 아니다.

### 3. 병렬 처리 시 ack 전략
- 처리 완료 후 ack
- 실패 시 재처리 전략 분리
- 필요한 경우 DLQ(Dead Letter Queue) 도입

ack를 너무 이르게 하면 데이터 유실 위험이 생기고, 너무 늦게 하면 중복 소비 가능성이 커진다.

## 운영 관점의 권장 사항

1. 긴 처리 로직이 있다면 먼저 `max.poll.interval.ms`를 검토한다.
2. 대량 배치성 처리라면 `max.poll.records`도 함께 줄여서 한 번에 가져오는 작업량을 제한한다.
3. 외부 API, DB I/O가 긴 경우 비동기 처리와 backpressure 전략을 함께 설계한다.
4. 리밸런싱 발생 로그를 기준으로 실제 장애 원인이 `pollTimeout`인지 처리 지연인지 구분한다.

## 결론

Reactive Kafka의 `pollTimeout`과 Kafka Consumer의 `max.poll.interval.ms`는 이름은 비슷하지만 역할이 전혀 다르다.

- `pollTimeout`은 브로커에서 데이터를 기다리는 시간
- `max.poll.interval.ms`는 Consumer가 살아 있다고 판단되는 최대 간격

운영 환경에서는 이 둘을 함께 보되, 장애 예방의 핵심은 `pollTimeout`보다 메시지 처리 시간, ack 타이밍, 리밸런싱 조건을 정확히 이해하는 데 있다.
