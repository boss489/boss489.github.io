---
title: "Chapter 08. Session and Keep Alive"
permalink: /aws-backend/part-04/08-session-keep-alive/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Session and Keep Alive
## 요청 연결과 사용자 상태 관리

> **학습 목표**
>
> - 서버 세션과 Sticky Session의 문제를 이해한다.
> - Keep Alive가 연결 재사용에 미치는 영향을 설명할 수 있다.
> - Scale Out 환경에서 세션을 어떻게 다뤄야 하는지 이해한다.
> - ALB와 애플리케이션의 Idle Timeout 불일치 문제를 설명할 수 있다.

---

# 왜 Session과 Keep Alive를 함께 알아야 하는가

쇼핑몰 API 서버가 한 대일 때는 로그인 세션을 서버 메모리에 저장해도 같은 프로세스가 모든 요청을 처리한다.

[Auto Scaling](/aws-backend/part-04/07-auto-scaling/)으로 서버가 세 대가 되면 다음 요청은 다른 서버에 전달될 수 있다.

이때 사용자 상태를 공유하지 않으면 로그인이나 장바구니가 사라진 것처럼 보인다.

한편 Client, [ALB](/aws-backend/part-04/04-alb/), Spring Boot 사이의 연결을 매 요청마다 새로 만들면 TCP와 TLS 연결 비용이 반복된다.

Session은 **여러 요청 사이의 사용자 상태**, Keep Alive는 **여러 요청에서 연결을 재사용하는 방법**을 다룬다.

두 개념은 서로 다르지만 Scale Out 환경의 안정성과 성능을 함께 결정한다.

---

# Session이란?

서버 세션은 사용자 상태를 서버 메모리에 저장하는 방식이다.

서버가 여러 대가 되면 사용자의 다음 요청이 다른 서버로 갈 수 있다.

이 경우 세션을 찾지 못해 로그인 상태가 풀릴 수 있다.

일반적인 서버 세션은 Client 쿠키에 Session ID를 저장하고 실제 데이터는 서버에 저장한다.

Session ID는 인증 정보 자체가 아니라 서버의 세션 데이터를 찾는 식별자이다.

---

| 방식 | 상태 위치 | 장점 | 단점 |
|---|---|---|---|
| Local Session | 각 서버 메모리 | 단순하고 빠름 | 서버 간 공유 불가 |
| Sticky Session | 각 서버 메모리 | 기존 구조 변경이 적음 | 부하 편중, 장애 시 손실 |
| Redis Session | 외부 Redis | 서버 간 공유, 수평 확장 | Redis 운영 필요 |
| Token/JWT | Client가 Token 보관 | 서버 Stateless 구성 | 폐기와 크기 관리 필요 |

---

# Sticky Session

Sticky Session은 같은 사용자의 요청을 같은 서버로 보내는 방식이다.

간단하지만 특정 서버에 부하가 몰릴 수 있고 장애 시 세션이 사라질 수 있다.

ALB는 Cookie 기반 Stickiness를 Target Group 속성으로 제공한다.

Sticky Session은 임시 호환 수단으로 사용할 수 있지만 무상태 애플리케이션이나 외부 세션 저장소가 확장에 더 유리하다.

---

# Spring Session과 Redis

Spring Session Data Redis를 사용하면 `HttpSession` 데이터를 Redis에 저장할 수 있다.

각 Spring Boot 인스턴스가 같은 Redis를 사용하므로 어느 Target이 요청을 받아도 같은 세션을 조회한다.

```
            ┌── App A ──┐
Client ─ ALB├── App B ──┼── Redis Session
            └── App C ──┘
```

Maven에 Spring Session Redis Starter를 추가한다.

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Redis 연결과 세션 저장소를 설정한다.

```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
  data:
    redis:
      host: redis.internal.example.com
      port: 6379
```

운영에서는 ElastiCache 연결의 Security Group, 암호화, 인증, 장애 조치와 Timeout 정책을 함께 구성한다.

세션 객체는 직렬화되므로 불필요하게 큰 객체나 민감 정보를 저장하지 않는다.

---

# Keep Alive란?

Keep Alive는 TCP 연결을 요청마다 새로 만들지 않고 재사용하는 방식이다.

연결 생성 비용을 줄이지만, 너무 오래 유지하면 리소스를 점유할 수 있다.

```
연결 재사용 없음
Request 1: TCP + TLS + HTTP
Request 2: TCP + TLS + HTTP

Keep Alive
TCP + TLS
  ├── Request 1
  ├── Request 2
  └── Request 3
```

HTTP/1.1에서는 지속 연결이 일반적이며 여러 요청이 같은 TCP 연결을 순차적으로 사용할 수 있다.

ALB 환경에는 Client와 ALB 사이 연결과 ALB와 Target 사이 연결이 별도로 존재한다.

```
Client
  │ Keep Alive A
  ▼
ALB
  │ Keep Alive B
  ▼
Spring Boot
```

한쪽 연결이 재사용된다고 다른 쪽도 같은 수명으로 유지되는 것은 아니다.

---

# Idle Timeout

Idle Timeout은 연결에서 데이터가 오가지 않을 때 유지하는 시간을 의미한다.

ALB의 기본 Idle Timeout은 널리 알려진 기준으로 60초이며 요구사항에 따라 조정할 수 있다.

Spring Boot 내장 서버, Reverse Proxy, HTTP Client도 각각 연결 관련 Timeout을 가진다.

| Timeout | 범위 | 목적 |
|---|---|---|
| Connect Timeout | 연결 수립 | 연결 불가능 대상에서 빠르게 실패 |
| Read Timeout | 응답 데이터 대기 | 느린 요청 제한 |
| Idle Timeout | 유휴 연결 유지 | 연결 재사용과 자원 회수 균형 |
| Request Timeout | 전체 요청 처리 | 장시간 작업 제한 |

서로 다른 Timeout을 하나의 값으로 생각하면 장애 원인을 잘못 판단할 수 있다.

---

# Timeout 불일치와 ALB 502

애플리케이션 서버가 ALB보다 먼저 Keep Alive 연결을 닫았는데 ALB가 그 연결을 재사용하려 하면 `502 Bad Gateway`가 발생할 수 있다.

```
1. ALB ───── App 연결 생성
2. App ──X── 연결 먼저 종료
3. ALB ───── 기존 연결 재사용 시도
4. Client ←─ 502
```

일반적으로 Target의 Keep Alive Timeout을 ALB Idle Timeout보다 충분히 길게 설정하여 ALB가 먼저 유휴 연결을 정리하게 한다.

정확한 설정 항목과 동작은 사용하는 Tomcat, Jetty, Netty 버전과 HTTP 프로토콜에 따라 확인해야 한다.

장시간 API가 ALB Idle Timeout보다 오래 아무 데이터도 전송하지 않으면 ALB가 연결을 닫아 `504 Gateway Timeout`이 발생할 수 있다.

---

# 실무에서는 어떻게 사용할까

로그인 기반 쇼핑몰은 세션을 Redis에 외부화하여 서버 추가와 교체가 사용자 상태에 영향을 주지 않게 한다.

애플리케이션 서버는 가능한 한 무상태로 유지하고 업로드 파일이나 작업 상태도 공유 저장소에 둔다.

ALB와 Target의 연결 Timeout은 장시간 요청과 서버 설정을 기준으로 정렬한다.

---

# 장애 사례와 주의할 점

## Scale Out 후 로그인이 간헐적으로 풀리는 경우

세션이 각 서버 메모리에 있고 Stickiness도 적용되지 않았을 가능성이 크다.

요청별 Target과 Session ID를 추적하고 Redis 같은 공유 저장소로 외부화한다.

## 특정 Target에만 부하가 몰리는 경우

긴 Sticky Cookie와 사용자별 요청량 차이로 분산이 불균형할 수 있다.

Target별 요청 수를 확인하고 세션 외부화 후 Stickiness 제거를 검토한다.

## 간헐적인 ALB 502가 발생하는 경우

Target 프로세스 종료, 잘못된 HTTP 응답과 함께 Keep Alive Timeout 불일치를 확인한다.

ALB Access Log의 오류, 애플리케이션 종료 로그와 서버 연결 설정을 같은 시간대로 비교한다.

## Redis 장애가 로그인 장애로 확대되는 경우

세션 저장소가 중앙 의존성이므로 고가용성, 모니터링, 적절한 연결 Timeout이 필요하다.

세션 데이터의 TTL과 장애 시 사용자 재로그인 정책도 명확히 정한다.

---

# 기억해야 할 내용

- Session은 요청 사이의 사용자 상태이고 Keep Alive는 네트워크 연결 재사용이다.
- Scale Out 환경에서 서버 메모리 세션은 로그인 유실과 서버 종속을 만든다.
- Sticky Session은 단순하지만 장애와 부하 분산에 약점이 있다.
- Spring Session과 Redis를 사용하면 여러 서버가 세션을 공유할 수 있다.
- Keep Alive는 TCP와 TLS 연결 생성 비용을 줄인다.
- Target의 Keep Alive Timeout과 ALB Idle Timeout 불일치는 간헐적 502를 만들 수 있다.
- 세션과 연결 설정은 실제 장애 전환과 부하 테스트로 검증해야 한다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 09. Part 4 Summary**이다.

DNS에서 Spring Boot까지의 요청 흐름과 ALB, Target Group, Health Check, Auto Scaling의 관계를 하나의 운영 관점으로 정리한다.

