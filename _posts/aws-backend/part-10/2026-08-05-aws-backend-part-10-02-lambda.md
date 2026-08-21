---
title: "Chapter 02. Lambda"
permalink: /aws-backend/part-10/02-lambda/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Lambda
## 이벤트가 도착한 순간에 실행되는 함수

> **학습 목표**
>
> - Lambda 실행 모델을 설명할 수 있다.
> - Cold Start와 Timeout의 의미를 이해한다.
> - Lambda에 적합한 작업을 구분할 수 있다.
> - 동시성, 재시도, 멱등성을 고려해 함수를 설계할 수 있다.

---

# 왜 Lambda가 필요한가

사용자가 S3에 이미지를 올릴 때마다 썸네일을 생성해야 한다고 가정해 보자.

업로드 시점은 일정하지 않고 실제 변환은 몇 초면 끝나는데 이 작업만을 위해 EC2 프로세스를 계속 실행하면 대부분의 시간이 대기 상태가 된다.

S3 이벤트가 발생한 순간에만 코드가 실행되고 작업이 끝나면 자원을 반환한다면 서버 용량과 배포 환경을 상시 관리할 필요가 없다.

Lambda는 이런 **짧고 독립적인 이벤트 처리**를 함수 단위로 실행하고 트래픽에 맞춰 실행 환경을 확장한다.

반대로 영상 인코딩처럼 15분을 넘거나 상시 연결을 유지하는 작업은 Lambda보다 ECS Fargate, AWS Batch 같은 실행 환경이 적합하다.

---

# Lambda란?

Lambda는 이벤트가 발생했을 때 등록된 코드를 관리형 실행 환경에서 실행하는 AWS의 Function as a Service이다.

API Gateway, EventBridge, S3, SQS 같은 서비스가 Lambda를 호출할 수 있다.

---

# 동작 흐름

S3 이미지 업로드 이벤트를 처리하는 흐름은 다음과 같다.

```
사용자
  │ 이미지 업로드
  ▼
S3 Bucket
  │ ObjectCreated Event
  ▼
Lambda
  ├──▶ 원본 이미지 읽기
  ├──▶ 썸네일 변환
  └──▶ S3 thumbnail/ 경로에 저장
             │
             └──▶ CloudWatch Logs
```

1. S3는 객체 생성 이벤트를 Lambda에 전달한다.
2. Lambda 서비스는 사용 가능한 실행 환경이 없으면 새 환경을 초기화한다.
3. 핸들러는 이벤트에서 Bucket과 Object Key를 읽어 이미지를 변환한다.
4. 결과를 별도 경로에 저장하고 로그와 지표를 남긴다.
5. 호출이 실패하면 이벤트 소스의 정책에 따라 재시도되거나 실패 대상으로 전달된다.

썸네일 결과 경로를 입력 Key로 결정하면 같은 이벤트가 다시 와도 동일한 결과를 덮어써 멱등성을 확보하기 쉽다.

---

# 실행 환경과 콜드 스타트

Lambda 호출에는 실행 환경을 새로 만드는 **초기화 단계**와 핸들러를 실행하는 **호출 단계**가 있다.

새 실행 환경에서 런타임과 애플리케이션을 초기화할 때 생기는 추가 지연을 콜드 스타트라고 한다.

Java는 JVM과 Spring 컨텍스트 초기화 때문에 작은 스크립트 런타임보다 콜드 스타트 영향이 커질 수 있다.

| 완화 방법 | 효과 | 주의점 |
|---|---|---|
| 의존성 축소 | 클래스 로딩과 초기화를 줄인다. | 필요한 기능까지 제거하지 않는다. |
| 지연 초기화 | 요청에 불필요한 Bean 생성을 피한다. | 첫 사용 지연을 측정한다. |
| SnapStart | 초기화된 Java 실행 환경의 스냅샷을 활용한다. | 런타임 지원과 초기화 시점의 고유값을 확인한다. |
| Provisioned Concurrency | 준비된 환경을 유지한다. | 유휴 비용과 필요한 동시성을 계산한다. |
| GraalVM 네이티브 이미지 | JVM 시작 부담을 낮출 수 있다. | 빌드와 리플렉션 호환성이 복잡해진다. |

---

# 동시성과 스로틀링

동시에 들어온 요청 수만큼 여러 실행 환경이 만들어질 수 있으며 이 동시 실행 수를 **Concurrency**라고 한다.

계정과 Region의 동시성 한도 또는 함수에 설정한 예약 동시성에 도달하면 추가 호출은 스로틀링된다.

SQS를 이벤트 소스로 사용하면 Batch Size, 동시성, Visibility Timeout을 함께 조정해 하류 데이터베이스가 감당할 수 있는 속도로 처리해야 한다.

---

# Lambda와 ECS Fargate 비교

| 기준 | Lambda | ECS Fargate |
|---|---|---|
| 최대 실행 | 한 호출은 최대 15분이다. | 장시간 실행에 적합하다. |
| 확장 단위 | 함수 호출 | 컨테이너 Task |
| 시작 지연 | 콜드 스타트가 있을 수 있다. | Task 시작 시간이 필요하다. |
| 배포물 | Zip 또는 컨테이너 이미지 | 컨테이너 이미지 |
| 적합한 작업 | API, 이벤트, 짧은 배치 | 상시 서버, 긴 작업, Consumer |
| 제어 수준 | 관리형 런타임 제약이 있다. | 런타임과 프로세스 제어가 넓다. |

요청이 지속적으로 많고 큰 Spring Boot 애플리케이션을 그대로 운영해야 한다면 Fargate가 구조와 비용 면에서 단순할 수 있다.

---

# 설정 예시

Java 17 함수를 생성하고 메모리와 제한 시간을 조정하는 CLI 예시는 다음과 같다.

```bash
aws lambda create-function \
  --function-name thumbnail-generator \
  --runtime java17 \
  --handler com.example.ThumbnailHandler::handleRequest \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --zip-file fileb://target/function.zip

aws lambda update-function-configuration \
  --function-name thumbnail-generator \
  --memory-size 1024 \
  --timeout 30
```

---

# Spring Boot / Java에서는 어떻게 쓰는가

단순한 이벤트 핸들러는 `RequestHandler<I, O>` 구현이 가장 직접적이다.

```java
public final class OrderCreatedHandler implements
        RequestHandler<OrderCreatedEvent, HandlerResult> {
    private final DynamoDbClient dynamoDbClient = DynamoDbClient.create();
    @Override
    public HandlerResult handleRequest(OrderCreatedEvent event, Context context) {
        context.getLogger().log("eventId=" + event.eventId());
        saveIfAbsent(event);
        return new HandlerResult(event.orderId(), "PROCESSED");
    }
    private void saveIfAbsent(OrderCreatedEvent event) {
        // eventId를 조건부 쓰기 Key로 사용해 중복 처리를 막는다.
    }
}
record OrderCreatedEvent(String eventId, String orderId) {}
record HandlerResult(String orderId, String status) {}
```

Spring Cloud Function은 `Function`, `Consumer`, `Supplier` Bean을 Lambda 어댑터와 연결해 비즈니스 로직을 테스트하기 쉽게 만든다.

```java
@Configuration
class OrderFunctionConfiguration {
    @Bean
    Function<OrderCreatedEvent, HandlerResult> processOrder(
            OrderService orderService
    ) {
        return event -> orderService.process(event);
    }
}
```

기존 Spring MVC 애플리케이션의 변경을 줄여야 하면 `AWSLambdaStreamHandler` 기반 Serverless Java Container를 검토할 수 있지만 전체 컨텍스트 초기화가 지연 시간과 메모리를 늘릴 수 있다.

**항상 낮은 지연이 필요한 API, WebSocket 장기 연결, 15분 초과 작업, 상시 실행 Consumer, 로컬 메모리 상태에 의존하는 서비스에는 Lambda를 쓰지 말아야 한다.**

---

# 실무에서는 어떻게 사용할까

| 작업 | 이벤트 소스 | 핵심 설계 |
|---|---|---|
| 이미지 변환 | S3 | 중복 알림과 재귀 호출을 방지한다. |
| 주문 API | API Gateway | 입력 검증과 응답 형식을 고정한다. |
| 메시지 소비 | SQS | 부분 Batch 실패와 DLQ를 구성한다. |
| 예약 정리 | EventBridge Scheduler | 실행 이력과 중복 실행을 기록한다. |
| 데이터 변경 후처리 | DynamoDB Streams | 순서와 재처리 범위를 이해한다. |

함수는 한 가지 변경 이유를 갖도록 작게 유지하되 지나치게 잘게 나눠 네트워크 호출과 운영 대상을 폭증시키지 않는다.

---

# 장애 사례

콜드 스타트가 API 응답 지연을 일으키면 초기화 시간과 핸들러 시간을 분리해 측정하고 SnapStart 또는 Provisioned Concurrency 적용 여부를 판단한다.

갑작스러운 이벤트 증가로 동시 실행 한도에 걸리면 스로틀링과 재시도가 겹쳐 하류 서비스까지 과부하될 수 있다.

Lambda를 VPC에 연결한 뒤 NAT Gateway나 필요한 VPC Endpoint가 없으면 외부 결제 API, 패키지 저장소, AWS 공개 엔드포인트 호출이 실패할 수 있다.

각 실행 환경이 RDS 커넥션 풀을 만들면 동시 확장 시 데이터베이스 연결이 고갈되므로 RDS Proxy, 작은 Pool, 동시성 제한을 검토한다.

비동기 이벤트가 재시도되는데 멱등성 처리가 없으면 결제, 쿠폰, 알림이 중복 수행될 수 있다.

DLQ나 On-failure Destination이 없으면 최대 재시도 후 실패한 이벤트를 복구하기 어려워진다.

---

# 주의할 점

함수의 임시 저장 공간은 실행 환경에 종속되므로 영구 저장소로 사용하지 않는다.

Timeout을 하류 HTTP 클라이언트 Timeout보다 짧게 두면 작업 중 함수가 종료될 수 있고 반대로 너무 길게 두면 장애 감지가 늦어진다.

---

# 비용과 성능 고려사항

Lambda 비용은 요청 수, 실행 시간, 할당 메모리의 영향을 받으며 Provisioned Concurrency, 로그, 데이터 전송 비용은 별도로 고려한다.

메모리를 늘리면 CPU도 함께 증가하므로 CPU 집약 작업은 더 빨리 끝나 총 실행 비용이 낮아질 수 있다.

Power Tuning 같은 측정 방식으로 실제 입력의 지연 시간과 비용을 비교하고 감으로 메모리를 정하지 않는다.

함수 안에서 매번 SDK 클라이언트와 Spring 컨텍스트를 생성하면 실행 시간과 연결 수가 늘어나므로 안전한 객체는 실행 환경에서 재사용한다.

---

# 기억해야 할 내용

- Lambda는 이벤트가 발생할 때 코드를 함수 단위로 실행한다.
- 한 번의 실행 시간은 최대 15분이므로 장시간 작업에는 적합하지 않다.
- Java 콜드 스타트는 의존성 축소, SnapStart, Provisioned Concurrency 등으로 완화한다.
- 동시성 증가는 하류 데이터베이스와 외부 API의 용량을 함께 압박한다.
- 비동기 이벤트와 메시지는 중복될 수 있으므로 멱등성을 설계한다.
- VPC 연결 시 외부 통신 경로와 VPC Endpoint 구성을 확인한다.
- DLQ, 실패 대상, 로그, 지표, 알람을 함께 구성해야 운영할 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 Lambda를 HTTP API로 공개하는 **API Gateway**를 학습한다.

라우팅, 인증, Lambda Proxy Integration, 제한 시간과 오류 응답 설계를 자세히 알아본다.

