---
title: "Chapter 09. Summary"
permalink: /aws-backend/part-11/09-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 09. Summary
## 운영 신호를 하나의 대응 체계로 연결한다
> **학습 목표**
>
> - Part 11의 핵심 개념을 한눈에 비교한다.
> - 쇼핑몰 운영의 최소 관측 구성을 점검한다.
> - 장애 탐지부터 복구 검증까지의 순서를 정리한다.
---
# 핵심 개념 표
| 영역 | 핵심 질문 | 주요 구성 | 주의점 |
|---|---|---|---|
| Monitoring | 지금 고객 영향이 있는가 | SLI, SLO, 로그·메트릭·트레이스 | 신호 수집 자체를 목표로 삼지 않는다. |
| CloudWatch Logs | 어떤 사건이 발생했는가 | Group, Stream, retention, Insights | 민감정보를 기록하지 않는다. |
| CloudWatch Metrics | 상태가 시간에 따라 어떻게 변하는가 | Namespace, Dimension, Period, percentile | 고카디널리티 차원을 피한다. |
| Alarm | 지금 사람이 행동해야 하는가 | M out of N, missing data, SNS | 알람 피로와 오탐을 관리한다. |
| Dashboard | 고객 영향과 원인이 어디에 있는가 | RED, USE, 비즈니스 지표 | runbook과 상세 화면을 연결한다. |
| CloudTrail | 누가 AWS 구성을 변경했는가 | Management/Data Event, Trail, S3 | 조직 수집과 무결성을 보장한다. |
| X-Ray | 어느 호출 구간이 느리거나 실패했는가 | Trace, Segment, Sampling | 비동기 context를 전파한다. |
| Spring Boot | 세 신호를 어떻게 생성하고 연결하는가 | Actuator, Micrometer, MDC, OTel | endpoint 보호와 중복 수집을 막는다. |
---
# 신호 선택 기준
| 상황 | 먼저 볼 신호 | 다음 신호 |
|---|---|---|
| 주문 오류율 증가 | Metrics와 Alarm | Logs와 Trace |
| 특정 주문 문의 | JSON Logs | 같은 traceId의 Trace |
| 서비스 간 지연 | Trace | 구간별 Metrics와 Logs |
| AWS 설정 변경 | CloudTrail | 배포 기록과 애플리케이션 Metrics |
| 전환율 하락 | 비즈니스 Dashboard | RED, USE, Logs |
| 수집 누락 | Telemetry 자체 Metrics | Agent와 Collector Logs |
---
# 최소 운영 아키텍처
```
고객
  │
  ▼
ALB ──▶ Spring Boot ──▶ Redis / Aurora / 외부 결제
          │
          ├─ JSON Logs + correlationId ──▶ CloudWatch Logs
          ├─ Micrometer Metrics ─────────▶ CloudWatch / AMP
          └─ OpenTelemetry Trace ────────▶ ADOT / X-Ray
                                             │
CloudTrail ──▶ 중앙 S3       Alarm ──▶ SNS ──▶ On-call
                                             │
                                   Dashboard / Runbook
```
---
# 운영 체크리스트
## 로그
- [ ] 환경과 서비스별 Log Group 이름 규칙이 있다.
- [ ] Log Stream 생성 주체를 설명할 수 있다.
- [ ] 보존 기간이 명시되어 있다.
- [ ] JSON 필드 이름이 서비스 간 표준화되어 있다.
- [ ] correlationId와 traceId가 기록된다.
- [ ] 비밀번호, 토큰, 카드 정보가 기록되지 않는다.
- [ ] Logs Insights 저장 쿼리가 준비되어 있다.
- [ ] Metric Filter와 Subscription Filter 목적이 구분되어 있다.
## 메트릭
- [ ] Namespace와 메트릭 이름 규칙이 있다.
- [ ] Dimension 허용 목록과 카디널리티 예산이 있다.
- [ ] Count, Average, percentile을 의미에 맞게 사용한다.
- [ ] 서비스 RED와 자원 USE 지표가 있다.
- [ ] 주문 성공률 같은 비즈니스 지표가 있다.
- [ ] 데이터 없음과 0의 의미가 정의되어 있다.
- [ ] EMF 또는 Registry의 중복 발행이 없다.
## 알람과 대시보드
- [ ] M out of N과 Missing Data 정책을 검토했다.
- [ ] 고객 영향 기반의 페이지 알람이 있다.
- [ ] Composite Alarm으로 중복 알림을 줄였다.
- [ ] SNS 구독 전달을 실제로 시험했다.
- [ ] 모든 알람에 소유자와 runbook이 있다.
- [ ] 대시보드 첫 화면에서 영향 범위를 판단할 수 있다.
- [ ] 배포와 구성 변경 시각을 비교할 수 있다.
- [ ] 정기적으로 오탐과 무응답 알람을 정리한다.
## 감사와 추적
- [ ] 모든 Region의 조직 Trail이 활성화되어 있다.
- [ ] 감사 로그가 별도 보안 계정 S3에 저장된다.
- [ ] 중요한 Data Event 범위를 선택했다.
- [ ] 로그 파일 검증 절차가 runbook에 있다.
- [ ] HTTP와 메시지 경계에서 trace context를 전파한다.
- [ ] 비동기 Executor에서 context가 유지된다.
- [ ] Sampling 정책과 예외 조사 절차가 있다.
## Spring Boot
- [ ] Actuator endpoint를 최소 공개하고 보호한다.
- [ ] liveness와 readiness를 구분한다.
- [ ] MeterRegistry를 생성자 주입한다.
- [ ] MDC를 요청 종료 시 정리한다.
- [ ] `/actuator/prometheus`를 수집할 안전한 collector 경로가 있다.
- [ ] CloudWatch Registry와 Prometheus 경로의 목적이 명확하다.
- [ ] OpenTelemetry Agent의 service, environment, version이 설정되어 있다.
- [ ] exporter 실패가 업무 요청을 막지 않는다.
---
# 장애 대응 순서
1. 고객 영향과 시작 시각을 확인한다.
2. 핵심 SLI와 알람 상태를 확인한다.
3. 최근 배포와 CloudTrail 변경을 확인한다.
4. RED 지표로 실패 서비스와 구간을 좁힌다.
5. USE 지표로 자원 포화 여부를 확인한다.
6. correlationId로 구조화 로그를 검색한다.
7. traceId로 분산 호출 병목을 확인한다.
8. 안전한 완화 조치를 실행한다.
9. 동일 지표가 정상 범위로 회복했는지 검증한다.
10. 타임라인과 재발 방지 항목을 기록한다.
---
# 기억할 내용
- 로그는 문맥, 메트릭은 추세, 트레이스는 경로를 설명한다.
- 알람은 관찰이 아니라 행동을 시작시키는 신호이다.
- CloudTrail은 애플리케이션 로그를 대체하지 않으며 제어 영역 변경을 기록한다.
- 고카디널리티 값은 로그나 트레이스에서 찾고 메트릭 Dimension으로 만들지 않는다.
- 비용은 수집량, 시계열 수, 보존, 조회, 알람, 샘플링의 영향을 받는다.
- 정상 기준선과 runbook이 없는 그래프는 장애 대응 도구가 되기 어렵다.
---
# 다음 Chapter
다음 Chapter에서는 [Interview](/aws-backend/part-11/10-interview/) 질문으로 개념을 점검한다.
