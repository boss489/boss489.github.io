---
title: "Chapter 10. Disaster Recovery"
permalink: /aws-backend/part-16/10-disaster-recovery/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Disaster Recovery
## Region 장애를 전제로 한 복구 목표와 훈련

> **학습 목표**
>
> - Backup/Restore부터 Multi-Site까지 DR 전략을 비교한다.
> - 업무별 RTO와 RPO로 복구 방식을 선택한다.
> - Region 간 복제의 지연과 충돌 위험을 설명한다.
> - 정기 Restore와 Failover 훈련으로 목표를 실측한다.

---

# 요구사항과 실패 시나리오

Multi-AZ는 한 Region 안의 AZ 장애 가용성 설계이며 Region 재해 복구 전략을 대신하지 않는다.

Backup/Restore는 비용이 낮지만 인프라 생성과 데이터 복원 때문에 RTO가 길다.

Pilot Light는 핵심 데이터와 최소 구성만 대기 Region에 유지하고 재해 때 컴퓨팅을 확장한다.

Warm Standby는 축소된 전체 스택을 계속 실행하여 Pilot Light보다 빠르게 전환한다.

Multi-Site Active/Active는 빠른 전환 잠재력이 있지만 데이터 쓰기 충돌과 운영 복잡도와 비용이 가장 크다.

피크 시간에 하위 시스템 하나가 느려졌을 때 전체 주문 경로가 함께 멈추는 구조인지 먼저 질문한다.

네트워크 재시도와 중복 메시지와 AZ 장애를 정상적인 운영 조건으로 간주하고 설계를 검증한다.

---

# 설계 원칙

요구사항을 측정 가능한 지표로 바꾸고 제약과 가정을 명시한다.

동기 경로는 사용자 응답에 꼭 필요한 작업으로 줄이고 나머지는 비동기로 분리한다.

상태는 명시적인 원본 저장소에 두고 캐시와 실행 노드는 교체 가능하게 만든다.

실패를 숨기기보다 Timeout과 격리와 재처리와 관측 지점을 설계한다.

가장 복잡한 기술보다 현재 규모에서 운영할 수 있고 진화 가능한 결정을 선택한다.

---

# 아키텍처

```text
Primary Region
 ECS --> Aurora primary --> async global replica
  |             |
  +--> S3 -- CRR -------------------+
  +--> backups in separate account   |
                                      v
DR Region: IaC + reduced/full stack + data
Route 53 failover after promotion decision
Runbook: freeze writes -> promote -> verify -> shift
Failback is a separate planned migration
```

다이어그램의 화살표는 실제 데이터 경로를 나타내며 관측 도구는 업무 저장소 뒤에 직렬로 놓인 구성 요소가 아니다.

---

# 선택지 비교

| 기준 | 선택 A | 선택 B | 결정 기준 |
|---|---|---|---|
| 전략 | 상대 비용 | 일반적 RTO | 복잡도 |
| Backup/Restore | 낮음 | 김 | 낮음 |
| Pilot Light | 낮음~중간 | 중간 | 중간 |
| Warm Standby | 중간~높음 | 짧음 | 높음 |
| Multi-Site | 높음 | 매우 짧게 목표 가능 | 매우 높음 |

비교표의 결정은 영구 결론이 아니라 현재 요구와 팀의 운영 역량에 기반한 선택이다.

---

# 동작 흐름과 결정

1. Multi-AZ는 한 Region 안의 AZ 장애 가용성 설계이며 Region 재해 복구 전략을 대신하지 않는다.

2. Backup/Restore는 비용이 낮지만 인프라 생성과 데이터 복원 때문에 RTO가 길다.

3. Pilot Light는 핵심 데이터와 최소 구성만 대기 Region에 유지하고 재해 때 컴퓨팅을 확장한다.

4. Warm Standby는 축소된 전체 스택을 계속 실행하여 Pilot Light보다 빠르게 전환한다.

5. Multi-Site Active/Active는 빠른 전환 잠재력이 있지만 데이터 쓰기 충돌과 운영 복잡도와 비용이 가장 크다.

6. 주문과 결제의 RPO는 이미지 썸네일과 로그보다 엄격하게 설정할 수 있다.

7. Aurora Global Database와 S3 Cross-Region Replication은 비동기 복제 지연과 미복제 창을 고려한다.

8. ElastiCache 데이터는 원본으로 간주하지 않고 Aurora 또는 이벤트에서 재구축 가능하게 한다.

9. Route 53 전환만으로 끝나지 않으며 Secret과 인증서와 Image와 Queue와 외부 PG 허용 목록을 준비한다.

10. 복원 훈련은 백업 존재 확인이 아니라 격리 환경에서 데이터 무결성과 애플리케이션 기동까지 검증한다.

결정 기록에는 선택하지 않은 대안과 다시 검토할 조건도 함께 남긴다.

---

# 구현 예

```yaml
dr:
  assumptions: "아래 목표는 예시이며 사업 영향 분석으로 확정"
  orders:
    strategy: warm-standby
    rpo: "복제 지연을 포함해 훈련으로 측정"
    rto: "승격과 검증과 DNS 전환을 포함"
  media:
    strategy: backup-restore
  drill:
    restore: quarterly
    regional-failover: semiannual
```

예제 설정과 수치는 동작 원리를 보여 주기 위한 가정이며 운영값은 측정과 부하 테스트로 확정한다.

---

# 실무 Trade-off

가용성을 높이는 복제와 대기 자원은 비용을 늘리므로 업무 중요도별로 등급을 나눈다.

강한 정합성은 안전한 결정을 돕지만 지연과 결합도를 높일 수 있어 필요한 경계에만 적용한다.

관리형 서비스는 운영 부담을 줄이지만 서비스 한도와 종속성과 비용 모델을 이해해야 한다.

비동기화는 장애 격리와 흡수력을 높이지만 상태 추적과 중복 처리와 재처리 운영이 필요하다.

초기 설계는 단순하게 시작하되 지표와 인터페이스를 남겨 병목이 확인되면 교체할 수 있게 한다.

---

# 장애, 보안과 비용

장애 시나리오는 증상, 탐지 지표, 자동 완화, 수동 Runbook과 복구 확인 순서로 작성한다.

IAM Role에는 필요한 Action과 Resource만 허용하고 장기 Access Key를 애플리케이션에 저장하지 않는다.

전송 구간 TLS와 저장 데이터 KMS 암호화를 적용하고 개인정보 접근은 감사 가능하게 남긴다.

재시도는 지수 Backoff와 Jitter와 횟수 제한을 사용하여 장애 중 부하 증폭을 막는다.

비용은 컴퓨팅 시간, 저장량, 요청 수, 데이터 전송과 고정 네트워크 비용으로 나눠 추적한다.

특정 달러 가격은 Region과 시점과 약정에 따라 달라지므로 확정값 대신 AWS Pricing Calculator와 실제 청구로 검증한다.

---

# 검증 질문

- 이 결정이 해결하려는 구체적인 요구사항과 제약은 무엇인가.
- 한 AZ 또는 외부 의존성이 실패하면 사용자가 어떤 응답을 받는가.
- 중복 요청과 지연된 이벤트와 부분 성공을 어떻게 식별하고 복구하는가.
- 가장 먼저 포화되는 자원과 이를 알려 주는 선행 지표는 무엇인가.
- 보안 경계와 데이터 소유자와 최소 권한의 증거는 무엇인가.
- 예상 비용의 가장 큰 항목과 트래픽 증가에 따른 기울기는 무엇인가.
- 설계 가정을 어떤 부하 테스트와 장애 훈련으로 반증할 수 있는가.

---

# 기억해야 할 내용

- Backup/Restore부터 Multi-Site까지 DR 전략을 비교한다.
- 업무별 RTO와 RPO로 복구 방식을 선택한다.
- Region 간 복제의 지연과 충돌 위험을 설명한다.
- 정기 Restore와 Failover 훈련으로 목표를 실측한다.
- 요구사항에서 제약과 대안을 거쳐 결정을 내리고 지표와 훈련으로 검증한다.
- 실패와 비용과 운영 책임이 없는 아키텍처 다이어그램은 완성된 설계가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 [Architecture Review](/aws-backend/part-16/11-architecture-review/)를 학습한다.

앞 장의 결정을 다음 장의 더 구체적인 경계와 운영 절차로 연결한다.
