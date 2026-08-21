---
title: "Chapter 01. Compute Overview"
permalink: /aws-backend/part-03/01-compute-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Compute Overview
## 애플리케이션이 실행되는 위치

> **학습 목표**
>
> - AWS Compute 서비스의 역할을 설명할 수 있다.
> - EC2, ECS, Lambda의 차이를 큰 흐름으로 이해한다.
> - Spring Boot 애플리케이션이 어떤 실행 환경 위에서 동작하는지 설명할 수 있다.
> - 워크로드에 맞는 실행 방식을 선택할 수 있다.

---

# 왜 Compute가 필요한가

쇼핑몰의 Spring Boot 코드는 Git 저장소에만 있어서는 사용자 요청을 처리할 수 없다.

코드를 실행할 CPU와 메모리, 요청을 받을 네트워크, 프로그램을 실행할 운영체제가 필요하다.

개발자 노트북은 전원이 꺼지거나 네트워크가 바뀌면 서비스가 중단되므로 운영 환경으로 적합하지 않다.

운영 환경에는 지속적으로 실행되고, 트래픽에 따라 확장되며, 장애가 발생하면 교체할 수 있는 실행 자원이 필요하다.

AWS에서 이러한 실행 자원을 제공하는 서비스 범주가 **Compute**이다.

---

# Compute란?

Compute는 애플리케이션 코드를 실행하는 자원이다.

백엔드 관점에서는 Spring Boot 프로세스가 실제로 떠 있는 실행 환경이라고 볼 수 있다.

대표 서비스는 다음과 같다.

| 서비스 | 특징 |
|---|---|
| EC2 | 가상 서버를 직접 관리 |
| ECS | 컨테이너를 클러스터에서 실행 |
| Lambda | 함수 단위로 실행 |
| App Runner | 컨테이너 기반 관리형 실행 환경 |

Part 3에서는 가장 기본이 되는 EC2를 중심으로 학습한다.

---

# 실행 환경의 계층

Spring Boot 애플리케이션은 여러 계층 위에서 하나의 프로세스로 실행된다.

```
AWS 물리 서버
└── 가상화 계층
    └── EC2 인스턴스
        └── Linux
            ├── JVM
            │   └── Spring Boot 프로세스
            └── Docker Engine
                └── Spring Boot 컨테이너
```

![Compute runtime](/assets/aws-backend/compute-runtime.png)

EC2를 사용하면 Linux부터 위쪽 계층을 사용자가 관리한다.

ECS나 Lambda처럼 관리 수준이 높은 서비스를 사용하면 AWS가 담당하는 범위가 넓어져 운영 부담이 줄어든다.

관리 범위가 줄어드는 대신 실행 방식과 배포 방법에 서비스별 제약이 생길 수 있다.

---

# EC2를 먼저 배우는 이유

ECS, EKS, Lambda를 쓰더라도 Compute의 기본 개념은 EC2에서 시작한다.

서버가 어떻게 생성되고, 디스크가 어떻게 붙고, 네트워크 인터페이스가 어떻게 연결되는지 이해해야 이후 서비스도 쉽게 이해할 수 있다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# EC2를 구성하는 자원

EC2 한 대는 독립된 제품이 아니라 여러 AWS 자원이 결합된 결과이다.

```
EC2 Instance
├── AMI: 시작할 OS와 파일 시스템
├── Instance Type: CPU와 메모리
├── EBS: 영속 블록 스토리지
├── ENI: VPC에 연결되는 네트워크 카드
├── Security Group: 송수신 접근 규칙
└── IAM Role: AWS API를 호출할 권한
```

Part 2에서 배운 VPC와 Subnet은 서버가 놓일 네트워크를 결정한다.

Part 3에서는 그 안에서 애플리케이션이 실행되는 과정을 다룬다.

---

# 실행 방식 비교

실행 방식을 고를 때는 익숙함보다 **운영 책임과 워크로드 특성**을 먼저 비교해야 한다.

| 기준 | EC2 | ECS | Lambda |
|---|---|---|---|
| 실행 단위 | 가상 머신 | 컨테이너 | 함수 |
| OS 관리 | 사용자가 담당 | 실행 방식에 따라 감소 | AWS가 담당 |
| 실행 시간 | 상시 서버에 적합 | 상시·배치 모두 가능 | 이벤트 기반 작업에 적합 |
| 확장 단위 | 인스턴스 | Task | 동시 실행 |
| 제어 범위 | 넓음 | 중간 | 제한적 |

Spring Boot REST API는 EC2와 ECS 등 여러 방식으로 실행할 수 있으므로 하나의 정답은 없다.

학습 단계에서는 EC2에 직접 배포해 프로세스와 네트워크를 이해하고, 규모가 커지면 관리형 플랫폼을 검토하는 흐름이 자연스럽다.

---

# AWS 콘솔과 CLI에서는

EC2 인스턴스 시작 화면에서는 AMI, 인스턴스 유형, Key Pair, 네트워크, 스토리지를 순서대로 선택한다.

CLI에서는 현재 리전의 실행 중인 인스턴스를 조회할 수 있다.

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,Ip:PrivateIpAddress}"
```

리소스를 다루기 전에는 AWS CLI가 바라보는 계정과 리전을 확인해야 한다.

```bash
aws sts get-caller-identity
aws configure get region
```

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot는 실행 위치와 무관하게 JVM 프로세스로 동작한다.

포트와 Profile 같은 환경별 값은 코드에 고정하지 않고 외부 설정으로 전달한다.

```yaml
server:
  port: ${SERVER_PORT:8080}

spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
```

동일한 JAR를 여러 환경에서 사용하고 설정만 바꾸면 빌드 결과와 운영 환경의 결합을 줄일 수 있다.

AWS API 호출 권한은 고정 Access Key 대신 EC2 IAM Role로 제공해야 한다.

---

# 실무에서는 어떻게 사용할까

초기 서비스는 EC2 한 대에서 시작할 수 있지만 운영 서비스는 인스턴스 장애를 전제로 설계한다.

```
Internet
   │
  ALB
   │
├── EC2-A ── Spring Boot
└── EC2-B ── Spring Boot
        │
       RDS
```

애플리케이션을 무상태로 만들면 문제가 생긴 인스턴스를 고치기보다 새 인스턴스로 교체하기 쉽다.

업로드 파일과 세션을 인스턴스 로컬 디스크에만 저장하면 교체와 수평 확장이 어려워진다.

파일은 S3, 세션은 Redis, 관계형 데이터는 RDS처럼 목적에 맞는 외부 저장소에 두는 것이 일반적이다.

Part 4에서는 ALB와 Auto Scaling이 여러 실행 자원으로 요청을 분산하고 교체하는 방식을 학습한다.

---

# 장애 사례

배포 후 API가 열리지 않는다고 해서 항상 애플리케이션 문제인 것은 아니다.

| 증상 | 확인할 계층 | 대표 원인 |
|---|---|---|
| 인스턴스 접속 불가 | VPC·ENI | 경로 또는 Security Group 누락 |
| 프로세스 시작 실패 | Linux·JVM | 환경 변수, 권한, 메모리 문제 |
| 재부팅 후 서비스 중단 | Systemd | 자동 시작 미설정 |
| 디스크 쓰기 실패 | EBS | 용량 부족 또는 마운트 오류 |
| 컨테이너 반복 종료 | Docker | 실행 명령 또는 설정 오류 |

장애 분석은 네트워크, 인스턴스, 운영체제, 런타임, 애플리케이션 순서로 계층을 좁혀 가는 것이 효율적이다.

---

# 비용과 성능 고려사항

Compute 비용은 주로 실행 시간, CPU와 메모리 크기, 운영체제 라이선스, 데이터 전송에서 발생한다.

큰 인스턴스 한 대를 쓰는 수직 확장은 단순하지만 장애 영향 범위가 커질 수 있다.

여러 인스턴스를 쓰는 수평 확장은 가용성을 높일 수 있지만 로드밸런서와 배포 자동화가 필요하다.

CPU뿐 아니라 메모리, 응답 시간, JVM GC 지표를 함께 관찰하여 용량을 조정해야 한다.

---

# 기억해야 할 내용

- Compute는 애플리케이션을 실행하는 자원이다.
- EC2는 AWS의 기본 가상 서버 서비스다.
- AMI, EBS, ENI는 EC2를 이해하는 핵심 구성 요소다.
- Docker와 Systemd는 애플리케이션 실행 방식과 관련된다.
- 실행 방식은 운영 책임과 워크로드 특성을 기준으로 선택한다.
- 무상태 애플리케이션은 교체와 수평 확장이 쉽다.
- 장애는 실행 환경의 계층별로 분석한다.

---

# 다음 Chapter

다음 Chapter에서는 [EC2](/aws-backend/part-03/02-ec2/)를 학습한다.

가상 서버가 생성될 때 어떤 자원이 결합되며 인스턴스의 상태와 유형을 어떻게 선택하는지 알아본다.


