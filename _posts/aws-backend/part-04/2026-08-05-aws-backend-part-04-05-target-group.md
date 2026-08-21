---
title: "Chapter 05. Target Group"
permalink: /aws-backend/part-04/05-target-group/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Target Group
## ALB가 요청을 보낼 대상 묶음

> **학습 목표**
>
> - Target Group의 역할을 설명할 수 있다.
> - Target Type의 차이를 이해한다.
> - Target Group과 Health Check의 관계를 설명할 수 있다.
> - 등록 해제 지연이 배포와 종료에 미치는 영향을 이해한다.

---

# 왜 Target Group이 필요한가

쇼핑몰 API를 처리하는 Spring Boot 서버가 세 대라고 가정해 보자.

[ALB](/aws-backend/part-04/04-alb/)는 `/api/*` 요청을 받아도 어떤 서버의 어느 포트로 보낼지 알아야 한다.

배포나 [Auto Scaling](/aws-backend/part-04/07-auto-scaling/)으로 서버 수가 계속 바뀌므로 Listener Rule에 서버 주소를 하나씩 넣는 방식은 운영할 수 없다.

Target Group은 요청을 처리할 대상을 하나의 논리적 묶음으로 관리하고 ALB Rule과 실제 서버를 분리한다.

서버가 추가되거나 제거되어도 Rule은 같은 Target Group을 계속 가리킬 수 있다.

---

# Target Group이란?

Target Group은 ALB가 요청을 전달할 대상들의 묶음이다.

대상은 EC2, IP, Lambda 등이 될 수 있다.

ECS에서는 보통 Task IP가 Target으로 등록된다.

```
ALB Listener :443
      │
      └── Rule: /api/*
              │
              ▼
        Target Group :8080
        ├── Target A: Healthy
        ├── Target B: Healthy
        └── Target C: Unhealthy
```

ALB는 정상으로 판단된 Target A와 B에만 요청을 전달한다.

Target Group은 대상 목록뿐 아니라 프로토콜, 포트, Health Check와 등록 해제 동작도 함께 정의한다.

---

# Target Type

| Type | 등록 단위 | 대표 용도 |
|---|---|---|
| `instance` | EC2 Instance ID | EC2 기반 애플리케이션 |
| `ip` | 대상의 IP 주소 | ECS `awsvpc`, 온프레미스 연동 |
| `lambda` | Lambda 함수 | 서버리스 HTTP 처리 |
| `alb` | 다른 ALB | NLB 뒤에 ALB 연결 |

ECS Fargate는 `ip` 타입을 사용한다.

Target Type은 생성 후 변경할 수 없으므로 실행 환경에 맞게 새 Target Group을 만들어야 한다.

`instance` 타입에서는 트래픽이 인스턴스의 지정 포트로 전달된다.

`ip` 타입에서는 각 ECS Task의 ENI 사설 IP처럼 개별 IP가 Target이 된다.

---

# 프로토콜과 포트

Target Group은 ALB가 Target과 통신할 프로토콜과 기본 포트를 가진다.

Spring Boot가 `8080`으로 실행되면 Target Group도 해당 포트로 요청을 보내도록 구성해야 한다.

```
Client HTTPS :443
       │
       ▼
ALB TLS Termination
       │ HTTP :8080
       ▼
Spring Boot
```

Client와 ALB 사이의 Listener 포트와 ALB에서 Target으로 가는 포트는 서로 다를 수 있다.

| 구간 | 대표 프로토콜과 포트 | 담당 설정 |
|---|---|---|
| Client → ALB | HTTPS `443` | Listener |
| ALB → Target | HTTP `8080` | Target Group |
| ALB → Health Endpoint | HTTP `8080` | Health Check |

내부 구간도 암호화해야 하는 요구사항이 있다면 Target Group에 HTTPS를 사용할 수 있지만 인증서와 애플리케이션 구성이 함께 필요하다.

---

# Target 등록과 해제

Target은 등록 직후 Health Check를 통과해야 정상 상태가 된다.

Target을 제거하면 ALB는 새 요청 전달을 중단하고 진행 중인 요청을 마칠 시간을 준다.

이 대기 동작을 등록 해제 지연(Deregistration Delay) 또는 Connection Draining 관점에서 이해할 수 있다.

```
Target 등록
  └── Initial
      └── Health Check 통과
          └── Healthy
              └── 요청 수신

Target 해제
  └── Draining
      └── 진행 중 요청 정리
          └── Deregistered
```

지연 시간이 너무 짧으면 긴 요청이 중단될 수 있고 너무 길면 배포와 Scale In 완료가 늦어진다.

---

# Health Check와의 관계

[Health Check](/aws-backend/part-04/06-health-check/)는 Target이 실제 요청을 받을 준비가 되었는지 판단한다.

ALB는 각 Target을 독립적으로 검사하고 상태가 정상인 Target만 라우팅 대상으로 사용한다.

| 상태 | 의미 | 확인할 항목 |
|---|---|---|
| `initial` | 등록 후 검사 중 | 시작 시간, 경로 |
| `healthy` | 요청 전달 가능 | 정상 상태 |
| `unhealthy` | 검사 실패 | 응답 코드, Timeout |
| `draining` | 등록 해제 진행 | 종료 시간, 진행 요청 |
| `unused` | 사용 조건 불충족 | Listener 연결, AZ |

모든 Target이 비정상이면 사용자는 ALB에서 `503 Service Unavailable`을 받을 수 있다.

---

# AWS 콘솔에서는

일반적인 생성 순서는 다음과 같다.

1. EC2 콘솔의 `Target Groups`에서 Target Type을 선택한다.
2. 프로토콜과 애플리케이션 포트를 입력한다.
3. Target이 있는 VPC를 선택한다.
4. Health Check 경로와 성공 조건을 설정한다.
5. EC2 또는 IP Target을 등록한다.
6. ALB Listener Rule의 Forward Action에 연결한다.

Target Group만 생성하고 Listener Rule에 연결하지 않으면 사용자 요청은 전달되지 않는다.

---

# Spring Boot에서는 어떻게 쓰는가

애플리케이션 수신 포트와 Target Group 포트를 일치시킨다.

```yaml
server:
  port: 8080
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Graceful Shutdown은 Scale In이나 배포 중 새 요청을 받지 않게 된 서버가 처리 중인 요청을 마치는 데 도움을 준다.

애플리케이션 종료 제한 시간과 Target Group 등록 해제 지연을 함께 설계해야 한다.

---

# 실무에서는 어떻게 사용할까

주문 API와 상품 API를 서로 다른 Target Group으로 분리하면 독립적으로 배포하고 확장할 수 있다.

ALB Rule은 `/orders/*`와 `/products/*`를 각각의 Target Group으로 전달한다.

Blue/Green 배포에서는 구버전과 신버전 Target Group을 만들고 Forward Action의 비율이나 배포 서비스로 트래픽을 전환한다.

여러 AZ에 Target을 분산하여 한 AZ의 서버가 실패해도 다른 AZ가 요청을 처리하도록 구성한다.

---

# 장애 사례와 주의할 점

## Target이 계속 비정상인 경우

애플리케이션 포트, Health Check 경로, Security Group, 성공 응답 코드를 확인한다.

특히 App Security Group은 ALB Security Group에서 오는 Target 포트를 허용해야 한다.

## ALB 503이 발생하는 경우

Rule이 가리키는 Target Group에 정상 Target이 없거나 Target 등록이 완료되지 않았을 수 있다.

`HealthyHostCount`와 대상 상태 사유를 먼저 확인한다.

## 배포 중 요청이 끊기는 경우

프로세스를 먼저 종료한 뒤 Target을 해제하거나 Graceful Shutdown 시간이 지나치게 짧을 수 있다.

Target 등록 해제, 새 요청 중단, 진행 중 요청 완료, 프로세스 종료 순서를 맞춰야 한다.

---

# 기억해야 할 내용

- Target Group은 ALB 뒤의 대상 묶음이다.
- Listener Rule은 Target Group을 가리키고 Target Group은 실제 서버를 관리한다.
- 실행 환경에 따라 `instance`, `ip`, `lambda` 같은 Target Type을 선택한다.
- Target Group 포트와 애플리케이션 수신 포트가 맞아야 한다.
- Health Check가 정상인 대상만 트래픽을 받는다.
- Target 해제 시 진행 중인 요청을 위한 등록 해제 지연이 적용된다.
- Graceful Shutdown과 등록 해제 시간을 함께 설계해야 한다.
- 모든 Target이 비정상이면 ALB 503이 발생할 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 Target이 요청을 받을 준비가 되었는지 판단하는 **Health Check**를 학습한다.

검사 설정과 상태 전환, Spring Boot Actuator를 안전한 준비 상태 Endpoint로 연결하는 방법을 살펴본다.

