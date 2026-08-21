---
title: "Chapter 09. Part 14 Summary"
permalink: /aws-backend/part-14/09-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Part 14 Summary
## Cost Optimization 핵심 정리

Part 14의 핵심은 비용을 단순히 낮추는 것이 아니라 비즈니스 가치, 가용성, 성능과 운영 가능성을 함께 최적화하는 것이다.

가격은 Region, 시점과 구매 모델에 따라 바뀌므로 특정 가격을 암기하지 않고 최신 공식 정보를 확인한다.

---

# 전체 비용 흐름

```
사용자
  -> CloudFront / S3
  -> ALB / NAT / Network
  -> EC2 / ECS / Lambda
  -> EBS / RDS / Aurora
  -> Billing data
  -> detect -> allocate -> optimize -> govern
  -> cost/order + SLO
```

---

# 가시성

| 도구 | 역할 |
|---|---|
| Cost Explorer | 비용과 사용량 추세 탐색 |
| Budgets | 계획 임계치와 예측 알림 |
| Cost Anomaly Detection | 평소와 다른 비용 변화 탐지 |
| CUR·Data Exports | 상세 청구 데이터 내보내기와 분석 |
| Tag·Cost Categories | 팀과 제품으로 비용 할당 |

계정 분리는 권한 경계와 비용 경계를 명확하게 만들며 공통 비용에는 합의된 배부 규칙이 필요하다.

---

# 컴퓨팅

Right Sizing은 평균 CPU 하나가 아니라 피크, 메모리, GC, 네트워크, 지연과 오류율을 함께 보고 결정한다.

예측 가능한 비운영 시간에는 스케줄을 사용하고 변동 수요에는 Auto Scaling을 적용한다.

Graviton은 JDK, 컨테이너 이미지와 네이티브 의존성 호환성을 검증한 뒤 도입한다.

Spot은 중단 신호, 작업 재시도, 체크포인트와 정상 종료가 준비된 워크로드에 적용한다.

Fargate, EC2와 Lambda는 실행 패턴, 상태성, 시작 시간, 제약과 팀 운영 역량으로 비교한다.

---

# 네트워크

NAT Gateway는 실행 시간과 처리 데이터가 주요 비용 요소다.

AZ 간 전송 경로와 Public IPv4 사용 필요성을 아키텍처 다이어그램에 표시한다.

S3 트래픽에는 S3 Gateway Endpoint를 우선 검토한다.

Interface Endpoint는 사설 경로의 장점과 endpoint 고정비 및 처리 데이터 비용을 NAT 경로와 비교한다.

ALB 비용은 요청 수뿐 아니라 새 연결, 활성 연결, 처리 바이트와 규칙 평가를 반영하는 LCU 관점으로 본다.

---

# 스토리지와 데이터베이스

미연결 EBS, 오래된 Snapshot과 불필요한 S3 버전을 정기적으로 찾아 owner와 복구 요구를 확인한다.

S3 Lifecycle은 접근 빈도, 최소 보관 조건, 전환과 복원 요구를 근거로 설계한다.

RDS는 인스턴스, 스토리지, I/O, 백업, 데이터 전송과 Read Replica 비용을 분리해 본다.

Read Replica는 실제 읽기 분산 효과가 있을 때 유지한다.

Aurora I/O 모델은 워크로드와 선택 옵션에 따라 결과가 달라질 수 있으므로 과도하게 일반화하지 않는다.

---

# CloudFront와 S3

Cache Hit Ratio가 높아지면 원본 S3 요청과 원본 데이터 흐름을 줄일 수 있다.

TTL은 변경 반영 요구와 캐시 재사용 사이에서 결정한다.

캐시 키에는 응답을 실제로 바꾸는 Header, Cookie와 Query String만 포함한다.

텍스트 압축은 전송량을 줄이지만 CPU와 지연을 함께 측정한다.

정적 자산은 버전 URL과 긴 TTL을 사용하고 반복적인 전체 Invalidation을 피한다.

---

# 할인 약정

Savings Plans와 Reserved Instances는 약정 범위와 변경 유연성이 서로 다르다.

할인 적용과 특정 AZ의 용량 예약은 같은 개념이 아니다.

Right Sizing 이후 안정적인 온디맨드 기준선을 측정하고 확실한 일부 사용량부터 약정한다.

utilization과 coverage를 함께 검토해 과약정과 미할인 사용량을 구분한다.

---

# FinOps 운영 루프

```
detect -> allocate -> optimize -> govern
   ^                              |
   +------------------------------+
```

모든 개선 항목에는 owner, 기한, 예상 효과, 되돌리기와 검증 지표가 있어야 한다.

총비용뿐 아니라 `cost/order` 같은 Unit Economics로 성장과 비효율을 구분한다.

주간 review는 이상과 즉시 조치를 다루고 월간 review는 추세, 약정과 구조 개선을 다룬다.

---

# 다음 장

→ **[Chapter 10. Interview Questions](/aws-backend/part-14/10-interview/)**
