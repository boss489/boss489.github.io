---
title: "Chapter 07. ECS Deployment"
permalink: /aws-backend/part-08/07-ecs-deployment/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 07. ECS Deployment
## 새 컨테이너 이미지를 서비스에 반영하기
> **학습 목표**
>
> - ECS 배포 흐름을 설명할 수 있다.
> - ALB Health Check가 배포에 미치는 영향을 이해한다.
> - Rolling Deployment의 기본 동작을 설명할 수 있다.
> - Rolling Update와 Blue/Green Deployment를 비교할 수 있다.
> - Circuit Breaker와 graceful shutdown을 이용해 실패 영향을 줄일 수 있다.
---
# 왜 ECS 배포 전략이 필요한가
EC2에 JAR를 직접 덮어쓰고 프로세스를 재시작하면 애플리케이션이 시작되는 동안 요청을 받을 서버가 없어 다운타임이 발생할 수 있다.

여러 서버를 순서대로 배포하더라도 새 버전의 Health Check를 확인하지 않고 기존 서버를 종료하면 전체 Capacity가 줄거나 오류가 사용자에게 전달된다.

배포 실패 후 이전 JAR와 설정을 찾아 수동으로 되돌리는 방식은 복구 시간이 길고 어떤 버전으로 돌아갔는지도 불명확하다.

ECS Service는 새 Task를 먼저 시작하고 정상 여부를 확인한 뒤 기존 Task를 줄이는 방식으로 버전을 교체할 수 있다.

안전한 배포는 Image Push만으로 끝나지 않으며 Task Definition, Service 배포 설정, ALB Health Check와 종료 유예 시간을 함께 설계해야 한다.
---
# ECS Deployment란?
ECS Deployment는 Service가 사용하는 Task Definition Revision 또는 배포 구성을 변경해 기존 Task를 새 Task로 교체하는 과정이다.

새 Image를 배포하려면 고유 Tag나 Digest를 ECR에 Push하고 이를 참조하는 새 Task Definition Revision을 등록하는 방식이 가장 명확하다.

`latest` Tag만 덮어쓰고 같은 Revision을 재사용하면 배포 이력에서 실제 Image 내용을 확인하기 어렵다.

Service는 Desired Count와 `minimumHealthyPercent`, `maximumPercent` 범위 안에서 새 Task와 기존 Task 수를 조절한다.
---
# 전체 배포 흐름
```text
Developer
   │ Git Push
   ▼
CI Test & Docker Build
   │
   ▼
ECR Push: git-a1b2c3d
   │
   ▼
Task Definition Revision :13
   │
   ▼
ECS Service Deployment
   ├── New Task ── Health Check ──┐
   └── Old Task ── Deregistration │
                                  ▼
                           ALB Target Group
```
![ECS deployment](/assets/aws-backend/ecs-deployment.png)
1. CI가 테스트를 통과한 Commit으로 Image를 빌드한다.
2. Image에 Git SHA 같은 고유 Tag를 붙여 [ECR](/aws-backend/part-08/04-ecr/)에 Push한다.
3. 새 Image URI를 포함한 Task Definition Revision을 등록한다.
4. [ECS Service](/aws-backend/part-08/06-task-service/)를 새 Revision으로 업데이트한다.
5. ECS가 배포 설정 범위 안에서 새 Task를 시작한다.
6. ALB Health Check를 통과한 새 Task가 트래픽을 받는다.
7. 기존 Task는 Target에서 제외된 뒤 종료 신호를 받아 정상 종료한다.
---
# Rolling Update
Rolling Update는 하나의 Service 안에서 새 Task를 점진적으로 늘리고 기존 Task를 줄이는 기본 배포 방식이다.
| 설정 | 의미 | 잘못 설정했을 때 |
|---|---|---|
| `desiredCount` | 평상시 유지할 Task 수 | 처리 Capacity 부족 또는 과다 |
| `minimumHealthyPercent` | 배포 중 유지할 최소 정상 비율 | 너무 낮으면 Capacity 급감 |
| `maximumPercent` | 배포 중 허용할 최대 Task 비율 | 여유 Capacity가 없으면 Pending |
| Health Check Grace Period | 시작 후 Health 판정 유예 | 짧으면 느린 앱이 반복 종료 |
Desired Count가 4이고 최대 비율이 200%이면 설정상 배포 중 최대 8개 Task까지 허용될 수 있지만 실제 Capacity와 Fargate 할당량도 충분해야 한다.

최소 정상 비율과 최대 비율은 반올림 규칙과 Desired Count에 따라 실제 Task 수가 달라질 수 있으므로 배포 전 계산과 검증이 필요하다.
---
# Blue/Green Deployment
Blue/Green은 기존 환경인 Blue와 새 환경인 Green을 동시에 준비한 뒤 트래픽을 전환하는 방식이다.
| 기준 | Rolling Update | Blue/Green |
|---|---|---|
| 교체 방식 | 같은 Service에서 점진 교체 | 두 Task Set 사이 트래픽 전환 |
| 추가 Capacity | 설정 범위만큼 필요 | 두 환경 동시 운영 Capacity 필요 |
| 검증 | Health Check 중심 | Test Listener 등 사전 검증 가능 |
| 롤백 | 이전 Revision으로 재배포 | 트래픽을 Blue로 되돌리기 용이 |
| 복잡도 | 상대적으로 낮음 | 구성 요소와 자동화가 더 많음 |
Blue/Green은 위험한 변경의 검증과 빠른 트래픽 복구에 유리하지만 두 환경을 동시에 유지하는 시간 동안 자원 사용이 증가한다.

DB Schema처럼 두 버전이 동시에 접근하는 자원은 어떤 배포 방식에서도 하위 호환성을 고려해야 한다.
---
# ALB Health Check와 트래픽 전환
ALB는 Target Group Health Check를 통과한 Task에만 일반 요청을 전달한다.

Health Check 경로가 실제 애플리케이션에 없거나 인증이 필요하거나 Timeout보다 초기화가 오래 걸리면 정상 Task도 Unhealthy로 판정된다.

Spring Boot Actuator의 Readiness는 요청 수신 가능성을 나타내도록 구성하고 외부 의존성 장애를 어떤 상태로 반영할지 신중히 결정한다.

모든 일시적인 DB 지연을 즉시 Unhealthy로 만들면 정상 Task가 한꺼번에 Target에서 제외되어 장애가 커질 수 있다.
---
# 종료와 Deregistration Delay
기존 Task를 종료하기 전 ALB Target에서 등록 해제하고 진행 중인 연결을 처리할 시간을 줘야 한다.
```text
ALB Target Deregistration
          │
          ▼
Deregistration Delay
          │
          ▼
ECS sends stop signal
          │
          ▼
Spring Boot graceful shutdown
          │
          ▼
Task stopped
```
Deregistration Delay를 요청 처리 시간보다 지나치게 짧게 두면 배포 중 장기 요청과 연결이 끊길 수 있다.

반대로 지나치게 길면 배포와 Scale-in 완료가 늦어질 수 있으므로 실제 최대 요청 시간과 연결 특성을 기준으로 조정한다.
---
# 실습: 새 Revision 배포
새 Task Definition을 등록한 뒤 반환된 ARN을 Service Update에 사용한다.
```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json

aws ecs update-service \
  --cluster production-app \
  --service order-api \
  --task-definition order-api:13

aws ecs wait services-stable \
  --cluster production-app \
  --services order-api
```
같은 Task Definition을 사용하되 Task를 다시 시작해야 할 때는 강제 새 배포를 요청할 수 있다.
```bash
aws ecs update-service \
  --cluster production-app \
  --service order-api \
  --force-new-deployment
```
`--force-new-deployment`는 변경 가능한 Tag 운영의 추적 문제를 해결하지 않으므로 정상 배포에서는 새 Revision과 고유 Image 식별자를 사용한다.

---
# Circuit Breaker와 롤백
Deployment Circuit Breaker는 새 배포가 정상 상태에 도달하지 못하면 실패로 판정하고 설정에 따라 이전 완료 배포로 롤백할 수 있다.

이전 Task Definition Revision과 Image가 ECR Lifecycle Policy로 삭제되지 않도록 실제 롤백 보존 기간을 맞춘다.
---
# Spring Boot에서는 어떻게 쓰는가
Spring Boot는 종료 신호를 받았을 때 진행 중인 요청을 마칠 수 있도록 graceful shutdown을 활성화한다.
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info
```
Task Definition은 Profile, Port와 종료 시간을 명시한다.
```json
{
  "name": "order-api",
  "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:git-a1b2c3d",
  "portMappings": [
    {
      "containerPort": 8080,
      "protocol": "tcp"
    }
  ],
  "environment": [
    {
      "name": "SPRING_PROFILES_ACTIVE",
      "value": "prod"
    }
  ],
  "stopTimeout": 45
}
```
Spring 종료 유예 시간, ECS `stopTimeout`, ALB Deregistration Delay는 독립된 값이지만 하나의 요청 종료 흐름으로 함께 테스트해야 한다.
---
# 실무에서는 어떻게 사용할까
CI는 Test, Image Build, Scan, ECR Push, Task Definition 등록, Service Update 순서로 각 산출물의 식별자를 다음 단계에 전달한다.

운영 배포 전에 스테이징에서 같은 Image Digest를 검증하고 운영에서는 Image를 다시 빌드하지 않는다.

DB Migration은 Expand and Contract 같은 하위 호환 전략을 사용해 구버전과 신버전 Task가 동시에 실행되는 시간을 견디게 한다.
---
# 장애 사례
| 증상 | 주요 원인 | 확인 지점 |
|---|---|---|
| 새 Task 반복 종료 | Health 경로, Port, Grace Period 오류 | Target Health, Service Event |
| 배포가 `PENDING` | 배포용 추가 CPU, 메모리, IP 부족 | Cluster Capacity, Subnet |
| 종료 코드 `137` | 메모리 한도 초과 | Task 지표, JVM Heap |
| 배포 중 요청 끊김 | 짧은 Deregistration Delay 또는 `stopTimeout` | ALB와 앱 로그 |
| 롤백 Image Pull 실패 | ECR Lifecycle Policy로 Image 삭제 | Repository와 Digest |
| 새 코드가 보이지 않음 | `latest` 재사용과 Revision 혼동 | 실행 Task의 Image Digest |
Task가 계속 재시작되는 경우 먼저 Service Event, Stopped Reason, Target Health Reason, Container 로그 순서로 범위를 좁힌다.
---
# 주의할 점
- `latest` 대신 고유 Tag 또는 Digest와 새 Task Definition Revision을 사용한다.
- Health Check 경로, 응답 코드, Interval, Timeout과 Grace Period를 함께 검증한다.
- 배포 중 추가 Task를 실행할 Capacity와 사설 IP를 확보한다.
- Deregistration Delay를 실제 요청 처리 시간보다 무조건 짧게 두지 않는다.
- graceful shutdown과 ECS `stopTimeout`을 부하 환경에서 검증한다.
- 롤백에 필요한 Revision과 ECR Image를 보존한다.
- DB 변경은 구버전과 신버전의 동시 실행을 고려한다.
---
# 비용과 성능 고려사항
Rolling Update는 배포 중 `maximumPercent`만큼 Task가 추가되어 Fargate vCPU와 메모리 사용 또는 EC2 Capacity 요구가 일시적으로 늘어난다.

Blue/Green은 Blue와 Green을 동시에 유지하므로 전환 기간의 컴퓨팅과 로그 수집 자원이 더 필요하다.

ECR Image 저장량, Private Subnet에서 NAT Gateway를 통한 Pull, CloudWatch Logs와 배포 모니터링도 비용 요소이다.

Health Check 간격을 짧게 하면 전환이 빨라질 수 있지만 요청 수와 일시 오류에 대한 민감도도 높아지므로 안정성과 속도를 함께 측정한다.
---
# 기억해야 할 내용
- ECS 배포는 새 Revision의 Task를 시작하고 정상 확인 후 기존 Task를 교체하는 과정이다.
- Rolling Update는 점진 교체하고 Blue/Green은 두 환경 사이 트래픽을 전환한다.
- ALB Health Check를 통과한 Task만 요청을 받아야 한다.
- `latest` 대신 Image Digest와 Task Definition Revision으로 배포를 추적한다.
- Circuit Breaker는 실패 배포의 감지와 롤백을 도울 수 있다.
- Deregistration Delay, `stopTimeout`, graceful shutdown을 함께 설계한다.
- 배포 중 추가 Capacity와 롤백 Image 보존이 필요하다.
---
# 다음 Chapter
다음 Chapter는 **Chapter 08. Part 8 Summary**이다.

Docker Image 빌드부터 ECR 저장, ECS Cluster와 Service 운영, 안전한 배포까지의 전체 흐름을 하나의 아키텍처로 정리한다.

이후 Part 9에서는 같은 컨테이너 배치, 복구와 배포 문제를 Kubernetes와 EKS가 어떻게 해결하는지 비교한다.

