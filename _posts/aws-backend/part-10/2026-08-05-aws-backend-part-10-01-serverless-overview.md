---
title: "Chapter 01. Serverless Overview"
permalink: /aws-backend/part-10/01-serverless-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Serverless Overview
## 서버 운영보다 비즈니스 흐름에 집중하는 실행 모델

> **학습 목표**
>
> - Serverless의 의미를 설명할 수 있다.
> - Lambda, API Gateway, EventBridge, DynamoDB, Step Functions의 관계를 이해한다.
> - 서버리스가 적합한 경우와 그렇지 않은 경우를 구분할 수 있다.
> - 비용, 장애, 운영 관점의 설계 기준을 설명할 수 있다.

---

# 왜 Serverless가 필요한가

하루에 두 번 실행되어 5분 만에 끝나는 정산 배치를 위해 EC2를 24시간 켜 두면 실제 작업 시간보다 대기 시간에 더 많은 자원을 사용한다.

트래픽이 거의 없다가 특정 이벤트 직후 급증하는 API도 최대 부하에 맞춰 서버를 항상 유지하면 유휴 비용과 운영 부담이 커진다.

S3에 이미지가 업로드될 때 썸네일을 만들거나 매일 새벽 보고서를 생성하는 작업은 요청이 발생한 순간에만 실행 환경이 필요하다.

서버리스는 이런 **간헐적이고 이벤트 중심인 작업에 실행 환경을 자동으로 할당하고 사용한 만큼만 자원을 소비**하게 한다.

다만 서버를 직접 관리하지 않는 대신 실행 한도, 재시도, 권한, 관찰 가능성 같은 분산 시스템 설계를 애플리케이션이 책임져야 한다.

---

# Serverless란?

Serverless는 서버가 없다는 뜻이 아니라 서버 프로비저닝, 운영체제 패치, 용량 확장 같은 인프라 운영을 클라우드 제공자가 담당하는 실행 모델이다.

AWS가 실행 환경을 관리하고 사용자는 함수, 이벤트, 권한, 데이터 흐름을 설계한다.

![Serverless flow](/assets/aws-backend/serverless-flow.png)

---

# Part 10 전체 지도

| Chapter | 서비스 | 핵심 역할 |
|---|---|---|
| [02. Lambda](/aws-backend/part-10/02-lambda/) | 함수 실행 | 이벤트가 발생할 때 코드를 실행한다. |
| [03. API Gateway](/aws-backend/part-10/03-api-gateway/) | API 진입점 | HTTP 요청을 인증하고 백엔드로 전달한다. |
| [04. EventBridge](/aws-backend/part-10/04-eventbridge/) | 이벤트 라우팅 | 이벤트를 규칙에 따라 여러 대상으로 전달한다. |
| [05. DynamoDB](/aws-backend/part-10/05-dynamodb/) | 서버리스 데이터 저장 | 조회 패턴에 맞춘 Key 기반 데이터를 저장한다. |
| [06. Step Functions](/aws-backend/part-10/06-step-functions/) | 워크플로우 | 여러 작업의 순서, 분기, 재시도, 실패를 관리한다. |

Chapter 05는 Key 설계부터 Enhanced Client와 실전 문제까지 **7개 하위 챕터**로 나뉜다.

---

# 동작 흐름

동기 API와 비동기 후처리를 함께 사용하는 대표 구조는 다음과 같다.

```
Client
  │ HTTPS
  ▼
API Gateway ──▶ Lambda ──▶ DynamoDB
                    │
                    └──▶ EventBridge
                           ├──▶ 알림 Lambda
                           ├──▶ 정산 Target
                           └──▶ Step Functions
                                  ├──▶ 결제 확인
                                  └──▶ 배송 요청
```

1. 클라이언트 요청은 API Gateway에서 라우팅, 인증, 요청 제한을 거친다.
2. Lambda는 비즈니스 로직을 실행하고 DynamoDB에 상태를 저장한다.
3. 주문 완료 같은 후속 사실은 EventBridge에 이벤트로 발행한다.
4. 각 Rule은 이벤트를 알림, 정산, 워크플로우 같은 독립된 Target으로 전달한다.
5. Step Functions는 여러 단계가 있는 업무의 상태와 실패 처리를 관리한다.

---

# 서버 기반과 서버리스 비교

| 기준 | 서버 기반 EC2/ECS | 서버리스 |
|---|---|---|
| 실행 단위 | 인스턴스 또는 컨테이너 | 요청, 함수, 상태 전이 |
| 확장 | 용량과 정책을 구성한다. | 서비스가 요청에 따라 확장한다. |
| 유휴 자원 | 최소 실행 용량이 존재한다. | 서비스에 따라 0에 가깝게 축소한다. |
| 실행 제약 | 비교적 자유롭다. | 시간, 크기, 동시성 한도가 있다. |
| 운영 책임 | 런타임과 배포 환경 관리가 크다. | 코드, 권한, 이벤트 설계에 집중한다. |
| 적합한 부하 | 지속적이고 예측 가능한 부하 | 간헐적이거나 급변하는 이벤트 부하 |
| 디버깅 | 단일 프로세스 추적이 비교적 쉽다. | 분산 추적과 상관관계 ID가 중요하다. |

서버리스가 항상 더 저렴하거나 단순한 것은 아니며 지속적인 고부하와 복잡한 네트워크 요구에서는 컨테이너가 더 적합할 수 있다.

---

# Lambda와 ECS Fargate 선택 기준

| 질문 | Lambda가 유리한 경우 | ECS Fargate가 유리한 경우 |
|---|---|---|
| 실행 시간 | 짧고 15분 이내이다. | 장시간 또는 상시 실행한다. |
| 호출 패턴 | 이벤트가 간헐적이다. | 트래픽이 지속적이다. |
| 런타임 제어 | 관리형 환경으로 충분하다. | 프로세스와 OS 패키지 제어가 필요하다. |
| 연결 방식 | 짧은 요청과 이벤트 처리이다. | 긴 연결이나 상시 소비자가 필요하다. |
| 배포 단위 | 작은 독립 기능이다. | 큰 Spring Boot 애플리케이션이다. |

기존 Spring Boot 모놀리스를 함수 단위로 억지로 분리하기보다 변경 이유와 부하 특성이 분명한 작업부터 Lambda로 옮기는 편이 안전하다.

---

# 설정 예시

다음 명령은 함수가 사용할 IAM 역할이 준비되어 있다는 전제에서 Java 17 Lambda 함수를 생성하는 형태이다.

```bash
aws lambda create-function \
  --function-name thumbnail-generator \
  --runtime java17 \
  --handler com.example.ThumbnailHandler::handleRequest \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --zip-file fileb://function.zip
```

운영 환경에서는 IaC 도구로 함수, 권한, 이벤트 소스를 함께 버전 관리하는 것이 좋다.

---

# Spring Boot / Java에서는 어떻게 쓰는가

작은 함수는 AWS Lambda Java Core의 `RequestHandler<I, O>`를 구현하면 의존성과 초기화 비용을 줄일 수 있다.

```java
public final class ThumbnailHandler
        implements RequestHandler<S3Event, String> {

    @Override
    public String handleRequest(S3Event event, Context context) {
        String objectKey = event.getRecords().get(0)
                .getS3().getObject().getKey();
        context.getLogger().log("resize target=" + objectKey);
        return objectKey;
    }
}
```

기존 Spring 생태계의 함수형 Bean을 재사용하려면 Spring Cloud Function을 사용할 수 있고, Spring Boot 웹 애플리케이션 호환이 중요하면 Serverless Java Container의 `AWSLambdaStreamHandler` 방식을 검토할 수 있다.

Spring 컨텍스트 초기화는 JVM 콜드 스타트를 키울 수 있으므로 의존성을 줄이고 초기화를 지연하며 Java 런타임의 Lambda SnapStart 또는 Provisioned Concurrency를 요구 수준에 맞게 검토한다.

GraalVM 네이티브 이미지는 시작 시간을 줄일 수 있지만 빌드 복잡성, 리플렉션 설정, 라이브러리 호환성을 검증해야 한다.

**항상 낮은 지연이 필요한 API, 긴 연결, 15분을 넘는 작업, 큰 메모리 내 상태, 지속적인 고부하를 가진 Spring Boot 서비스에는 Lambda를 사용하지 않는 편이 낫다.**

---

# 실무에서는 어떻게 사용할까

| 시나리오 | 구성 | 설계 핵심 |
|---|---|---|
| 이미지 썸네일 | S3 → Lambda | 원본 Key를 멱등성 Key로 사용한다. |
| 예약 배치 | EventBridge Scheduler → Lambda | 중복 실행과 지연 실행을 허용한다. |
| 주문 API | API Gateway → Lambda → DynamoDB | 인증, 제한, 조건부 쓰기를 적용한다. |
| 주문 후처리 | EventBridge → 여러 Target | 소비자 실패를 서로 격리한다. |
| 결제 흐름 | Step Functions → Lambda/ECS | 재시도와 보상 상태를 명시한다. |

Part 9에서 다룬 Kubernetes가 장기 실행 컨테이너를 오케스트레이션한다면 Part 10은 이벤트와 관리형 서비스로 실행 단위를 더 작게 나누는 관점이다.

---

# 장애 사례와 주의할 점

첫 요청이 콜드 스타트를 만나면 평소보다 응답이 늦어질 수 있으므로 지연 시간 분포를 측정하고 필요한 함수에만 초기화 완화 전략을 적용한다.

동시 실행 한도에 도달하면 Lambda 호출이 스로틀링되므로 예약 동시성, 이벤트 소스 처리량, 재시도 폭증을 함께 관찰한다.

Lambda를 VPC의 Private Subnet에 연결한 뒤 NAT Gateway나 VPC Endpoint를 구성하지 않으면 외부 API와 일부 AWS 공개 엔드포인트 호출이 실패할 수 있다.

함수 인스턴스가 동시에 RDS 연결을 만들면 커넥션 풀이 고갈될 수 있으므로 연결 재사용, 동시성 제한, RDS Proxy를 검토한다.

비동기 이벤트는 재시도 과정에서 중복 전달될 수 있으므로 이벤트 ID나 업무 Key를 이용한 멱등성 처리가 필요하다.

DLQ 또는 실패 대상을 설정하지 않고 로그 보존과 알람도 구성하지 않으면 처리되지 않은 이벤트를 뒤늦게 복구하기 어렵다.

---

# 비용과 성능 고려사항

Lambda 비용은 개념적으로 **요청 수 × 실행 시간 × 할당 메모리**의 영향을 받으며 연결된 API Gateway, 로그, 데이터 전송 비용도 함께 본다.

Lambda는 메모리를 늘리면 CPU 자원도 함께 늘어나므로 실행 시간이 크게 줄면 오히려 전체 비용이 감소할 수 있어 실제 입력으로 측정해야 한다.

DynamoDB는 읽기와 쓰기 용량, 저장량, 인덱스와 요청 패턴이 비용을 좌우한다.

EventBridge와 API Gateway는 이벤트 또는 요청 규모에 따라 비용이 증가하고 Step Functions는 Standard의 상태 전이 수와 Express의 실행 횟수·시간 기반 과금 차이를 이해해야 한다.

CloudWatch 로그를 과도하게 남기면 비용과 검색 부담이 커지므로 구조화 로그, 보존 기간, 샘플링 정책을 정한다.

---

# 기억해야 할 내용

- Serverless는 서버가 없는 구조가 아니라 서버 운영 책임을 AWS에 위임하는 모델이다.
- Lambda는 실행, API Gateway는 HTTP 진입점, EventBridge는 이벤트 라우팅을 담당한다.
- DynamoDB는 Key 중심 데이터 저장소이고 Step Functions는 여러 작업의 상태를 관리한다.
- 간헐적 이벤트에는 유리하지만 지속적인 고부하와 긴 실행에는 컨테이너가 나을 수 있다.
- 재시도되는 이벤트를 전제로 멱등성, DLQ, 관찰 가능성을 설계해야 한다.
- 비용은 함수만이 아니라 API, 로그, 데이터 전송, 상태 전이를 함께 계산해야 한다.
- 관리할 서버가 줄어도 분산 시스템의 복잡성은 사라지지 않는다.

---

# 다음 Chapter

다음 Chapter에서는 서버리스 실행 계층의 핵심인 **Lambda**를 학습한다.

이벤트가 함수를 실행하는 과정과 Java 핸들러, 콜드 스타트, 동시성, 장애 대응을 자세히 알아본다.

- 서버 프로비저닝이 필요 없다.
- 사용량 기반 과금이다.
- 이벤트 기반 처리에 적합하다.
- 작은 기능을 빠르게 만들 수 있다.

---

# 한계

- 실행 시간 제한
- Cold Start
- 로컬 디버깅 어려움
- 복잡한 흐름 추적 어려움
- 상태 관리 제약

---

# 기억해야 할 내용

Serverless는 운영 부담을 줄이지만 설계 복잡도를 없애지는 않는다.


