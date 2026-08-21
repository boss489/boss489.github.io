---
title: "Chapter 02. ALB 502"
permalink: /aws-backend/part-15/02-alb-502/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. ALB 502
## ALB와 Target 사이의 HTTP 교환 실패

> **학습 목표**
>
> - ALB가 생성한 502와 Target 응답 502를 구분할 수 있다.
> - Access Log 필드로 실패 경계를 찾을 수 있다.
> - 연결, HTTP, timeout과 TLS 가설을 안전하게 검증할 수 있다.

---

# 실제 장애 징후

사용자는 `502 Bad Gateway`를 받고 ALB의 `HTTPCode_ELB_5XX_Count`가 증가할 수 있다.

일부 Target이나 특정 배포 버전에서만 오류가 발생하면 간헐적인 502로 나타난다.

Target 로그에 요청이 없거나 응답 완료 기록이 없다면 ALB와 Target 사이를 우선 조사한다.

---

# ALB 502의 정의와 가능한 원인

ALB 502는 ALB가 선택한 Target과 정상적인 HTTP 교환을 완료하지 못했음을 뜻한다.

대표 원인은 연결 거부, 연결 reset, 잘못된 HTTP 응답, idle timeout 불일치와 Target HTTPS의 TLS 문제다.

Target이 명시적으로 502를 반환한 경우에는 ALB 전달 실패가 아니라 애플리케이션 또는 그 하위 프록시의 응답이다.

ALB Access Log의 `elb_status_code`와 `target_status_code`를 비교해야 두 경우를 구분할 수 있다.

`elb_status_code=502`이고 `target_status_code=-`이면 Target 응답 코드를 받기 전 실패했을 가능성이 크다.

두 필드가 모두 502이면 Target이 반환한 502일 수 있으므로 처리 시간과 애플리케이션 로그를 확인한다.

---

# 계층 구조

```text
Client
  |
ALB listener
  |  elb_status_code
Target connection
  |  target_status_code
Spring Boot connector
  |
Controller and dependency
```

502는 요청 경로의 어느 경계에서 유효한 응답이 깨졌는지 찾아야 해결할 수 있다.

---

# 증거 기반 진단 순서

## 1. 영향을 확인한다

오류 시작 시각, URL, Host, Target AZ, 배포 버전과 전체 요청 대비 비율을 기록한다.

## 2. 최근 변경을 확인한다

배포, Target 포트, 프로토콜, 인증서, keep-alive와 ALB idle timeout 변경을 확인한다.

## 3. 메트릭과 Access Log를 확인한다

`HTTPCode_ELB_5XX_Count`, `HTTPCode_Target_5XX_Count`, `TargetConnectionErrorCount`와 `TargetResponseTime`을 같은 시간축에서 본다.

Access Log의 `request`, `target:port`, `request_processing_time`, `target_processing_time`, `response_processing_time`, `elb_status_code`, `target_status_code`, `error_reason`을 확인한다.

```sql
fields @timestamp, request_url, target_ip, target_port,
       elb_status_code, target_status_code, error_reason
| filter elb_status_code = 502
| stats count() by target_ip, target_status_code, error_reason
| sort count() desc
```

로그 저장 형식에 따라 실제 필드 파싱 규칙을 먼저 적용해야 한다.

## 4. Target 연결과 응답을 검증한다

Target 내부에서 애플리케이션이 등록 포트에 listen하는지 확인한다.

```bash
ss -lntp
curl --verbose --max-time 5 http://127.0.0.1:8080/actuator/health/readiness
```

ALB 보안 그룹에서 Target 보안 그룹으로 등록 포트가 허용되는지 확인한다.

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN"
```

## 5. 가설을 구분한다

연결 거부는 프로세스 미기동, 잘못된 포트 또는 빠른 종료와 함께 나타난다.

연결 reset은 애플리케이션 종료, 프록시 종료, keep-alive 수명 불일치 또는 중간 장비의 연결 종료와 연관될 수 있다.

malformed response는 잘못된 status line, header 또는 HTTP 프로토콜 처리 문제를 의심한다.

ALB idle timeout보다 서버나 프록시의 keep-alive가 짧으면 재사용 연결이 예기치 않게 닫힐 수 있다.

HTTPS Target에서는 프로토콜 설정과 Target 인증서의 TLS 협상 가능 여부를 확인한다.

## 6. 안전하게 완화한다

오류가 특정 Target과 버전에 집중되면 해당 버전으로의 신규 배포를 중단하고 검증된 버전으로 되돌린다.

Target을 제외할 때는 남은 용량과 connection draining을 확인해 부하 집중을 피한다.

---

# Spring Boot 관찰 포인트

Tomcat, Jetty 또는 Netty의 활성 연결, 요청 처리 스레드, rejected 요청과 종료 로그를 확인한다.

`server.tomcat.keep-alive-timeout` 같은 서버 설정과 ALB idle timeout의 관계를 검토한다.

graceful shutdown과 readiness를 사용해 종료 중인 인스턴스가 새 요청을 받지 않게 한다.

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

의존 서비스 예외가 애플리케이션의 502 응답으로 매핑되었다면 trace와 예외 유형을 함께 기록한다.

---

# 대응과 복구

포트나 프로토콜 불일치는 Target Group 설정과 애플리케이션 listen 설정을 일치시켜 복구한다.

timeout 불일치는 각 홉의 의미와 수명을 확인한 뒤 서버가 연결을 먼저 닫지 않도록 조정한다.

복구 후 ALB 생성 5xx와 Target 5xx가 모두 정상 수준으로 돌아오는지 확인한다.

---

# 재발 방지

ALB 5xx와 Target 5xx를 별도 경보와 대시보드로 관리한다.

배포 전 Target 포트, health endpoint, graceful shutdown과 프로토콜을 자동 검증한다.

Access Log를 활성화하고 필드 파서를 버전 관리해 장애 시 즉시 검색할 수 있게 한다.

---

# 기억해야 할 내용

- ALB 502는 Target과 정상 HTTP 교환을 완료하지 못했을 때 발생할 수 있다.
- `elb_status_code`와 `target_status_code`가 실패 경계를 구분하는 핵심 증거다.
- 연결 reset, 거부, malformed response, timeout과 TLS를 각각 검증한다.
- Target이 반환한 502는 애플리케이션 계층에서 조사한다.
- 특정 Target 제외 전 남은 처리 용량을 확인한다.

---

# 다음 Chapter

다음 장에서는 [ALB 503 장애](/aws-backend/part-15/03-alb-503/)를 분석한다.
