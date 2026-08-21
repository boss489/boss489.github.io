---
title: "Chapter 03. Container Runtime"
permalink: /aws-backend/part-08/03-container-runtime/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 03. Container Runtime
## 이미지를 실제 프로세스로 실행하기
> **학습 목표**
>
> - Container와 Image의 차이를 설명할 수 있다.
> - 컨테이너 포트와 환경 변수의 의미를 이해한다.
> - 컨테이너 로그 처리 기준을 설명할 수 있다.
> - CPU와 메모리 제한이 Runtime에 미치는 영향을 설명할 수 있다.
> - 종료 신호와 graceful shutdown을 연결할 수 있다.
---
# 왜 Container Runtime을 이해해야 하는가
Image 빌드와 ECR Push까지 성공했는데 ECS Task가 시작 직후 종료되거나 ALB Health Check에 계속 실패하는 상황을 생각해 보자.

원인은 Image 자체가 아니라 잘못된 실행 명령, 누락된 환경 변수, 포트 불일치, 메모리 제한 또는 종료 신호 처리에 있을 수 있다.

오케스트레이터는 컨테이너를 다시 시작할 수 있지만 Runtime 설정이 틀렸다면 같은 실패를 반복할 뿐이다.

따라서 Image를 어떻게 만들었는지뿐 아니라 Image가 실제 Linux 프로세스로 실행될 때 어떤 자원과 네트워크와 설정을 받는지 이해해야 한다.
---
# Container란?
Container는 Image의 읽기 전용 Layer 위에 쓰기 가능한 Layer를 추가하고 격리된 프로세스로 실행한 인스턴스이다.

같은 Image로 여러 Container를 만들 수 있으며 각 Container는 서로 다른 환경 변수, 네트워크 주소, 자원 제한을 가질 수 있다.

Container Runtime은 Image를 가져오고 파일 시스템을 준비하며 Namespace와 Cgroup을 적용한 뒤 지정된 프로세스를 시작한다.
```text
Host Linux Kernel
├── Container Runtime
│   ├── Container A
│   │   ├── PID Namespace
│   │   ├── Network Namespace
│   │   └── Cgroup: CPU / Memory
│   └── Container B
│       ├── PID Namespace
│       ├── Network Namespace
│       └── Cgroup: CPU / Memory
└── Shared Kernel
```
---
# Image에서 Task까지의 동작 흐름
```text
Developer
   │
CI Docker Build
   │
ECR Push
   │
ECS Service
   │ Task Placement
   ▼
Task
├── ENI (awsvpc)
├── CPU / Memory
└── Container Runtime
    └── Java Process :8080
             ▲
             │
      ALB Target Group
```
1. ECS가 Task Definition에 지정된 Image를 ECR에서 Pull한다.
2. Runtime이 Image Layer와 쓰기 가능한 Container Layer를 준비한다.
3. `awsvpc` 모드에서는 Task에 ENI와 사설 IP가 할당된다.
4. 환경 변수와 Secret, CPU와 메모리, 로그 설정이 적용된다.
5. `ENTRYPOINT`와 `command`를 조합해 Java 프로세스를 시작한다.
6. ALB Target Group이 Task IP의 Container Port로 Health Check를 보낸다.
---
# 포트와 네트워크
Spring Boot가 컨테이너 내부에서 `8080` 포트로 실행되면 Task Definition의 Container Port와 Target Group Port도 일관되어야 한다.

| 환경 | 연결 방식 |
|---|---|
| 로컬 Docker | `-p 8080:8080`으로 Host Port를 게시 |
| ECS `awsvpc` | Task ENI의 IP와 Container Port로 접근 |
| ALB 연동 | Target Group에 Task IP가 Target으로 등록 |
`awsvpc` 네트워크 모드에서는 각 Task가 ENI를 가지므로 Security Group을 Task 수준에 연결할 수 있다.

ALB Security Group에서 Task Security Group의 애플리케이션 포트로만 접근을 허용하는 방식이 일반적이다.
---
# 환경 변수와 Secret
DB 주소, Spring Profile, 외부 API 주소처럼 환경별로 달라지는 값은 Image에 고정하지 않고 Task 실행 시 주입한다.
```bash
docker run --rm \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_HOST=db.internal.example \
  order-api:2026.08.05
```
일반 환경 변수는 Task Definition을 조회할 수 있는 사용자에게 보일 수 있으므로 비밀번호와 API Key에는 적합하지 않다.

Secret은 Secrets Manager나 Systems Manager Parameter Store에 저장하고 Task Definition의 `secrets`에서 ARN으로 참조한다.

Task Execution Role에는 Secret 값을 Task 시작 시 가져오는 데 필요한 권한이 필요하고, 애플리케이션이 실행 중 AWS API를 호출할 권한은 Task Role에 부여한다.
---
# 로그와 표준 출력
컨테이너의 로컬 파일에만 로그를 쓰면 Task가 교체될 때 로그도 함께 사라져 장애 분석이 어렵다.

Spring Boot 로그는 표준 출력과 표준 오류로 보내고 ECS의 `awslogs` Log Driver로 CloudWatch Logs에 전달하는 방식이 일반적이다.
```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/order-api",
      "awslogs-region": "ap-northeast-2",
      "awslogs-stream-prefix": "app"
    }
  }
}
```
Log Group 생성과 로그 전송에는 Task Execution Role 권한이 사용되며 애플리케이션의 Task Role과 혼동하면 안 된다.

---
# CPU와 메모리
CPU 제한은 프로세스가 사용할 수 있는 계산 자원을 조절하고 메모리 제한은 Task가 확보할 수 있는 메모리 경계를 정한다.
| 자원 | 제한이 너무 낮을 때 | 제한이 없거나 너무 높을 때 |
|---|---|---|
| CPU | 응답 지연, Health Check Timeout | 다른 워크로드와 Capacity 비효율 |
| Memory | OOM Kill, 종료 코드 `137` | 과다 예약과 비용 증가 |
JVM Heap만 보고 메모리를 정하면 Metaspace, Thread Stack, Direct Buffer와 Native Memory를 놓칠 수 있다.

Container Memory보다 `-Xmx`를 낮게 두고 실제 부하에서 전체 RSS와 GC를 관찰해 여유 공간을 정해야 한다.
---
# 실습: Runtime 상태 확인
로컬에서 실행 중인 프로세스, 로그, 자원 사용량과 종료 코드를 확인한다.
```bash
docker run -d \
  --name order-api \
  --memory 768m \
  --cpus 1 \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=local \
  order-api:2026.08.05

docker logs -f order-api
docker stats order-api
docker inspect order-api
docker stop --time 40 order-api
```
`docker stop`은 먼저 종료 신호를 보내고 지정 시간 안에 끝나지 않으면 강제 종료하므로 ECS의 Task 종료 과정과 유사한 부분이 있다.
---
# Spring Boot에서는 어떻게 쓰는가
Spring Boot는 종료 신호를 받았을 때 새 요청 수락을 중단하고 진행 중인 요청을 마무리하도록 graceful shutdown을 활성화할 수 있다.
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
ECS Container Definition의 `stopTimeout`은 애플리케이션의 종료 유예 시간과 로그 Flush 시간을 포함하도록 설정한다.
```json
{
  "name": "order-api",
  "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:2026.08.05",
  "essential": true,
  "portMappings": [
    {
      "containerPort": 8080,
      "protocol": "tcp"
    }
  ],
  "stopTimeout": 45
}
```
Actuator의 `/actuator/health/readiness`는 트래픽 수신 가능 여부를, `/actuator/health/liveness`는 프로세스 복구 필요성을 표현하도록 설계한다.

Health Endpoint를 외부에 과도하게 노출하지 말고 ALB와 운영 네트워크에서 필요한 경로만 접근하게 한다.
---
# 실무에서는 어떻게 사용할까
HTTP API는 상태를 RDS, Redis, S3 같은 외부 서비스에 두어 어느 Task가 요청을 받아도 같은 결과를 내도록 설계한다.

컨테이너 내부 파일 시스템은 임시 공간으로 취급하고 크기 제한과 정리 정책을 둔다.

Task 종료 시 애플리케이션이 신호를 받아 정상 종료되는지, ALB Target 해제와 종료 순서에서 진행 중인 요청이 끊기지 않는지 부하 테스트한다.

CPU와 메모리는 추측으로 고정하지 않고 실제 트래픽의 지표와 GC 로그를 기준으로 조정한다.
---
# 장애 사례
| 증상 | 원인 후보 | 확인 방법 |
|---|---|---|
| Task가 즉시 종료 | 잘못된 Command 또는 필수 환경 변수 누락 | Stopped Reason, Container 로그 |
| ALB에서 Unhealthy | 포트 또는 Health 경로 불일치 | Target Health Reason |
| 종료 코드 `137` | 메모리 초과 또는 강제 종료 | ECS Event, Memory 지표 |
| 로그가 없음 | Log Driver 또는 Execution Role 오류 | Task 시작 Event, IAM |
| 종료 중 요청 끊김 | 짧은 `stopTimeout` 또는 비활성 graceful shutdown | 배포 시 요청 실패율 |
---
# 주의할 점
- PID 1 프로세스가 종료 신호를 Java 프로세스에 전달하는지 확인한다.
- Secret을 일반 환경 변수나 Image Layer에 저장하지 않는다.
- 애플리케이션 포트, Container Port, Target Group Port를 일치시킨다.
- 로그는 Container 파일이 아니라 중앙 저장소로 전송한다.
- JVM Heap 외 Native Memory를 포함해 Task 메모리를 산정한다.
- `awsvpc` Task의 ENI와 Security Group 한도를 고려한다.
---
# 비용과 성능 고려사항
Fargate는 Task에 지정한 vCPU와 메모리 조합 및 실행 시간 등이 과금의 중심이므로 과다 할당과 반복 재시작을 줄여야 한다.

EC2 Launch Type에서는 기반 인스턴스 비용이 발생하며 Task 자원 예약을 조정해 인스턴스 사용률과 장애 여유를 균형 있게 관리한다.

CloudWatch Logs의 수집량과 보존 기간도 비용 요소이므로 로그 수준과 보존 정책을 환경별로 설정한다.

Task마다 ENI를 사용하는 `awsvpc` 모드는 네트워크 격리를 제공하지만 IP 주소와 ENI Capacity가 Scale-out 한계가 될 수 있다.
---
# 기억해야 할 내용
- Container는 Image를 실행한 격리 프로세스이다.
- 같은 Image도 환경 변수와 자원 제한에 따라 다르게 동작한다.
- `awsvpc` 모드에서는 각 Task가 ENI와 사설 IP를 가진다.
- 표준 출력 로그는 `awslogs`를 통해 CloudWatch Logs로 보낸다.
- 종료 코드 `137`은 메모리 초과나 강제 종료 가능성을 확인해야 한다.
- graceful shutdown 시간과 ECS `stopTimeout`을 함께 설계한다.
- Task Role과 Task Execution Role은 서로 다른 책임을 가진다.
---
# 다음 Chapter
다음 Chapter에서는 [ECR](/aws-backend/part-08/04-ecr/)을 학습한다.

빌드한 Image를 IAM으로 보호되는 Registry에 Push하고 ECS가 안전하게 Pull하도록 구성하는 방법을 살펴본다.
---
# 포트
Spring Boot가 컨테이너 내부에서 8080 포트로 실행되면, ECS Task Definition과 Target Group도 이 포트를 알아야 한다.
---
# 환경 변수
DB 주소, Profile, 외부 API 주소 같은 값은 Image 안에 고정하지 않고 환경 변수로 주입한다.

Secret은 별도 Secret Manager나 Parameter Store를 사용하는 것이 좋다.
---
# 기억해야 할 내용
Container는 실행 중인 애플리케이션 프로세스다.

포트, 환경 변수, 로그 설정이 배포 성공을 좌우한다.


