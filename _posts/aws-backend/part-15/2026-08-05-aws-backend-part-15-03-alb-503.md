---
title: "Chapter 03. ALB 503"
permalink: /aws-backend/part-15/03-alb-503/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. ALB 503
## 사용할 수 있는 Target이 없는 상태

> **학습 목표**
>
> - ALB 503과 애플리케이션 503을 구분할 수 있다.
> - Target 등록과 health 상태를 증거로 조사할 수 있다.
> - 용량을 확인하며 안전하게 가용 Target을 복구할 수 있다.

---

# 실제 장애 징후

사용자는 ALB를 통해 요청할 때 `503 Service Unavailable`을 받는다.

ALB의 `HTTPCode_ELB_5XX_Count`가 증가하고 Target Group의 HealthyHostCount가 감소할 수 있다.

배포 직후 모든 Target이 동시에 draining 또는 unhealthy가 되면 전체 요청이 실패할 수 있다.

---

# ALB 503의 정의와 가능한 원인

ALB가 전달에 사용할 수 있는 Target을 찾지 못하면 ALB 자체가 503을 반환할 수 있다.

Target 미등록, 잘못된 Target Group 전달 규칙, 모든 Target의 unhealthy 또는 unused 상태가 대표 원인이다.

Target 유형과 네트워크, health check 포트, 경로, 성공 코드 불일치도 가용 Target을 없앨 수 있다.

과부하된 애플리케이션이 직접 503을 반환하는 경우에는 가용 Target이 없다는 뜻이 아니다.

Access Log에서 `elb_status_code=503`과 `target_status_code=-`이면 ALB 자체 응답을 우선 의심한다.

`target_status_code=503`이면 Target까지 요청이 도달했으므로 애플리케이션 과부하나 의존성 보호 로직을 조사한다.

---

# 계층 구조

```text
Client
  |
ALB listener rule
  |
Target Group
  |
registered + healthy Target
  |
Spring Boot readiness
```

Listener Rule이 올바른 Target Group을 가리켜도 그 그룹에 usable Target이 없으면 전달할 곳이 없다.

---

# 증거 기반 진단 순서

## 1. 영향을 확인한다

어떤 Host와 Path에서 503이 발생하는지 확인해 특정 Listener Rule과 Target Group으로 범위를 좁힌다.

## 2. 최근 변경을 확인한다

서비스 배포, Auto Scaling, Target 등록 해제, health check와 Listener Rule 변경 시간을 확인한다.

## 3. ALB 응답인지 확인한다

`HTTPCode_ELB_5XX_Count`와 `HTTPCode_Target_5XX_Count`를 비교하고 Access Log의 두 상태 코드 필드를 확인한다.

```sql
fields @timestamp, request_url, elb_status_code,
       target_status_code, target_ip, error_reason
| filter elb_status_code = 503
| stats count() by target_status_code, target_ip, error_reason
| sort count() desc
```

## 4. Target 등록과 health를 확인한다

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --query 'TargetHealthDescriptions[].{Target:Target,Health:TargetHealth}'
```

`State`, `Reason`, `Description`을 함께 읽어 미등록과 health check 실패를 구분한다.

```bash
aws elbv2 describe-rules --listener-arn "$LISTENER_ARN"
aws elbv2 describe-target-groups --target-group-arns "$TARGET_GROUP_ARN"
```

Listener Rule의 우선순위와 action이 기대한 Target Group을 가리키는지 확인한다.

## 5. health check를 검증한다

Target 내부에서 health check 포트와 경로가 성공하는지 확인한다.

```bash
curl --silent --show-error --max-time 5 \
  http://127.0.0.1:8080/actuator/health/readiness
ss -lntp
```

ALB 보안 그룹에서 Target 보안 그룹으로 health check 포트가 허용되는지 확인한다.

readiness가 필수 의존성의 일시 장애 때문에 모든 인스턴스를 동시에 제외하는 구조인지 검토한다.

## 6. 애플리케이션 503을 구분한다

Target 503이면 요청 스레드, 큐, 커넥션 풀, rate limit과 circuit breaker 상태를 확인한다.

HealthyHostCount가 정상인데 Target 503만 증가하면 단순 Target 등록보다 애플리케이션 포화를 우선 조사한다.

## 7. 안전하게 완화한다

검증된 이전 버전의 Target을 다시 확보하거나 잘못된 Rule과 health check 설정을 되돌린다.

Target을 한꺼번에 교체하지 않고 최소 정상 용량을 유지하면서 단계적으로 복구한다.

---

# Spring Boot 관찰 포인트

readiness는 새 요청을 처리할 준비 상태를 나타내고 liveness는 프로세스 재시작 판단에 사용한다.

두 probe를 같은 과도한 의존성 검사로 구성하면 외부 장애가 전체 Target 제외와 재시작 폭풍으로 번질 수 있다.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

애플리케이션 503에는 보호 장치 이름, 현재 포화 자원과 Retry-After 정책을 구조화 로그로 남긴다.

요청 처리량, active 스레드, 큐 길이, Hikari pending과 downstream timeout을 같은 시간축에서 본다.

---

# 대응과 복구

잘못된 Target Group이면 Listener Rule을 검증된 설정으로 되돌린다.

health check 실패이면 포트, 경로, 성공 코드, 보안 그룹과 애플리케이션 준비 상태를 일치시킨다.

과부하이면 부하를 제한하고 검증된 방식으로 용량을 늘리며 의존 서비스의 한도를 함께 확인한다.

복구 뒤 모든 AZ의 HealthyHostCount와 실제 사용자 요청 성공률을 확인한다.

---

# 재발 방지

HealthyHostCount가 최소 안전 용량 아래로 내려갈 때 경보를 발생시킨다.

배포 정책에 minimum healthy percent와 단계적 교체를 적용해 정상 Target을 보존한다.

health endpoint의 계약과 의존성 포함 기준을 문서화하고 배포 전에 검증한다.

Listener와 Target Group 변경은 코드 리뷰와 자동 검증을 거치게 한다.

---

# 기억해야 할 내용

- ALB 503은 사용할 수 있는 Target이 없을 때 발생할 수 있다.
- 등록 상태, health 상태와 Listener Rule을 함께 확인한다.
- `target_status_code`가 있으면 애플리케이션 503 가능성을 조사한다.
- readiness 설계가 모든 Target을 동시에 제외하지 않게 해야 한다.
- 복구 중에도 최소 정상 용량을 유지한다.

---

# 다음 Chapter

다음 장에서는 [Compute 장애](/aws-backend/part-15/04-compute-failure/)를 분석한다.
