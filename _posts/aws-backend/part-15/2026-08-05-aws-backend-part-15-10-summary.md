---
title: "Chapter 10. Part 15 Summary"
permalink: /aws-backend/part-15/10-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Part 15 Summary
## Troubleshooting 핵심 정리

---

# 공통 진단 순서

장애 대응은 재시작이 아니라 사용자 영향과 범위를 정의하는 것에서 시작한다.

최근 배포, 인프라, 설정과 데이터 변경을 장애 시작 시각과 비교한다.

메트릭으로 이상 계층과 시간을 찾고 로그와 trace로 요청 맥락을 확인한다.

가설에는 예상 증거와 반증 조건을 두고 한 번에 하나씩 검증한다.

안전한 완화는 되돌릴 수 있어야 하며 전후 지표로 효과를 확인해야 한다.

복구 후에는 핵심 사용자 여정, backlog와 데이터 정합성까지 확인한다.

```text
Impact
  |
Recent changes
  |
Metrics / Logs / Traces
  |
Hypothesis
  |
Safe mitigation
  |
Recovery validation
```

---

# 계층별 핵심 증거

| 계층 | 대표 증거 | 핵심 질문 |
|---|---|---|
| ALB 502 | Access Log 상태 코드와 error reason | ALB가 Target 응답을 받았는가 |
| ALB 503 | Target 등록과 health | 사용할 수 있는 Target이 있는가 |
| Compute | OS 자원, 종료 이유, event | 어느 실행 단위와 자원이 실패했는가 |
| RDS | Hikari와 RDS wait 및 지표 | pool, query, lock 또는 storage 문제인가 |
| Redis | memory, eviction, shard와 slowlog | cache 장애가 원본을 압박하는가 |
| DNS | authoritative 응답, TTL과 trace | 원본, 위임 또는 cache 문제인가 |
| Network | SG, NACL, route와 flow | packet과 응답이 양방향으로 이동하는가 |

---

# ALB

ALB 502는 Target과 정상 HTTP 교환을 완료하지 못한 경우와 Target이 직접 502를 반환한 경우를 구분한다.

`elb_status_code`와 `target_status_code`는 ALB와 Target 사이의 실패 경계를 찾는 핵심 필드다.

연결 reset, 거부, malformed response, idle timeout 불일치와 TLS를 각각 증거로 검증한다.

ALB 503은 usable Target 부재와 Target이 과부하로 직접 반환한 503을 구분한다.

Target Group 등록, health reason, Listener Rule과 readiness를 함께 확인한다.

---

# Compute

EC2, ECS와 EKS는 CPU, memory, disk, inode, file descriptor, socket과 OOM이라는 공통 자원을 사용한다.

EC2 status와 journal, ECS stopped reason, EKS event 및 이전 컨테이너 로그를 확인한다.

문제 대상을 격리하기 전에 남은 Target이 트래픽을 감당할 수 있는지 확인한다.

---

# RDS와 Redis

RDS는 connection exhaustion, lock, deadlock, slow query, failover, storage와 replica lag를 나누어 본다.

Hikari active, pending과 max를 DBConnections와 같은 시간축에서 비교한다.

무분별한 retry는 연결 폭주와 중복 쓰기를 만들 수 있다.

Redis는 cache인지 원본인지 먼저 구분해야 복구와 데이터 검증 범위를 정할 수 있다.

memory, eviction, connection, hot key, slow command, failover와 stampede를 shard별로 확인한다.

cache 장애에서는 RDS 같은 원본 저장소를 보호하는 완화를 우선한다.

---

# DNS와 Network

DNS는 authoritative 원본과 recursive cache 및 JVM cache를 구분한다.

NXDOMAIN, SERVFAIL과 NODATA의 의미가 다르므로 응답 코드를 정확히 기록한다.

`dig +trace`로 위임 경로를 확인하고 Alias와 CNAME을 구분한다.

Security Group은 stateful이고 NACL은 stateless이므로 NACL은 양방향 규칙이 필요하다.

route, IGW, NAT와 ephemeral port를 실제 source와 destination 흐름으로 확인한다.

Reachability Analyzer는 구성 경로를 분석하고 Flow Logs는 관찰된 network 흐름의 단서를 제공한다.

---

# Incident Response

severity는 기술 구성 요소보다 사용자 영향, 범위와 데이터 위험을 기준으로 정한다.

Incident Commander가 우선순위와 의사결정을 조정하고 Operations, Communication과 Scribe가 역할을 나눈다.

mitigation-first는 사용자 영향을 먼저 줄이고 안정화 뒤 근본 원인을 계속 분석하는 방식이다.

timeline에는 관찰, 가설, 조치, 결과와 의사결정 시각을 기록한다.

상태 공지는 확인된 영향, 현재 조치와 다음 업데이트 시각을 제공한다.

blameless postmortem은 개인 비난 대신 시스템의 방어 계층과 의사결정을 개선한다.

모든 후속 action에는 owner, deadline과 검증 방법을 둔다.

RTO와 RPO는 문서가 아니라 실제 복구 훈련으로 검증한다.

---

# 최종 체크리스트

- [ ] 사용자 영향과 시작 시각을 정의했는가.
- [ ] 최근 변경을 확인했는가.
- [ ] 정상 기준선과 장애 지표를 비교했는가.
- [ ] 완화의 위험과 rollback 조건을 확인했는가.

---

# 다음 Chapter

다음 장에서는 [Interview Questions](/aws-backend/part-15/11-interview/)로 장애 대응 판단을 점검한다.
