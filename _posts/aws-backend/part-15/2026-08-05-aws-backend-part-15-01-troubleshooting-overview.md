---
title: "Chapter 01. Troubleshooting Overview"
permalink: /aws-backend/part-15/01-troubleshooting-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Troubleshooting Overview
## 증거로 가설을 좁히는 장애 분석

> **학습 목표**
>
> - 장애 대응의 공통 순서를 설명할 수 있다.
> - 영향 범위와 최근 변경을 먼저 확인할 수 있다.
> - 메트릭, 로그, 추적 데이터를 연결해 가설을 검증할 수 있다.
> - 안전한 완화와 근본 복구를 구분할 수 있다.

---

# 실제 장애 징후

사용자는 느린 응답, 오류 코드, 연결 실패 또는 데이터 불일치로 장애를 먼저 경험한다.

운영자는 오류율, 지연 시간, 트래픽, 포화도와 가용성 경보로 이상을 감지한다.

한 지표의 이상은 원인이 아니라 조사 시작점이므로 다른 계층의 증거와 교차 검증해야 한다.

---

# Troubleshooting이란?

Troubleshooting은 관찰 가능한 증거로 가능한 원인을 줄이고 서비스를 안전하게 복구하는 과정이다.

좋은 대응은 가장 익숙한 서버를 재시작하는 것이 아니라 사용자 영향과 변경 이력을 먼저 확인한다.

진단 중에는 시각, 명령, 결과, 판단을 타임라인에 남겨 다음 담당자가 같은 조사를 반복하지 않게 한다.

---

# 계층 구조

```text
User symptom
     |
DNS and edge
     |
ALB and target group
     |
Network path
     |
EC2 / ECS / EKS
     |
Spring Boot
     |
Redis / RDS
```

상위 계층의 증상은 하위 계층의 원인으로 발생할 수 있으므로 요청 경로를 따라 경계를 확인한다.

---

# 증거 기반 진단 순서

## 1. 영향을 정의한다

오류가 모든 사용자에게 발생하는지 특정 리전, 기능, 테넌트 또는 버전에만 발생하는지 확인한다.

오류율과 지연 시간의 시작 시각을 기록하고 핵심 사용자 여정의 성공 여부를 확인한다.

## 2. 최근 변경을 확인한다

애플리케이션 배포, 인프라 변경, 설정 변경, 인증서 갱신과 데이터 마이그레이션 시각을 비교한다.

변경과 장애의 시간적 상관관계는 가설이지 원인 확정이 아니므로 관측 자료로 검증한다.

## 3. 메트릭, 로그, 추적을 연결한다

메트릭으로 이상이 시작된 계층과 시간을 찾고 로그로 오류 맥락을 확인한다.

분산 추적의 trace ID와 애플리케이션 correlation ID로 느리거나 실패한 구간을 찾는다.

## 4. 검증 가능한 가설을 세운다

가설은 예상 증거와 반증 조건을 포함해 한 번에 하나씩 검증한다.

예를 들어 커넥션 풀 고갈 가설은 active가 max에 붙고 pending과 timeout이 함께 증가할 것을 예측한다.

## 5. 안전하게 완화한다

트래픽 우회, 문제 버전 롤백, 기능 플래그 비활성화 또는 부하 제한처럼 되돌릴 수 있는 조치를 우선한다.

완화 전후의 성공률과 지연 시간을 비교하고 부작용이 커지면 즉시 되돌린다.

---

# 명령과 관측 도구

```bash
aws cloudwatch describe-alarms --state-value ALARM
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateService
```

CloudTrail은 변경 주체와 API 호출을 확인하는 자료이며 애플리케이션 요청 로그를 대신하지 않는다.

CloudWatch Logs Insights에서는 공통 시간 범위와 요청 식별자를 사용한다.

```sql
fields @timestamp, @logStream, level, message, traceId
| filter level = "ERROR"
| sort @timestamp desc
| limit 100
```

조사 시간 범위는 장애 직전의 정상 구간을 포함해 기준선과 비교한다.

---

# Spring Boot 관찰 포인트

Actuator의 `health`, `metrics`, `prometheus` 엔드포인트는 인증과 네트워크 제한 아래 운영한다.

HTTP 서버 요청 수, 상태 코드, p95 지연 시간, JVM heap, GC pause, 스레드와 커넥션 풀을 함께 본다.

구조화 로그에는 timestamp, level, service, version, traceId, requestId와 오류 유형을 포함한다.

민감 정보와 인증 토큰은 로그에 남기지 않는다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

---

# 대응과 복구

완화는 사용자 영향을 줄이는 단기 조치이고 복구는 정상 상태를 안정적으로 유지하는 단계다.

복구 후에는 합성 요청, 핵심 API, 비동기 처리와 데이터 정합성을 확인한다.

모니터링이 정상으로 돌아와도 누적된 큐와 재시도 트래픽이 다시 포화를 만들 수 있어 추세를 관찰한다.

---

# 재발 방지

알람은 원인이 아니라 사용자 증상과 서비스 수준 목표를 중심으로 설계한다.

런북에는 확인 명령, 기대 결과, 안전한 완화, 롤백 조건과 담당 팀을 기록한다.

배포 표시와 설정 변경 이벤트를 대시보드에 함께 나타내면 상관관계를 빠르게 찾을 수 있다.

게임 데이와 복구 훈련으로 문서가 실제 환경에서 실행 가능한지 검증한다.

---

# 기억해야 할 내용

- 영향 범위를 정의한 뒤 최근 변경을 확인한다.
- 메트릭으로 위치를 찾고 로그와 추적으로 맥락을 검증한다.
- 가설에는 예상 증거와 반증 조건이 있어야 한다.
- 재시작은 원인을 지우고 영향을 넓힐 수 있으므로 첫 단계가 아니다.
- 완화는 되돌릴 수 있고 관찰 가능한 방식으로 수행한다.
- 복구 후에는 데이터와 누적 작업까지 검증한다.

---

# 다음 Chapter

다음 장에서는 [ALB 502 장애](/aws-backend/part-15/02-alb-502/)를 증거 기반으로 구분한다.
