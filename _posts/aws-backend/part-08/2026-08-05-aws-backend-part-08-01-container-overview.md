---
title: "Chapter 01. Container Overview"
permalink: /aws-backend/part-08/01-container-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 01. Container Overview
## 애플리케이션 실행 환경을 표준화하기
> **학습 목표**
>
> - 컨테이너를 사용하는 이유를 설명할 수 있다.
> - Docker, ECR, ECS의 관계를 이해한다.
> - EC2 직접 운영과 ECS 운영의 차이를 설명할 수 있다.
> - Part 8에서 다룰 이미지 빌드부터 배포까지의 흐름을 설명할 수 있다.
---
# 왜 컨테이너 플랫폼이 필요한가
개발자의 로컬에서는 정상 실행되던 Spring Boot 애플리케이션이 운영 EC2에서는 Java 버전과 환경 설정 차이로 시작되지 않는 상황을 생각해 보자.

JAR 파일을 서버마다 직접 복사하면 어느 서버에 어떤 버전이 배포되었는지 추적하기 어렵고, 장애가 발생했을 때 이전 버전으로 되돌리는 절차도 사람의 기억에 의존한다.

서버가 한두 대일 때는 SSH 접속과 셸 스크립트로 버틸 수 있지만 인스턴스와 배포 횟수가 늘어나면 실행 환경의 차이와 수작업이 장애 원인이 된다.

컨테이너 플랫폼은 애플리케이션과 실행 환경을 **Image**라는 동일한 배포 단위로 만들고, 필요한 개수의 실행 인스턴스를 자동으로 유지한다.

Part 7에서 S3에 데이터를 안전하게 저장하는 방법을 다뤘다면, Part 8에서는 상태를 외부 저장소에 둔 Spring Boot 애플리케이션을 컨테이너로 운영하는 방법을 다룬다.
---
# 컨테이너란?
컨테이너는 Image에 정의된 파일 시스템과 실행 명령을 격리된 프로세스로 실행한 단위이다.

가상 머신처럼 별도의 운영체제 전체를 실행하지 않고 호스트의 Linux 커널을 공유하기 때문에 상대적으로 시작이 빠르고 배포 단위가 작다.

Docker는 Image를 만들고 실행하는 도구이며, ECR은 Image를 저장하는 Registry이고, ECS는 여러 컨테이너의 배치와 복구와 배포를 관리하는 오케스트레이션 서비스이다.

[Part 3의 Docker Chapter](/aws-backend/part-03/08-docker/)는 EC2 한 대에서 Image와 Container를 실행하는 기초를 설명한다.

이 Part는 그 기초를 바탕으로 재현 가능한 Image 빌드, ECR 배포, ECS 운영과 무중단 교체에 집중한다.
---
# VM과 Container 비교
| 구분 | Virtual Machine | Container |
|---|---|---|
| 격리 단위 | Guest OS를 포함한 VM | 호스트 커널을 공유하는 프로세스 |
| 시작 시간 | 운영체제 부팅이 필요함 | 프로세스 시작 중심임 |
| 배포 단위 | VM Image | Container Image |
| 자원 오버헤드 | 상대적으로 큼 | 상대적으로 작음 |
| 격리 강도 | 하이퍼바이저 경계 | 커널 기능 기반 경계 |
| 운영 책임 | Guest OS 패치 포함 | Image와 Runtime 보안 포함 |
컨테이너가 VM보다 항상 우월한 것은 아니며, 격리 요구와 운영 모델에 따라 두 방식을 함께 사용할 수 있다.

ECS의 EC2 Launch Type은 EC2 위에 컨테이너를 배치하므로 VM과 컨테이너를 결합한 형태이다.
---
# Part 8 전체 구조
Spring Boot 소스 코드는 CI에서 Image로 빌드되고 ECR을 거쳐 ECS Service의 Task로 실행된다.
```text
Developer
   │ Git Push
   ▼
CI Test & Docker Build
   │
   ▼
ECR Repository
   │ Image Pull
   ▼
ECS Cluster
   └── Service
       ├── Task ── Container(Spring Boot)
       └── Task ── Container(Spring Boot)
              ▲
              │ Health Check / Traffic
         ALB Target Group
```
![ECS deployment](/assets/aws-backend/ecs-deployment.png)

흐름은 다음 순서로 진행한다.
1. 개발자가 코드를 Push하면 CI가 테스트를 실행한다.
2. CI는 Dockerfile로 Image를 한 번 빌드하고 식별 가능한 Tag를 붙인다.
3. Image를 [ECR](/aws-backend/part-08/04-ecr/) Repository에 Push한다.
4. Task Definition이 ECR Image URI와 CPU, 메모리, 포트, 권한을 선언한다.
5. ECS Service가 새 Task를 시작하고 원하는 Task 수를 유지한다.
6. ALB Target Group의 Health Check를 통과한 Task만 요청을 받는다.
---
# ECS 계층 관계
ECS의 핵심 객체는 Cluster, Task Definition, Task, Service이며 각각의 책임을 섞어 이해하면 장애 분석이 어려워진다.
| 객체 | 역할 | 비유 |
|---|---|---|
| Cluster | 실행 Capacity를 논리적으로 묶음 | 실행 공간 |
| Task Definition | 컨테이너 실행 설정의 버전 있는 선언 | 설계도 |
| Task | Task Definition을 실행한 인스턴스 | 실행물 |
| Service | 원하는 Task 수와 배포 상태를 유지 | 관리자 |
Cluster가 애플리케이션을 직접 실행하는 것이 아니라, Cluster 안의 Capacity에 Task가 배치된다.

Service 없이 일회성 Task를 실행할 수도 있지만 HTTP API처럼 계속 살아 있어야 하는 워크로드는 일반적으로 Service로 관리한다.
---
# 실습으로 전체 흐름 확인하기
로컬에서 Maven 빌드와 Docker Image 생성을 먼저 확인한다.
```bash
./mvnw clean package
docker build -t order-api:2026.08.05 .
docker run --rm -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=local \
  order-api:2026.08.05
```
로컬 실행이 성공하면 ECR 로그인과 Push를 수행한다.
```bash
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

docker tag order-api:2026.08.05 \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:2026.08.05

docker push \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:2026.08.05
```
실제 계정 ID와 Region, Repository 이름은 자신의 환경에 맞게 바꿔야 한다.
---
# Spring Boot에서는 어떻게 쓰는가
Spring Boot 3.x 애플리케이션은 Java 17 이상의 Runtime을 포함한 Image로 만들고 환경별 설정은 실행 시점에 주입한다.
```dockerfile
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /workspace
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /workspace/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```
운영 Profile과 종료 방식, Actuator 노출 범위는 설정으로 관리한다.
```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true
```
`SPRING_PROFILES_ACTIVE=prod`는 Task Definition의 환경 변수로 주입하고 비밀번호는 Secrets Manager나 Parameter Store에서 참조한다.

ALB Health Check는 `/actuator/health/readiness`처럼 요청 수신 가능성을 나타내는 경로를 사용하고, ECS의 `stopTimeout`은 애플리케이션의 graceful shutdown 시간보다 충분히 길게 맞춘다.
---
# 실무에서는 어떻게 사용할까
CI에서 검증한 Image를 한 번만 빌드하고 개발, 스테이징, 운영 환경에서 같은 Digest를 승격하면 환경별 재빌드로 생기는 차이를 줄일 수 있다.

Image는 불변 산출물로 취급하며 실행 중인 컨테이너에 접속해 파일을 수정하지 않는다.

애플리케이션 로그는 표준 출력으로 보내 CloudWatch Logs에 모으고, 업로드 파일과 세션은 각각 S3와 Redis 같은 외부 저장소에 둔다.

Chapter별 학습 순서는 다음과 같다.
| Chapter | 핵심 질문 |
|---|---|
| [02. Docker Image](/aws-backend/part-08/02-docker-image/) | 재현 가능한 배포 산출물을 어떻게 만드는가 |
| [03. Container Runtime](/aws-backend/part-08/03-container-runtime/) | Image가 어떤 격리 프로세스로 실행되는가 |
| [04. ECR](/aws-backend/part-08/04-ecr/) | Image를 어디에 안전하게 저장하는가 |
| [05. ECS Cluster](/aws-backend/part-08/05-ecs-cluster/) | Fargate와 EC2 Capacity를 어떻게 선택하는가 |
| [06. Task and Service](/aws-backend/part-08/06-task-service/) | 실행 설정과 원하는 개수를 어떻게 유지하는가 |
| [07. ECS Deployment](/aws-backend/part-08/07-ecs-deployment/) | 새 버전을 어떻게 안전하게 교체하는가 |
---
# 장애 사례
| 증상 | 주요 원인 | 확인 지점 |
|---|---|---|
| 로컬만 정상 실행 | Java 버전 또는 환경 변수 차이 | Image Runtime, Task 환경 변수 |
| Task 반복 재시작 | Health Check 경로 또는 시작 시간 오류 | Service Event, Target Health |
| 종료 코드 `137` | 메모리 한도 초과로 강제 종료 | Task 메모리, 애플리케이션 지표 |
| 어떤 버전인지 모름 | `latest` Tag 재사용 | Task Definition Image URI, Digest |
플랫폼은 실패한 Task를 다시 시작할 뿐이며, 잘못된 설정의 원인을 자동으로 고쳐 주지는 않는다.
---
# 주의할 점
- Image에 Secret을 포함하지 않는다.
- 컨테이너 Writable Layer에 영속 데이터를 저장하지 않는다.
- `latest` 대신 Git SHA나 릴리스 버전과 Digest로 배포를 식별한다.
- Task Role과 Task Execution Role의 책임을 분리한다.
- Health Check와 graceful shutdown을 배포 설정과 함께 검증한다.
Task Role은 애플리케이션이 S3나 SQS 같은 AWS API를 호출할 때 사용하는 권한이다.

Task Execution Role은 ECS가 ECR Image를 Pull하고 CloudWatch Logs로 로그를 전송하는 등 Task 시작을 준비할 때 사용하는 권한이다.
---
# 비용과 성능 고려사항
Fargate는 Task에 할당한 vCPU와 메모리 및 실행 시간 등이 주요 과금 요소이고, EC2 Launch Type은 컨테이너가 적게 실행되어도 기반 인스턴스 비용이 발생한다.

ECR에는 Image 저장량과 데이터 전송이 비용 요소가 되며 사용하지 않는 Image가 쌓이지 않도록 Lifecycle Policy를 둔다.

Private Subnet의 Task가 NAT Gateway를 통해 ECR에서 Image를 Pull하면 NAT 처리 및 데이터 전송 비용이 생길 수 있으므로 ECR API, ECR DKR, S3 VPC Endpoint 사용을 검토한다.

Image 크기가 작으면 Pull과 Task 시작이 빨라지지만 진단 도구와 인증서가 누락되지 않았는지 함께 확인해야 한다.
---
# 기억해야 할 내용
- 컨테이너는 Image를 격리된 프로세스로 실행한 단위이다.
- Docker는 Image 빌드와 실행을 담당하고 ECR은 Image를 저장한다.
- ECS는 Cluster 안에서 Task를 배치하고 Service로 원하는 수를 유지한다.
- Task Role은 애플리케이션 권한이고 Task Execution Role은 Task 시작 준비 권한이다.
- 운영 Image는 추적 가능한 Tag와 Digest로 식별한다.
- ALB Health Check와 graceful shutdown은 안전한 교체의 핵심이다.
- Part 8은 EC2 위 Docker 기초를 넘어 ECR과 ECS 운영에 집중한다.
---
# 다음 Chapter
다음 Chapter에서는 [Docker Image](/aws-backend/part-08/02-docker-image/)의 Layer, Tag, Digest와 빌드 캐시를 학습한다.

Spring Boot 애플리케이션을 작고 재현 가능한 불변 산출물로 만드는 방법을 자세히 살펴본다.

