---
title: "Chapter 06. Step Functions"
permalink: /aws-backend/part-10/06-step-functions/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Step Functions
## 여러 작업의 순서와 실패를 상태 머신으로 관리하기

> **학습 목표**
>
> - Step Functions의 역할을 설명할 수 있다.
> - 여러 Lambda 작업을 순서대로 연결하는 방식을 이해한다.
> - 재시도, 분기, 보상 처리의 필요성을 설명할 수 있다.
> - Standard와 Express Workflow의 선택 기준을 설명할 수 있다.

---

# 왜 Step Functions가 필요한가

주문 완료 이후 결제 승인, 재고 차감, 배송 요청, 알림 발송을 순서대로 처리해야 한다고 가정해 보자.

하나의 Lambda가 다음 Lambda를 직접 호출하면 현재 단계, 재시도 횟수, 실패 원인을 여러 함수의 코드와 로그에서 추적해야 한다.

이 흐름을 애플리케이션 코드의 중첩된 `try-catch`와 호출 체인으로 구현하면 업무 절차와 장애 처리가 뒤섞인다.

Step Functions는 **순서, 분기, 병렬 실행, 대기, 재시도, 실패, 보상 흐름을 상태 머신으로 명시**하여 장기 업무의 현재 상태를 관리한다.

---

# Step Functions란?

Step Functions는 여러 작업과 AWS 서비스를 Amazon States Language로 정의한 상태 머신에 연결하는 관리형 워크플로우 서비스이다.

Lambda, ECS Task, Batch, SNS, SQS 등과 연결할 수 있다.

---

# 주요 State

| State | 역할 |
|---|---|
| Task | Lambda나 AWS 서비스 작업을 실행한다. |
| Choice | 입력 조건에 따라 다음 State를 선택한다. |
| Parallel | 여러 Branch를 병렬로 실행한다. |
| Map | 배열의 각 항목에 같은 흐름을 적용한다. |
| Wait | 지정 시각 또는 기간까지 실행을 대기한다. |
| Fail | 실행을 실패로 종료한다. |

---

# 동작 흐름

결제부터 배송까지의 상태 전이는 다음과 같다.

```
OrderCompleted
      │
      ▼
[결제 승인]
  │ 성공       │ 실패
  ▼            ▼
[재고 차감]   [주문 실패]
  │ 성공
  ▼
[배송 요청]
  │ 성공       │ 최종 실패
  ▼            ▼
[알림 발송]   [재고 복구]
  │            │
  ▼            ▼
[완료]       [결제 취소] ──▶ [보상 완료]
```

1. 주문 완료 이벤트가 상태 머신 실행을 시작한다.
2. 결제 승인 Task가 성공하면 재고 차감으로 이동하고 업무 오류면 실패 상태로 분기한다.
3. 일시적인 재고 서비스 오류에는 제한된 Backoff 재시도를 적용한다.
4. 배송 요청이 최종 실패하면 재고 복구와 결제 취소를 역순으로 실행한다.
5. 각 실행 이력과 State 입출력을 확인해 실패 지점과 보상 결과를 추적한다.

---

# Standard와 Express Workflow 비교

| 기준 | Standard | Express |
|---|---|---|
| 적합한 실행 | 오래 지속되고 감사가 필요한 흐름 | 짧고 호출량이 많은 흐름 |
| 실행 이력 | Step Functions에 상세 이력을 유지한다. | 로그와 모니터링 구성을 적극 활용한다. |
| 실행 의미 | 워크플로우 실행의 Exactly-once 모델에 초점이 있다. | At-least-once 또는 동기 Express 특성을 이해해야 한다. |
| 과금 관점 | 상태 전이 수가 중심이다. | 실행 횟수, 시간, 메모리 사용이 중심이다. |
---

# 재시도와 오류 처리

`Retry`는 네트워크 오류와 Throttling처럼 일시적인 실패에 제한적으로 사용하고 입력 검증 실패 같은 영구 오류는 즉시 `Catch`로 보낸다.

---

# 설정 예시

다음 ASL 정의는 결제 Task를 재시도하고 최종 실패 시 결제를 취소하는 간단한 흐름이다.

```json
{
  "StartAt": "ApprovePayment",
  "States": {
    "ApprovePayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "approve-payment",
        "Payload.$": "$"
      },
      "Retry": [{
        "ErrorEquals": ["Lambda.ServiceException"],
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }],
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelPayment"
      }],
      "Next": "RequestDelivery"
    },
    "RequestDelivery": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {"FunctionName": "request-delivery", "Payload.$": "$"},
      "End": true
    },
    "CancelPayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {"FunctionName": "cancel-payment", "Payload.$": "$"},
      "Next": "WorkflowFailed"
    },
    "WorkflowFailed": {
      "Type": "Fail",
      "Cause": "보상 완료"
    }
  }
}
```

정의 파일로 Standard 상태 머신을 만드는 CLI 예시는 다음과 같다.

```bash
aws stepfunctions create-state-machine \
  --name order-workflow \
  --definition file://order-workflow.asl.json \
  --role-arn arn:aws:iam::123456789012:role/step-functions-role \
  --type STANDARD
```

---

# Spring Boot / Java에서는 어떻게 쓰는가

Spring Boot 애플리케이션은 AWS SDK for Java v2의 `SfnClient`로 상태 머신 실행을 시작할 수 있다.

```java
@Service
class OrderWorkflowService {
    private final SfnClient sfnClient;
    private final ObjectMapper objectMapper;
    OrderWorkflowService(
            SfnClient sfnClient, ObjectMapper objectMapper
    ) {
        this.sfnClient = sfnClient;
        this.objectMapper = objectMapper;
    }
    String start(OrderWorkflowInput input) throws JsonProcessingException {
        StartExecutionResponse response = sfnClient.startExecution(
                StartExecutionRequest.builder()
                        .stateMachineArn(System.getenv("STATE_MACHINE_ARN"))
                        .name(input.workflowId())
                        .input(objectMapper.writeValueAsString(input))
                        .build()
        );
        return response.executionArn();
    }
}

record OrderWorkflowInput(String workflowId, String orderId) {}
```

Java 콜드 스타트가 전체 워크플로우 지연에 누적되면 SnapStart, Provisioned Concurrency, 의존성 축소, GraalVM 네이티브 이미지를 선택적으로 적용한다.

**하나의 ACID 트랜잭션으로 끝나는 단순 CRUD, 초저지연 요청, 지속적인 대량 스트림 처리에는 Step Functions와 Lambda 조합을 사용하지 않는 편이 낫다.**

---

# 실무에서는 어떻게 사용할까

| 시나리오 | State 구성 | 핵심 설계 |
|---|---|---|
| 주문 Saga | Task, Choice, Catch | 보상 순서와 멱등성을 정의한다. |
| 승인 절차 | Task, Wait, Callback | 승인 기한과 취소를 관리한다. |
| 파일 처리 | Map, Parallel | 항목별 실패와 동시성을 제한한다. |
| 장기 배치 | ECS/Batch Task | 작업 상태를 동기 Integration으로 추적한다. |
| 장애 자동화 | Choice, AWS SDK Integration | 자동 조치의 안전장치를 둔다. |

---

# 장애 사례

모든 오류에 같은 재시도를 적용하면 입력 오류도 반복 호출되어 상태 전이와 하류 부하만 증가한다.

보상 Lambda에 멱등성이 없으면 재시도 중 결제 취소나 재고 복구가 여러 번 수행될 수 있다.

---

# 주의할 점

워크플로우 입력에 비밀번호, Token, 불필요한 개인정보를 넣지 않고 민감값은 실행 시 안전한 저장소에서 조회한다.

---

# 비용과 성능 고려사항

Standard Workflow는 상태 전이 수가 비용의 중심이므로 지나치게 세분화된 Pass State와 불필요한 Polling을 줄인다.

Parallel과 Map의 동시성을 크게 높이면 처리 시간은 줄지만 Lambda 동시성, 외부 API 제한, RDS 커넥션을 고갈시킬 수 있다.

---

# 기억해야 할 내용

- Step Functions는 여러 작업의 순서와 상태를 상태 머신으로 표현한다.
- Task, Choice, Parallel, Map, Wait, Succeed, Fail State를 목적에 맞게 사용한다.
- 일시적 오류만 제한적으로 재시도하고 영구 오류는 Catch로 분기한다.
- 분산 작업은 자동 Rollback되지 않으므로 보상 트랜잭션을 설계한다.
- 재시도되는 Task는 결제와 배송 같은 외부 작업에서도 멱등성을 가져야 한다.
- Standard는 상태 전이 수, Express는 실행 시간 기반 과금 관점이 다르다.
- 실행 이력, 실패 알람, 수동 복구 절차까지 있어야 운영 가능한 워크플로우가 된다.

---

# 다음 Chapter
다음 Chapter는 **Chapter 07. Part 10 Summary**이다.

Lambda, API Gateway, EventBridge, DynamoDB, Step Functions를 하나의 서버리스 아키텍처로 연결하고 서비스 선택 기준과 장애 대응 원칙을 정리한다.

