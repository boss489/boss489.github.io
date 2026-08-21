---
title: "Chapter 05. ECS Cluster"
permalink: /aws-backend/part-08/05-ecs-cluster/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 05. ECS Cluster
## 컨테이너를 실행할 논리적 묶음
> **학습 목표**
>
> - ECS Cluster의 의미를 설명할 수 있다.
> - Fargate와 EC2 Launch Type의 차이를 이해한다.
> - Cluster가 직접 애플리케이션을 실행하는 것은 아님을 설명할 수 있다.
> - Capacity Provider와 Task 배치의 관계를 이해한다.
> - `awsvpc` 네트워크에서 Task ENI의 역할을 설명할 수 있다.
---
# 왜 ECS Cluster가 필요한가
EC2에 SSH로 접속해 컨테이너를 직접 실행하다 보면 특정 서버에 컨테이너가 몰리고 장애 난 서버의 애플리케이션을 다른 서버에 수동으로 다시 띄워야 한다.

배포할 때도 어느 서버에 여유 CPU와 메모리가 있는지 사람이 판단해야 하며 서버를 추가해도 컨테이너가 자동으로 이동하지 않는다.

ECS Cluster는 컨테이너 워크로드와 실행 Capacity를 논리적으로 묶고 Scheduler가 Task를 배치할 범위를 제공한다.

애플리케이션 운영자는 개별 호스트보다 Service와 Task의 원하는 상태에 집중하고 ECS가 사용 가능한 Capacity에 실행 단위를 배치한다.
---
# ECS Cluster란?
ECS Cluster는 Service, Task와 Capacity Provider를 조직하는 Region 단위의 논리적 리소스이다.

Cluster 자체가 CPU나 메모리를 제공하거나 Container를 직접 실행하는 것은 아니다.

Fargate를 사용하면 AWS가 관리하는 서버리스 Capacity에 Task가 실행되고, EC2를 사용하면 Cluster에 연결된 Container Instance의 Capacity에 Task가 실행된다.
```text
ECS Cluster
├── Capacity Provider
│   ├── FARGATE
│   └── EC2 Auto Scaling Group
├── Service: order-api
│   ├── Task
│   └── Task
└── Standalone Task
```
Task Definition은 Cluster 밖에서 재사용 가능한 실행 설계도이고 Task와 Service는 특정 Cluster에서 실행된다.
---
# ECS의 계층 구조
| 계층 | 책임 | 주요 변경 시점 |
|---|---|---|
| Cluster | 실행 범위와 Capacity 연결 | 환경 또는 운영 경계 구성 |
| Task Definition | Image와 자원, 포트, 역할 선언 | 실행 설정 변경 |
| Task | 선언을 실제로 실행 | 배포, 복구, 일회 작업 |
| Service | 원하는 Task 수와 배포 유지 | Scale-out, 새 Revision 배포 |
Cluster 이름을 애플리케이션 이름과 동일시하면 여러 Service를 묶는 운영 경계를 이해하기 어렵다.

개발, 스테이징, 운영을 Cluster로 분리할 수 있지만 계정과 VPC 분리 요구까지 포함해 경계를 결정해야 한다.
---
# Fargate와 EC2 Launch Type
Fargate는 기반 서버 운영을 AWS에 맡기고 Task 단위로 vCPU와 메모리를 지정하는 방식이다.

EC2 Launch Type은 사용자가 AMI, 인스턴스 유형, Auto Scaling Group, ECS Agent와 Capacity를 관리하는 방식이다.
| 기준 | Fargate | EC2 Launch Type |
|---|---|---|
| 서버 관리 | AWS가 기반 호스트 관리 | 사용자가 EC2 관리 |
| Capacity 단위 | Task의 vCPU와 메모리 | EC2 인스턴스 |
| 확장 관점 | Task 수 중심 | Task와 인스턴스 수 모두 고려 |
| 운영 복잡도 | 상대적으로 낮음 | 상대적으로 높음 |
| 자원 선택 | 지원되는 Task 조합 | 인스턴스 유형 범위에서 구성 |
| 적합한 경우 | 빠른 시작, 가변 워크로드 | 높은 사용률, 특수 인스턴스 요구 |
처음 시작하는 서비스에는 Fargate가 단순할 수 있지만 트래픽 패턴과 운영 역량에 따라 EC2가 더 적합할 수도 있다.

어느 방식이 항상 저렴하다고 단정할 수 없으며 실제 Task 크기, 사용률, 유휴 Capacity와 할인 약정을 함께 비교해야 한다.
---
# Capacity Provider
Capacity Provider는 ECS가 Task를 실행할 Capacity와 확장 방식을 표현한다.

Fargate에서는 `FARGATE`와 `FARGATE_SPOT`을 사용할 수 있고 EC2에서는 Auto Scaling Group 기반 Capacity Provider를 연결할 수 있다.

Capacity Provider Strategy는 `base`와 `weight`로 최소 배치 수와 이후 Task의 분배 비율을 정한다.

Spot Capacity는 중단될 수 있으므로 재시작 가능한 Stateless 워크로드와 충분한 On-Demand 기반을 함께 설계해야 한다.
---
# Task 배치와 네트워크 흐름
```text
Developer
   │
CI Build ──> ECR Push
                   │
                   ▼
ECS Service ──> Scheduler
                   │ Capacity 선택
          ┌────────┴────────┐
          ▼                 ▼
      Fargate Task      EC2 Container Instance
          │                 └── Task
          └──── ENI / IP ───────┘
                    ▲
                    │
              ALB Target Group
```
1. Service가 원하는 Task 수를 Scheduler에 전달한다.
2. Scheduler는 Capacity Provider Strategy와 가용 자원을 확인한다.
3. Subnet, Security Group과 제약 조건을 만족하는 위치에 Task를 배치한다.
4. `awsvpc` 모드에서는 각 Task에 ENI와 사설 IP를 할당한다.
5. ALB Target Group에 Task IP가 등록되고 Health Check가 시작된다.
6. 정상 Target만 사용자 요청을 받는다.
여러 Availability Zone의 Private Subnet을 지정하면 Task를 분산해 단일 AZ 장애 영향을 줄일 수 있다.
---
# 실습: Cluster 생성과 확인
AWS CLI로 Cluster를 생성하고 Container Insights 설정을 활성화하는 예시이다.
```bash
aws ecs create-cluster \
  --cluster-name production-app \
  --settings name=containerInsights,value=enabled \
  --region ap-northeast-2

aws ecs describe-clusters \
  --clusters production-app \
  --include SETTINGS STATISTICS

aws ecs list-services \
  --cluster production-app

aws ecs list-tasks \
  --cluster production-app
```
Container Insights는 관찰 가능성을 높이지만 추가 Metrics와 Logs 수집 비용이 발생할 수 있으므로 필요한 환경과 보존 범위를 정한다.

Cluster를 만들었다고 Task가 자동으로 실행되는 것은 아니며 Task Definition과 Service 또는 `run-task` 호출이 추가로 필요하다.
---
# Spring Boot에서는 어떻게 쓰는가
Spring Boot 애플리케이션은 Stateless Container Image로 만들고 Cluster의 Service가 여러 Task로 실행하도록 구성한다.
```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/order-api.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```
환경별 Profile은 Image가 아니라 Task에 주입한다.
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
`SPRING_PROFILES_ACTIVE=prod`와 데이터베이스 Endpoint는 환경 변수로 전달하고 비밀번호는 Secret 참조를 사용한다.

ALB는 `/actuator/health/readiness`를 확인하고 ECS의 `stopTimeout`은 graceful shutdown보다 충분히 길게 설정한다.
---
# 네트워크와 보안 설계
Fargate Task는 일반적으로 Private Subnet에 두고 Internet-facing ALB는 Public Subnet에 둔다.

Task Security Group의 `8080` Inbound Source를 ALB Security Group으로 제한하면 인터넷에서 Task IP로 직접 접근하는 경로를 막을 수 있다.

ECR Image Pull과 외부 API 호출에는 NAT Gateway 또는 적절한 VPC Endpoint와 네트워크 경로가 필요하다.

Task Role은 Spring Boot가 AWS 서비스에 접근하는 최소 권한만 가지며 Task Execution Role은 ECR Pull과 로그 전송 같은 시작 준비 권한을 가진다.
---
# 실무에서는 어떻게 사용할까
운영 Cluster는 최소 두 개 Availability Zone의 Subnet을 사용하고 Service Task가 AZ에 분산되는지 확인한다.

Fargate와 EC2 Capacity를 혼합할 때는 장애 시 어떤 Capacity로 대체되는지와 배포 중 추가 Task를 수용할 여유가 있는지 검증한다.

Cluster는 팀, 환경, 보안 경계에 맞게 나누되 지나친 분할로 모니터링과 정책 관리가 복잡해지지 않게 한다.

EC2 Capacity Provider를 사용하면 인스턴스 Drain, ECS 최적화 AMI 업데이트와 Auto Scaling 정책도 운영 절차에 포함한다.
---
# 장애 사례
| 증상 | 주요 원인 | 확인 지점 |
|---|---|---|
| Task가 `PENDING`에 머묾 | CPU, 메모리, ENI 또는 IP 부족 | Service Event, Capacity |
| Fargate Task 시작 실패 | 지원되지 않는 CPU/Memory 조합 | Task Definition |
| ALB 연결 실패 | Subnet, Route, Security Group 오류 | ENI와 Target Health |
| EC2에 Task 배치 실패 | Agent 미연결 또는 인스턴스 자원 부족 | Container Instance 상태 |
| 일부 AZ만 과부하 | Subnet 또는 Capacity 불균형 | Task의 AZ 분포 |
배포 중에는 기존 Task와 새 Task가 동시에 실행될 수 있으므로 평상시 Desired Count만 수용하는 Capacity로는 새 배포가 멈출 수 있다.
---
# 주의할 점
- Cluster를 실제 컴퓨팅 자원과 동일하게 이해하지 않는다.
- Fargate와 EC2의 책임 경계를 명확히 구분한다.
- 여러 AZ의 Subnet과 충분한 사설 IP를 확보한다.
- `awsvpc` Task의 ENI와 Security Group 한도를 확인한다.
- 배포 시 최대 동시 Task를 수용할 Capacity를 확보한다.
- Task Role과 Task Execution Role을 분리한다.
---
# 비용과 성능 고려사항
Fargate는 Task에 할당한 vCPU와 메모리 및 실행 시간 등이 주요 비용 요소이며 작은 Task가 지나치게 많으면 운영 오버헤드도 커진다.

EC2 Launch Type은 Task가 실행되지 않는 유휴 시간에도 인스턴스 비용이 발생하지만 높은 사용률과 적절한 인스턴스 선택으로 효율을 높일 수 있다.

Fargate Spot과 EC2 Spot은 비용을 줄일 수 있지만 중단 가능성을 전제로 Service 복구와 종료 신호 처리를 설계해야 한다.

Private Subnet의 NAT Gateway, ECR Image 저장과 Pull, CloudWatch Logs와 Container Insights도 전체 비용에 포함한다.
---
# 기억해야 할 내용
- ECS Cluster는 Task와 Service를 조직하는 논리적 실행 범위이다.
- 실제 Capacity는 Fargate 또는 ECS Container Instance가 제공한다.
- Fargate는 서버 관리를 줄이고 EC2는 사용자가 Capacity를 관리한다.
- Capacity Provider는 Task가 사용할 Capacity와 분배 전략을 표현한다.
- `awsvpc` 모드의 Task는 ENI와 사설 IP를 가진다.
- Cluster, Task Definition, Task, Service의 책임은 서로 다르다.
- 배포와 장애 복구를 위해 여러 AZ와 여유 Capacity가 필요하다.
---
# 다음 Chapter
다음 Chapter에서는 [Task and Service](/aws-backend/part-08/06-task-service/)를 학습한다.

Cluster 위에서 Container 실행 설정을 Revision으로 관리하고 Service가 원하는 Task 수와 배포 상태를 유지하는 원리를 살펴본다.
---
# 기억해야 할 내용
Cluster는 실행 공간이고, 실제 애플리케이션은 Task로 실행된다.


