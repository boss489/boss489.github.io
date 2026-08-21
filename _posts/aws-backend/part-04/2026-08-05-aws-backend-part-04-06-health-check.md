---
title: "Chapter 06. Health Check"
permalink: /aws-backend/part-04/06-health-check/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Health Check
## 요청을 받을 수 있는 서버인지 확인하기

> **학습 목표**
>
> - Health Check의 목적을 설명할 수 있다.
> - Health Check 실패가 트래픽 흐름에 미치는 영향을 이해한다.
> - Spring Boot에서 Health Check 엔드포인트를 설계하는 기준을 설명할 수 있다.
> - Liveness와 Readiness의 차이를 구분할 수 있다.

---

# 왜 Health Check가 필요한가

Spring Boot 프로세스가 실행 중이어도 초기화가 끝나지 않았거나 연결 풀이 고갈되어 요청을 처리하지 못할 수 있다.

이 서버에 [ALB](/aws-backend/part-04/04-alb/)가 계속 요청을 보내면 일부 사용자만 반복적으로 오류를 경험한다.

운영자가 수동으로 서버를 제외하는 방식은 느리고 [Auto Scaling](/aws-backend/part-04/07-auto-scaling/) 환경에서는 대상도 계속 바뀐다.

Health Check는 각 Target의 준비 상태를 자동으로 확인하여 비정상 대상을 요청 경로에서 제외한다.

---

# Health Check란?

Health Check는 ALB가 Target이 정상인지 주기적으로 확인하는 기능이다.

정상 대상에게만 사용자 요청을 전달한다.

```
ALB
├── GET /actuator/health/readiness → App A: 200 Healthy
├── GET /actuator/health/readiness → App B: 503 Unhealthy
└── GET /actuator/health/readiness → App C: 200 Healthy

사용자 요청 → App A 또는 App C
```

Health Check는 알림만 만드는 기능이 아니라 실제 라우팅 대상을 바꾼다.

Target Group마다 프로토콜, 포트, 경로, 검사 주기, Timeout, 임계값과 성공 코드를 설정한다.

---

# 내부 동작 원리

각 ALB 노드는 등록된 Target에 주기적으로 Health Check 요청을 보낸다.

연속 성공 횟수가 Healthy Threshold에 도달하면 Target을 정상으로 판단한다.

연속 실패 횟수가 Unhealthy Threshold에 도달하면 해당 Target으로의 요청 전달을 중단한다.

```
Initial
  │ 연속 성공
  ▼
Healthy
  │ 연속 실패
  ▼
Unhealthy
  │ 다시 연속 성공
  └──────────────→ Healthy
```

한 번의 일시적 실패로 상태가 계속 바뀌는 것을 막기 위해 임계값을 사용한다.

| 설정 | 의미 | 잘못 설정했을 때 |
|---|---|---|
| Protocol | 검사 통신 방식 | HTTP와 HTTPS 불일치 |
| Port | 요청을 보낼 포트 | 연결 거부 |
| Path | 검사 Endpoint | 404 응답 |
| Timeout | 응답 대기 시간 | 느린 서버 오판 |
| Interval | 검사 주기 | 감지 지연 또는 요청 증가 |
| Healthy Threshold | 정상 전환 성공 횟수 | 배포 투입 지연 |
| Unhealthy Threshold | 비정상 전환 실패 횟수 | 제거 지연 |
| Success Codes | 정상 HTTP 코드 범위 | Redirect를 실패로 판정 |

---

# Liveness와 Readiness

Liveness는 프로세스를 재시작해야 하는 상태인지 판단한다.

Readiness는 현재 사용자 요청을 받을 준비가 되었는지 판단한다.

| 구분 | Liveness | Readiness |
|---|---|---|
| 핵심 질문 | 프로세스를 재시작해야 하는가 | 트래픽을 받아도 되는가 |
| 실패 시 동작 | 재시작 고려 | 라우팅 대상에서 제외 |
| 외부 의존성 | 신중하게 포함 | 핵심 의존성 포함 가능 |
| ALB 연결 | 보통 사용하지 않음 | 일반적으로 적합 |

데이터베이스가 잠시 느리다는 이유로 Liveness까지 실패시키면 모든 인스턴스가 동시에 재시작되는 연쇄 장애가 생길 수 있다.

ALB에는 사용자 요청 처리 가능성을 나타내는 Readiness Endpoint를 연결하는 것이 적합하다.

---

# Spring Boot Actuator 연동

Spring Boot 3.x에서는 Actuator의 Health Endpoint와 Probe Group을 사용할 수 있다.

Maven에 Actuator Starter를 추가한다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Health Endpoint를 노출하고 Probe를 활성화한다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never
```

대표 Endpoint는 다음과 같다.

```text
/actuator/health/liveness
/actuator/health/readiness
```

ALB Health Check 경로에는 `/actuator/health/readiness`를 연결할 수 있다.

`show-details: never`는 응답에 내부 컴포넌트와 오류 정보를 노출하지 않도록 돕는다.

---

# 보안 구성

Health Endpoint가 인증을 요구하면 ALB가 `401` 또는 `403`을 받아 Target을 비정상으로 판단할 수 있다.

Health Check 경로만 허용하되 상세 관리 Endpoint 전체를 공개해서는 안 된다.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(authorize -> authorize
            .requestMatchers("/actuator/health/**").permitAll()
            .anyRequest().authenticated())
        .build();
}
```

네트워크에서는 App Security Group이 ALB Security Group에서 오는 애플리케이션 포트만 허용한다.

---

# 배포와 Health Check

새 버전을 배포했는데 Health Check가 실패하면 ALB는 해당 대상에 트래픽을 보내지 않는다.

이 동작은 초기화되지 않은 서버가 사용자 요청을 받는 것을 막는다.

```
새 Target 시작
  └── Spring Boot 초기화
      └── Readiness 수락
          └── Health Check 통과
              └── 사용자 요청 수신
```

배포 시스템은 새 Target이 정상 상태가 된 뒤 구버전 Target을 제거해야 한다.

---

# 실무에서는 어떻게 사용할까

상품 API는 애플리케이션 초기화와 필수 의존성이 준비된 뒤 Readiness를 수락한다.

CloudWatch에서는 `HealthyHostCount`와 `UnHealthyHostCount`를 모니터링한다.

정상 Target 수가 서비스 최소치 아래로 내려가면 운영자가 즉시 확인할 수 있도록 Alarm을 구성한다.

Health Endpoint는 실제 사용자 요청보다 단순하고 빠르게 응답하도록 설계한다.

---

# 장애 사례

## 모든 Target이 비정상이라 ALB 503이 발생하는 경우

Health Check 경로의 404, Security Group 차단, 포트 불일치, 시작 실패를 확인한다.

대상 상태의 Reason Code와 Spring Boot 시작 로그를 함께 본다.

## Redirect 때문에 실패하는 경우

HTTP Health Check가 HTTPS Redirect를 반환하지만 성공 코드를 `200`만 허용하면 비정상으로 판정된다.

검사 프로토콜을 맞추거나 Health Check 경로가 Redirect되지 않게 구성한다.

## 인증 적용 후 비정상이 된 경우

Spring Security 변경으로 Endpoint가 `401`을 반환할 수 있다.

Health 경로의 접근 정책을 명시하고 실제 ALB 경로에서 응답을 검증한다.

## 의존성 장애로 전체 Target이 제외되는 경우

공유 의존성을 Readiness에 포함하면 모든 서버가 동시에 비정상이 될 수 있다.

어떤 의존성을 상태 판정에 포함할지 장애 시나리오로 검증한다.

---

# 기억해야 할 내용

- Health Check는 Target이 현재 트래픽을 받을 수 있는지 확인한다.
- 경로, 포트, Timeout, 임계값과 성공 코드가 상태 판정에 관여한다.
- Liveness는 재시작 판단, Readiness는 트래픽 수신 판단에 사용한다.
- ALB에는 일반적으로 Actuator의 Readiness Endpoint를 연결한다.
- Health Endpoint는 상세 정보를 노출하지 않고 빠르게 응답해야 한다.
- 모든 Target이 비정상이면 ALB 503이 발생할 수 있다.
- 배포 시 새 Target의 Health Check 통과 후 구버전을 제거한다.

---

# 다음 Chapter

다음 Chapter에서는 트래픽 변화에 맞춰 정상 Target 수를 조절하는 **Auto Scaling**을 학습한다.

Capacity와 Scaling Policy, 준비 시간과 Graceful Shutdown을 함께 설계하는 방법을 살펴본다.

