---
title: "Chapter 02. EC2"
permalink: /aws-backend/part-03/02-ec2/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. EC2
## AWS에서 사용하는 가상 서버

> **학습 목표**
>
> - EC2가 무엇인지 설명할 수 있다.
> - 인스턴스 타입과 상태 변화를 이해한다.
> - EC2가 VPC, AMI, EBS, ENI와 연결되는 방식을 설명할 수 있다.
> - 운영 환경에 맞는 구매 방식과 배치 전략을 선택할 수 있다.

---

# 왜 EC2가 필요한가

쇼핑몰 API를 운영하려면 Spring Boot 프로세스를 하루 종일 실행할 서버가 필요하다.

물리 서버를 구매하면 주문부터 설치까지 오래 걸리고, 트래픽이 줄어도 이미 산 장비 비용은 줄지 않는다.

EC2는 필요한 사양의 가상 서버를 몇 분 안에 만들고, 필요가 없어지면 종료할 수 있게 한다.

개발자는 서버의 CPU와 메모리, 운영체제, 네트워크와 디스크를 선택하면서도 물리 장비는 관리하지 않는다.

---

# EC2란?

EC2(Elastic Compute Cloud)는 AWS에서 제공하는 가상 서버 서비스다.

사용자는 CPU, 메모리, 네트워크 성능에 맞는 인스턴스 타입을 선택해 서버를 생성한다.

EC2는 다음 요소와 함께 만들어진다.

- AMI
- Instance Type
- EBS Volume
- ENI
- Security Group
- Key Pair

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# EC2가 만들어지는 구조

인스턴스 시작 요청은 여러 자원을 조합해 하나의 실행 환경을 만든다.

```
VPC
└── Subnet(AZ)
    └── ENI ── Security Group
        └── EC2 Instance
            ├── Instance Type
            └── EBS Root Volume
                  ▲
                 AMI
```

AMI로부터 루트 볼륨이 생성되고, 선택한 Subnet에 ENI가 만들어져 Private IP를 받는다.

인스턴스 타입이 CPU와 메모리 용량을 정하며 Security Group이 ENI의 통신을 제어한다.

EC2 인스턴스와 EBS, ENI는 서로 생명주기가 다르므로 종료 시 어떤 자원이 남는지 확인해야 한다.

---

# 인스턴스 타입

인스턴스 타입은 EC2의 CPU, 메모리, 네트워크 성능을 결정한다.

예시는 다음과 같다.

| 계열 | 용도 |
|---|---|
| t, m | 일반 목적 |
| c | CPU 중심 |
| r | 메모리 중심 |
| i | 스토리지 성능 중심 |
| g | GPU 중심 |

Spring Boot 일반 API 서버는 처음에는 `t` 또는 `m` 계열로 시작하는 경우가 많다.

버스터블 계열은 순간적인 부하에는 효율적이지만 CPU 크레딧 동작을 이해하지 않으면 지속 부하에서 성능이 낮아질 수 있다.

인스턴스 이름의 세대와 크기를 함께 읽고, 실제 CloudWatch 지표로 사양을 조정해야 한다.

---

# 상태 변화

EC2는 상태를 가진다.

```
pending -> running -> stopping -> stopped -> terminated
```

`stopped` 상태에서는 EBS 루트 볼륨이 남아 있을 수 있다.

`terminated` 상태는 인스턴스를 삭제한 상태다.

---

# Stop, Reboot, Terminate 비교

| 작업 | OS 재시작 | 호스트 이동 가능성 | 인스턴스 스토어 | EBS | 과금 관점 |
|---|---|---|---|---|---|
| Reboot | O | 일반적으로 동일 호스트 | 유지 | 유지 | 인스턴스 실행 계속 |
| Stop/Start | O | 다른 호스트에서 시작 가능 | 손실 | 유지 | 정지 중 EBS 등은 과금 |
| Terminate | 해당 없음 | 인스턴스 삭제 | 손실 | 설정에 따라 삭제 | 남은 자원은 별도 과금 |

Public IPv4 주소는 Stop/Start 후 바뀔 수 있으므로 고정 주소가 필요하면 Elastic IP나 DNS, ALB를 사용한다.

운영 데이터를 인스턴스 스토어나 임시 파일에만 두면 Stop 또는 장애 때 손실될 수 있다.

---

# AWS 콘솔과 CLI에서는

인스턴스를 만들 때는 AMI, 타입, Key Pair, VPC와 Subnet, Security Group, EBS, IAM Role을 확인한다.

CLI로 인스턴스 상태를 조회하는 예시는 다음과 같다.

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,Az:Placement.AvailabilityZone}"
```

상태 변경 명령은 서비스 중단을 일으키므로 대상 ID와 계정을 먼저 확인해야 한다.

```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0
```

---

# User Data와 초기화

User Data는 첫 부팅 시 패키지 설치나 설정 파일 생성을 자동화하는 데 사용할 수 있다.

```bash
#!/bin/bash
dnf update -y
dnf install -y java-17-amazon-corretto-headless
```

긴 초기화 작업은 인스턴스가 `running`이어도 애플리케이션 준비를 늦출 수 있다.

반복 배포에서는 User Data만 늘리기보다 AMI, IaC, 구성 관리 도구로 재현 가능한 과정을 만들어야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

JAR를 실행할 때 환경별 설정은 환경 변수로 주입한다.

```bash
SPRING_PROFILES_ACTIVE=prod \
java -XX:MaxRAMPercentage=75.0 -jar /opt/my-api/app.jar
```

JVM 힙을 인스턴스 메모리 전체로 잡으면 운영체제와 모니터링 에이전트가 사용할 메모리가 부족해질 수 있다.

애플리케이션의 `/actuator/health`를 열고 이후 Part 4의 ALB Health Check와 연결하면 비정상 인스턴스로 요청이 전달되는 것을 막을 수 있다.

---

# 실무에서는 어떻게 사용할까

운영 API는 한 인스턴스에 의존하지 않고 둘 이상의 AZ에 분산하는 것이 기본 방향이다.

```
             ALB
          ┌───┴───┐
     AZ-a │       │ AZ-c
        EC2       EC2
          └───┬───┘
             RDS
```

인스턴스는 수동으로 고치는 장비가 아니라 동일한 설정으로 다시 만들 수 있는 교체 가능한 자원으로 취급한다.

배포 산출물과 설정을 버전 관리하고 Auto Scaling의 Launch Template으로 생성 조건을 표준화한다.

---

# 장애 사례와 확인 순서

| 증상 | 원인 후보 | 확인 방법 |
|---|---|---|
| 상태 검사는 통과하지만 API 불가 | 프로세스·포트·SG | `systemctl`, `ss`, 규칙 확인 |
| 갑자기 응답이 느림 | CPU, 메모리, 디스크 병목 | CloudWatch와 OS 지표 |
| Stop/Start 후 접속 주소 변경 | Public IP 재할당 | DNS 또는 Elastic IP 확인 |
| 부팅은 됐지만 앱 미실행 | User Data·Systemd 실패 | cloud-init과 journal 로그 |
| 인스턴스 종료 후 데이터 손실 | 임시 저장소 사용 | EBS 삭제 설정과 저장 위치 |

EC2 상태 검사와 애플리케이션 Health Check는 검사하는 계층이 다르므로 둘 다 확인해야 한다.

---

# 비용과 성능 고려사항

비용은 인스턴스 실행 시간, 타입, 운영체제, EBS, Public IPv4, 데이터 전송 등으로 나뉜다.

On-Demand는 유연하고, Savings Plans나 예약 약정은 지속적인 사용량에 적합하며, Spot은 중단을 견딜 수 있는 작업에 적합하다.

약정 구매 전에 실제 사용 패턴을 관찰하고, 유휴 인스턴스와 연결되지 않은 EBS·Elastic IP도 함께 정리해야 한다.

---

# 기억해야 할 내용

- EC2는 가상 서버다.
- 인스턴스 타입은 서버 성능을 결정한다.
- EC2는 AMI로 시작하고 EBS와 ENI가 붙는다.
- EC2를 인터넷에 노출할지는 Subnet, Route Table, Public IP, Security Group이 결정한다.
- Stop, Reboot, Terminate는 자원 생명주기와 데이터 보존 방식이 다르다.
- 운영 인스턴스는 수동 복구보다 재현 가능한 방식으로 교체할 수 있어야 한다.
- 인스턴스 비용뿐 아니라 스토리지와 주소, 데이터 전송도 확인한다.

---

# 다음 Chapter

다음 Chapter에서는 [AMI](/aws-backend/part-03/03-ami/)를 학습한다.

EC2가 어떤 운영체제와 초기 파일 시스템으로 시작하는지, 표준 이미지를 어떻게 관리하는지 알아본다.


