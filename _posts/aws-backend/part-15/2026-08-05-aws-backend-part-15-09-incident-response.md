---
title: "Chapter 09. Incident Response"
permalink: /aws-backend/part-15/09-incident-response/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Incident Response
## 사용자 영향을 줄이는 협업 체계

> **학습 목표**
>
> - severity와 역할을 명확히 선언할 수 있다.
> - mitigation-first 원칙으로 대응할 수 있다.
> - timeline과 communication을 운영할 수 있다.
> - blameless postmortem과 후속 조치를 작성할 수 있다.

---

# 실제 장애 징후

기술 경보가 여러 개 발생해도 하나의 사용자 영향에서 파생된 사건일 수 있다.

담당자가 동시에 같은 변경을 수행하거나 서로 다른 가설을 공유하지 않으면 복구가 늦어진다.

상태 공지가 없으면 고객 지원과 경영진이 대응팀에 반복 질의해 조사 집중도가 떨어진다.

완화 후 backlog와 데이터 손상이 남아 있으면 서비스 지표가 정상이어도 incident는 끝나지 않았다.

---

# Incident Response란?

Incident Response는 서비스 장애를 감지하고 역할을 정해 영향을 완화하며 복구와 학습까지 수행하는 체계다.

severity는 기술 오류 이름이 아니라 사용자 영향, 범위, 지속 시간, 데이터와 보안 위험을 기준으로 정한다.

Incident Commander는 전체 우선순위, 역할, 의사결정과 다음 업데이트 시각을 관리한다.

Operations 담당은 완화와 복구 작업을 수행하고 Communication 담당은 이해관계자에게 검증된 사실을 전달한다.

Scribe는 시각, 관찰, 가설, 명령, 결정과 결과를 timeline에 기록한다.

작은 팀에서는 한 사람이 여러 역할을 맡을 수 있지만 최종 의사결정자는 명확해야 한다.

mitigation-first는 완전한 원인 규명보다 사용자 영향을 안전하게 줄이는 조치를 우선한다는 뜻이다.

---

# 대응 구조

```text
Detection
   |
Severity and scope
   |
Incident Commander
   +-- Operations
   +-- Communication
   +-- Scribe
   |
Safe mitigation
   |
Recovery validation
   |
Blameless postmortem
```

역할 분리는 명령 계층을 늘리는 것이 아니라 중복 작업과 정보 손실을 줄이기 위한 장치다.

---

# 증거 기반 대응 순서

## 1. 영향과 severity를 선언한다

영향받는 사용자, 기능, 지역, 오류율, 시작 시각과 데이터 위험을 한 문장으로 정리한다.

severity 기준표에 따라 incident 수준과 호출할 담당 팀을 정한다.

초기 정보가 불완전하면 추정임을 표시하고 다음 평가 시각을 정한다.

## 2. 역할과 채널을 연다

Incident Commander, Operations, Communication과 Scribe를 지정한다.

하나의 대화 채널, 문서와 화상 회의를 source of truth로 선언한다.

승인되지 않은 병렬 변경을 막고 각 작업에 수행자와 예상 결과를 기록한다.

## 3. 최근 변경과 증거를 모은다

배포, 인프라, 설정, 데이터 작업과 외부 공급자 event를 timeline에 추가한다.

metrics, logs, traces와 실제 사용자 요청을 같은 시간 범위에서 비교한다.

```bash
aws cloudwatch describe-alarms --state-value ALARM
aws cloudtrail lookup-events \
  --start-time "$START_TIME" \
  --end-time "$END_TIME"
```

명령 출력과 dashboard link는 민감 정보를 제거한 뒤 incident 문서에 연결한다.

## 4. 가설을 관리한다

각 가설에 근거, 예상 증거, 반증 조건, 담당자와 상태를 기록한다.

검증되지 않은 추측을 외부 공지에 확정 원인처럼 쓰지 않는다.

## 5. 안전하게 완화한다

롤백, traffic 우회, 기능 제한, rate limit과 dependency 격리 중 가장 되돌리기 쉬운 조치를 선택한다.

조치 전에 기대 효과, 위험, rollback 조건과 관찰 지표를 합의한다.

여러 변경을 동시에 하면 어느 조치가 효과를 냈는지 알기 어려우므로 긴급성이 허용하는 범위에서 분리한다.

## 6. communication을 지속한다

상태 공지는 영향, 현재 조치, 알려진 범위와 다음 업데이트 시각을 포함한다.

원인이 불명확하면 불명확하다고 말하고 검증되지 않은 복구 시각을 약속하지 않는다.

내부 기술 기록과 고객 대상 공지는 독자와 민감 정보 수준을 구분한다.

## 7. 복구를 검증한다

오류율과 지연뿐 아니라 핵심 사용자 여정, 비동기 backlog와 데이터 정합성을 확인한다.

일정 관찰 구간 동안 재발하지 않고 정상 용량이 유지되는지 확인한다.

Incident Commander가 종료 조건을 검토한 뒤 monitoring 상태 또는 종료를 선언한다.

---

# Spring Boot 관찰 포인트

배포 version, traceId, requestId와 instance ID를 구조화 로그에 넣어 timeline 연결을 돕는다.

Actuator와 Micrometer로 HTTP error, latency, JVM, executor, Hikari와 dependency 지표를 제공한다.

business 지표를 함께 보면 HTTP 성공처럼 보이지만 주문이나 결제가 실패하는 상황을 찾을 수 있다.

기능 플래그와 rate limit은 권한, 감사 기록, rollback 절차를 갖춰야 안전한 완화 수단이 된다.

---

# RTO와 RPO

RTO는 장애 후 서비스를 목표 시간 안에 복구하려는 기준이다.

RPO는 복구 시 허용 가능한 데이터 손실 시점을 나타내는 기준이다.

incident 중 선택하는 restore 시점, failover 방식과 기능 제한은 RTO와 RPO의 영향을 받는다.

두 목표는 문서에 적는 것만으로 충족되지 않으므로 실제 복구 훈련으로 측정한다.

---

# 대응과 복구

완화가 성공하면 긴급 변경을 정식 구성과 코드에 반영하거나 안전하게 원복한다.

backlog, 실패 이벤트, 중복 처리와 고객 보상 대상을 확인한다.

교대 시 현재 영향, 완료 작업, 열린 가설, 위험과 다음 결정을 구조화해 전달한다.

---

# Blameless Postmortem

blameless는 책임을 없애는 말이 아니라 개인 비난 대신 시스템 조건과 의사결정을 학습한다는 원칙이다.

문서에는 영향, 감지, timeline, 기여 요인, 잘된 점, 개선점과 복구 검증을 포함한다.

근본 원인을 하나의 개인 실수로 축약하지 않고 방어 계층이 왜 막지 못했는지 분석한다.

후속 action에는 구체적 결과물, owner, deadline과 검증 방법을 반드시 둔다.

action은 문서 작성 수보다 감지 시간, 완화 시간과 재발 가능성을 실제로 줄이는지 평가한다.

---

# 재발 방지

severity 기준, 연락망, 역할 카드와 상태 공지 template을 정기적으로 검토한다.

자주 발생하는 장애는 실행 가능한 runbook과 자동 진단으로 전환한다.

게임 데이에서 RTO와 RPO, 백업 복구, failover와 communication 흐름을 훈련한다.

postmortem action은 일반 backlog에서 잊히지 않게 별도 추적하고 기한 초과를 가시화한다.

---

# 기억해야 할 내용

- severity는 사용자 영향과 데이터 위험으로 정한다.
- Incident Commander가 우선순위와 의사결정을 조정한다.
- mitigation-first는 원인 분석을 포기하는 것이 아니다.
- timeline과 정기 communication이 협업 비용을 줄인다.
- 종료 전 backlog와 데이터 정합성까지 검증한다.
- postmortem action에는 owner와 deadline이 필요하다.
- RTO와 RPO는 복구 훈련으로 검증한다.

---

# 다음 Chapter

다음 장에서는 [Part 15 Summary](/aws-backend/part-15/10-summary/)로 핵심 내용을 정리한다.
