---
title: "Chapter 06. Task and Service"
permalink: /aws-backend/part-08/06-task-service/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 06. Task and Service
## ECS의 실행 단위와 유지 단위
> **학습 목표**
>
> - Task Definition, Task, Service의 관계를 설명할 수 있다.
> - Desired Count의 의미를 이해한다.
> - Service가 배포와 장애 복구에 미치는 영향을 설명할 수 있다.
> - Task Role과 Task Execution Role을 구분할 수 있다.
> - Task Definition의 주요 필드를 작성할 수 있다.
---
# 왜 Task와 Service가 필요한가
EC2에서 `docker run` 명령을 수동으로 실행하면 포트, 환경 변수, 메모리 제한과 로그 설정이 작업자의 셸 기록에만 남기 쉽다.

컨테이너 프로세스가 죽었을 때 누군가 다시 실행해야 하고 서버 장애가 발생하면 다른 서버를 찾아 같은 명령을 재현해야 한다.

ECS Task Definition은 실행 설정을 버전 있는 선언으로 만들고 Service는 그 선언대로 원하는 Task 수를 지속해서 유지한다.

이 구조 덕분에 운영자는 개별 Container를 직접 살리는 대신 원하는 상태를 선언하고 ECS가 실제 상태를 맞추도록 할 수 있다.
---
# Task Definition이란?
Task Definition은 하나 이상의 Container를 어떤 Image와 자원, 포트, 환경, 권한으로 실행할지 정의하는 Revision 기반 설계도이다.

내용을 수정하면 기존 Revision이 바뀌는 것이 아니라 새 Revision이 등록되며 Service가 사용할 Revision을 선택한다.
| 필드 | 의미 | 예 |
|---|---|---|
| `family` | Revision을 묶는 이름 | `order-api` |
| `containerDefinitions` | Image, Port, Log, Health 등 | Spring Boot Container |
| `cpu`, `memory` | Task 수준 자원 | Fargate 지원 조합 |
| `networkMode` | Task 네트워크 방식 | `awsvpc` |
| `requiresCompatibilities` | 실행 환경 | `FARGATE` |
| `executionRoleArn` | Image Pull과 로그 전송 권한 | Task Execution Role |
| `taskRoleArn` | 애플리케이션 AWS API 권한 | Task Role |
| `runtimePlatform` | OS와 CPU Architecture | Linux, X86_64 |
Task Definition은 Container Image 자체를 저장하지 않고 [ECR](/aws-backend/part-08/04-ecr/)의 Image URI를 참조한다.
---
# Task와 Service란?
Task는 Task Definition의 특정 Revision을 실제로 실행한 인스턴스이며 하나 이상의 Container를 포함할 수 있다.

Service는 지정한 Task Definition으로 `desiredCount`만큼 Task를 유지하고 비정상 Task를 대체하며 배포를 조정한다.
```text
ECS Cluster
└── Service: order-api
    ├── desiredCount: 2
    ├── Task A
    │   └── Container: Spring Boot
    └── Task B
        └── Container: Spring Boot

Task Definition
└── family: order-api
    ├── revision: 12
    └── image: order-api@sha256:...
```
배치 작업처럼 완료 후 종료되는 워크로드는 Standalone Task로 실행할 수 있지만 계속 요청을 받아야 하는 API는 일반적으로 Service로 관리한다.
---
# Desired Count와 실제 상태
Desired Count는 Service가 유지하려는 Task 수이며 현재 Running Count와 일시적으로 다를 수 있다.
| 상황 | Desired Count | ECS 동작 |
|---|---:|---|
| 정상 운영 | 2 | Running Task 2개 유지 |
| Task 1개 장애 | 2 | 대체 Task 시작 |
| Scale-out | 4 | Task 2개 추가 |
| 새 Revision 배포 | 2 | 배포 설정 범위에서 교체 |
Application Auto Scaling을 연결하면 CPU, 메모리 또는 ALB 요청 지표 등에 따라 Desired Count를 조정할 수 있다.
---
# 전체 동작 흐름
```text
Developer
   │
CI Build ──> ECR Push
                  │
                  ▼
Task Definition Revision
                  │
                  ▼
ECS Service ──> Scheduler ──> Task
     │                         ├── ENI
     │                         └── Spring Boot :8080
     │                                  ▲
     └──────── ALB Target Group ─────────┘
```
1. CI가 고유 Tag 또는 Digest로 Image를 ECR에 Push한다.
2. Image URI를 포함한 새 Task Definition Revision을 등록한다.
3. ECS Service가 새 Revision으로 업데이트된다.
4. Scheduler가 [ECS Cluster](/aws-backend/part-08/05-ecs-cluster/)의 Capacity에 Task를 배치한다.
5. `awsvpc` Task에 ENI와 사설 IP가 할당된다.
6. ALB Target Group Health Check를 통과한 Task가 트래픽을 받는다.
7. Task가 종료되면 Service가 Desired Count를 맞추기 위해 대체 Task를 시작한다.
---
# Task Definition 예시
다음 예시는 Fargate에서 Spring Boot Container 하나를 실행하는 핵심 구조이다.
```json
{
  "family": "order-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/order-api-execution",
  "taskRoleArn": "arn:aws:iam::123456789012:role/order-api-task",
  "runtimePlatform": {
    "operatingSystemFamily": "LINUX",
    "cpuArchitecture": "X86_64"
  },
  "containerDefinitions": [
    {
      "name": "order-api",
      "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:git-a1b2c3d",
      "essential": true,
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
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:order-db"
        }
      ],
      "stopTimeout": 45,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/order-api",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "app"
        }
      }
    }
  ]
}
```
CPU와 메모리 값은 예시이며 실제 부하와 Fargate에서 지원하는 조합을 기준으로 선택해야 한다.
---
# Task Role과 Task Execution Role
두 Role은 신뢰 주체가 Task와 관련되어 보이지만 사용 목적이 명확히 다르다.
| 역할 | 사용하는 주체 | 대표 권한 |
|---|---|---|
| Task Execution Role | ECS Agent 또는 Fargate 실행 환경 | ECR Pull, CloudWatch Logs 전송, Secret 주입 |
| Task Role | Container 안의 애플리케이션 | S3, SQS, DynamoDB 등 업무 API 호출 |
ECS가 시작 시 ECR에서 Image를 가져오려면 ECR Pull 권한은 Task Execution Role에 부여한다.

---
# 실습: Revision 등록과 Service 갱신
Task Definition JSON 파일을 등록하고 반환된 Revision을 확인한다.
```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json

aws ecs describe-task-definition \
  --task-definition order-api
```
Service를 새 Revision으로 업데이트하고 안정화 상태를 기다린다.
```bash
aws ecs update-service \
  --cluster production-app \
  --service order-api \
  --task-definition order-api:12

```
---
# Spring Boot에서는 어떻게 쓰는가
Actuator Probe와 graceful shutdown을 활성화한다.
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
```
ALB Health Check는 `/actuator/health/readiness`를 사용하고 Container의 `stopTimeout`은 Spring 종료 유예 시간보다 충분히 길게 둔다.

---
# 실무에서는 어떻게 사용할까
Task Definition JSON이나 Infrastructure as Code를 Git에서 관리해 실행 설정 변경도 코드 리뷰와 이력을 거치게 한다.

Image 변경 없이 CPU, 메모리, 환경 변수, Role 또는 로그 설정만 바뀌어도 새 Revision을 등록한다.

Service Event, Running Count, Pending Count, Target Health와 Container 로그를 하나의 배포 단위로 관찰한다.

---
# 장애 사례
| 증상 | 주요 원인 | 확인 지점 |
|---|---|---|
| `CannotPullContainerError` | Image URI, 네트워크, Execution Role 오류 | Service Event |
| Task 시작 전 실패 | Secret 조회 권한 부족 | Execution Role과 KMS 권한 |
| Task 반복 재시작 | Container 종료 또는 ALB Health 실패 | Stopped Reason, Target Health |
| 종료 코드 `137` | Task 메모리 부족 | Memory 지표와 JVM 설정 |
| S3 접근 거부 | Task Role 권한 또는 Resource ARN 오류 | CloudTrail, IAM Policy |
Task가 실행되기 전 발생한 ECR Pull과 로그 설정 문제는 애플리케이션 로그가 남지 않을 수 있으므로 ECS Service Event와 Stopped Reason을 먼저 확인한다.
---
# 주의할 점
- Task Definition Revision과 Image Digest를 배포 기록에 남긴다.
- `latest` Tag를 재사용해 Revision의 의미를 흐리지 않는다.
- Task Role과 Task Execution Role을 바꾸어 부여하지 않는다.
- 환경 변수와 Secret 참조를 구분한다.
- Task CPU와 메모리에는 모든 Container 자원 합계를 고려한다.
- `essential` Container 종료가 Task 전체에 미치는 영향을 확인한다.
- `awsvpc`의 Subnet, Security Group과 ENI Capacity를 검증한다.
---
# 비용과 성능 고려사항
Fargate는 Task Definition에 지정한 vCPU와 메모리 및 실행 시간 등이 비용의 핵심이므로 실제 사용량을 보고 Right-sizing한다.

EC2 Launch Type은 인스턴스 비용과 Task 배치 효율을 함께 관리하며 배포 시 새 Task를 수용할 여유를 남긴다.

Desired Count를 늘리면 처리 Capacity와 가용성이 높아질 수 있지만 Task 실행 비용과 로그 수집량도 증가한다.

ECR 저장량, CloudWatch Logs 보존, NAT Gateway를 통한 Image Pull도 Service 운영 비용에 포함한다.
---
# 기억해야 할 내용
- Task Definition은 Revision으로 관리되는 Container 실행 설계도이다.
- Task는 특정 Revision을 실행한 인스턴스이다.
- Service는 Desired Count와 배포 상태를 유지한다.
- `awsvpc` Task는 자체 ENI와 사설 IP를 가진다.
- Task Execution Role은 ECR Pull과 로그 전송에 사용된다.
- Task Role은 Spring Boot가 AWS API를 호출할 때 사용된다.
- Image, Revision, Health Check를 함께 추적해야 배포를 재현할 수 있다.
---
# 다음 Chapter
다음 Chapter에서는 [ECS Deployment](/aws-backend/part-08/07-ecs-deployment/)를 학습한다.

새 Task Definition Revision을 Rolling 또는 Blue/Green 방식으로 배포하고 ALB Health Check와 자동 롤백을 연결하는 방법을 살펴본다.
