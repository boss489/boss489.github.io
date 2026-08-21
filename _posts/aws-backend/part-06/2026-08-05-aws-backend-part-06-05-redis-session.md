---
title: "Chapter 05. Redis Session"
permalink: /aws-backend/part-06/05-redis-session/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Redis Session
## 여러 서버가 세션을 공유하는 방식

> **학습 목표**
>
> - 서버 메모리 세션의 한계를 설명할 수 있다.
> - Redis Session의 역할을 이해한다.
> - 세션 저장소 장애가 로그인에 미치는 영향을 설명할 수 있다.
> - Spring Session Data Redis를 구성할 수 있다.

---

# 왜 Redis Session이 필요한가

Spring Boot 서버 한 대가 로그인 세션을 JVM 메모리에 저장할 때는 문제가 없어 보인다.

트래픽 증가로 서버를 세 대로 늘리면 ALB는 같은 사용자의 다음 요청을 다른 서버로 보낼 수 있다.

첫 서버에만 세션이 존재하면 다른 서버는 사용자를 비로그인 상태로 판단하여 간헐적인 로그아웃이 발생한다.

Sticky Session은 요청을 같은 서버로 유도하지만 서버 장애와 재배포 때 세션이 사라지고 트래픽 분산도 불균형해질 수 있다.

외부 Redis에 세션을 저장하면 모든 서버가 같은 로그인 상태를 조회하므로 애플리케이션을 상태 비저장에 가깝게 확장할 수 있다.

---

# Redis Session이란?

Redis Session은 HTTP 세션 데이터와 만료 정보를 애플리케이션 서버 밖의 Redis에 저장하는 방식이다.

브라우저는 세션 자체가 아니라 무작위 세션 ID가 든 쿠키를 보내고, 서버는 그 ID로 Redis에서 상태를 조회한다.

Spring Session은 Servlet의 `HttpSession` 구현을 대체하여 기존 코드 변경을 줄인다.

---

# 동작 흐름

```
Client             ALB          App A/App B        Redis
  │ POST /login     │               │                │
  ├────────────────>├──────────────>│                │
  │                 │               ├─ save session ─>│
  │<── Set-Cookie SESSION=<id> ─────┤                │
  │ GET /orders + Cookie            │                │
  ├────────────────>├──── App B ───>│                │
  │                 │               ├─ get session ──>│
  │                 │               │<─ user state ───┤
  │<──────────── authenticated ─────┤                │
```

1. 로그인 성공 후 서버가 세션 ID와 인증 상태를 Redis에 저장한다.
2. 응답 쿠키에는 세션 ID만 전달한다.
3. 다음 요청이 어느 서버에 도착하든 쿠키의 ID로 Redis 세션을 조회한다.
4. 마지막 접근 시간과 정책에 따라 세션 만료가 갱신된다.
5. 로그아웃 시 서버가 세션을 삭제하고 쿠키를 만료시킨다.

---

# 세션 방식 비교

| 항목 | 로컬 세션 | Redis Session | JWT Access Token |
|------|-----------|---------------|------------------|
| 서버 확장 | Sticky Session 필요 가능 | 여러 서버가 공유 | 상태 저장 없이 검증 |
| 강제 로그아웃 | 해당 서버에서 가능 | 중앙 삭제로 쉬움 | 별도 차단 목록 필요 |
| 저장소 장애 | 특정 서버 영향 | 전체 로그인 요청 영향 가능 | 서명 검증은 지속 가능 |
| 서버 재시작 | 세션 손실 | Redis가 유지되면 지속 | 토큰 만료까지 지속 |
| 네트워크 호출 | 없음 | 요청별 Redis 접근 가능 | 보통 없음 |

JWT가 항상 더 좋은 것은 아니며 강제 만료, 토큰 탈취 대응, 클레임 최신성 요구를 함께 비교해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot 3.x에서는 `spring-session-data-redis` 의존성을 추가하고 Redis 연결을 설정한다.

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
}
```

자동 구성을 사용할 때는 프로퍼티로 저장소와 세션 타임아웃을 지정할 수 있으며 명시적 애너테이션과 중복 구성하지 않도록 방식을 통일한다.

```yaml
spring:
  session:
    timeout: 30m
    redis:
      namespace: shop:session
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
      ssl:
        enabled: true
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: lax
```

애플리케이션 코드는 표준 `HttpSession`을 사용해도 실제 저장은 Redis가 담당한다.

```java
@RestController
@RequestMapping("/api/session")
public class SessionController {

    @PostMapping("/login")
    public ResponseEntity<Void> login(
            @RequestBody LoginRequest request,
            HttpSession session) {
        session.setAttribute("memberId", request.memberId());
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping
    public ResponseEntity<Void> logout(HttpSession session) {
        session.invalidate();
        return ResponseEntity.noContent().build();
    }
}
```

세션 속성은 직렬화되어 저장되므로 JPA Entity나 큰 객체 대신 최소 식별자와 작은 불변 DTO를 넣는다.

Spring Security를 사용하면 인증 컨텍스트도 세션에 저장될 수 있으므로 직렬화 호환성과 민감 정보 범위를 점검한다.

---

# 실무에서는 어떻게 사용할까

ALB 뒤의 ECS Task나 EC2 인스턴스가 수평 확장되는 서버 렌더링 웹과 관리 시스템에서 유용하다.

세션 전용 Redis를 일반 캐시와 분리하면 캐시 eviction이 로그인 상태를 제거하는 사고를 줄일 수 있다.

쿠키에는 `Secure`, `HttpOnly`, `SameSite`를 적용하고 HTTPS만 허용하며 세션 ID를 로그에 남기지 않는다.

관리자와 일반 사용자의 세션 만료 정책을 구분하고 비밀번호 변경이나 계정 잠금 시 관련 세션을 폐기할 수 있어야 한다.

---

# 장애 사례

로컬 세션 상태로 서버를 확장하면 요청이 다른 서버에 도착할 때마다 로그인이 풀리는 것처럼 보인다.

Redis Failover 중 기존 연결이 끊기면 일부 요청에서 세션 조회가 실패할 수 있으므로 클라이언트 재연결, 짧은 타임아웃, 재시도 범위를 검증해야 한다.

일반 캐시와 세션을 같은 노드에 두고 `allkeys-lru`를 사용하면 메모리 압박 때 활성 세션도 eviction될 수 있다.

세션에 큰 장바구니나 객체 그래프를 저장하면 요청마다 직렬화와 네트워크 비용이 커지고 배포 후 역직렬화 실패가 발생할 수 있다.

---

# 주의할 점

Redis Session은 Redis를 인증 경로의 핵심 의존성으로 만들기 때문에 Multi-AZ와 자동 Failover, 모니터링을 검토해야 한다.

Failover가 있어도 연결 전환 동안 오류가 0이 된다는 뜻은 아니므로 애플리케이션의 사용자 경험을 설계한다.

세션 고정 공격을 막기 위해 로그인 성공 시 세션 ID를 교체하고 CSRF 보호를 유지한다.

세션 TTL이 너무 길면 탈취된 세션의 유효 기간이 늘어나고 너무 짧으면 정상 사용자가 자주 로그아웃된다.

직렬화 형식 변경은 롤링 배포의 구버전·신버전 호환성을 고려한다.

---

# 비용과 성능 고려사항

요청마다 세션을 읽고 변경 시 쓰기 때문에 연결 수, 네트워크 왕복, 직렬화 크기가 응답 시간에 영향을 준다.

ElastiCache 비용은 노드 메모리와 수, 복제본, Multi-AZ, 백업, 데이터 전송량에 따라 달라진다.

세션 수, 평균 세션 크기, 생성률, 만료율을 측정하여 필요한 메모리와 연결 용량을 산정한다.

모든 요청에서 세션을 불필요하게 변경하면 쓰기가 증가하므로 세션에 저장하는 값을 최소화한다.

---

# 기억해야 할 내용

- 로컬 세션은 수평 확장과 서버 재시작에 취약하다.
- Redis Session은 여러 서버가 중앙 세션 상태를 공유하게 한다.
- Spring Session Data Redis는 표준 `HttpSession`을 Redis 기반으로 대체한다.
- 세션에는 최소 식별자만 저장하고 민감 정보와 큰 객체를 피한다.
- 세션 TTL은 보안과 사용자 경험의 균형이다.
- Redis 장애와 eviction은 로그인 상태에 직접 영향을 줄 수 있다.
- JWT와 Redis Session은 강제 만료와 저장소 의존성을 기준으로 선택한다.

---

# 다음 Chapter

다음 Chapter에서는 여러 서버가 같은 작업을 동시에 실행하지 못하게 하는 **[Distributed Lock](/aws-backend/part-06/06-distributed-lock/)** 을 학습한다.

Redis 락의 획득과 안전한 해제, 만료 시간, 더 단순한 DB 대안을 살펴본다.


