---
title: "Chapter 04. 네트워크 비용 최적화"
permalink: /aws-backend/part-14/04-network-cost/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. 네트워크 비용 최적화
## 보이지 않는 경로별 전송 비용 찾기

> **학습 목표**
>
> - NAT Gateway의 시간과 처리 데이터 비용 요소를 구분할 수 있다.
> - AZ 간 전송, Public IPv4와 VPC Endpoint의 비용 영향을 설명할 수 있다.
> - ALB의 기본 비용과 LCU 개념을 트래픽 특성에 연결할 수 있다.

---

# 비용 폭증 시나리오

배포 규모는 그대로인데 이미지 처리 작업을 추가한 뒤 네트워크 비용이 급증한 상황을 가정한다.

Private Subnet의 작업이 S3로 대용량 파일을 주고받으며 NAT Gateway를 통과하면 처리 데이터가 커질 수 있다.

서로 다른 AZ의 애플리케이션과 데이터 저장소 사이 통신도 요청량이 같아도 전송량을 늘릴 수 있다.

---

# 핵심 정의

## NAT Gateway 비용

게이트웨이가 존재하는 시간과 통과한 처리 데이터가 주요 요소다.

## AZ 간 전송

가용 영역 경계를 넘는 데이터 흐름에 서비스별 전송 조건이 적용되는 비용 요소다.

## Public IPv4

공인 IPv4 주소 사용 자체가 관리해야 할 비용 항목이다.

## VPC Endpoint

서비스를 사설 경로로 연결하며 유형에 따라 시간 고정비와 처리 데이터 비용이 생길 수 있다.

## LCU

ALB의 새 연결, 활성 연결, 처리 바이트와 규칙 평가 같은 차원을 반영하는 용량 단위다.

---

# 비용 흐름

```
Private Spring Boot
  -> NAT Gateway ── 인터넷·AWS API
  -> Interface Endpoint ─ 대상 서비스
  -> S3 Gateway Endpoint ─ S3
ALB -> AZ-a Target
    -> AZ-b Target ─ AZ 경계
각 경로 -> 시간·주소·처리 데이터·LCU
```

이 흐름에서 사용량이 생성되는 지점과 청구 데이터에서 보이는 지점을 연결해야 한다.

---

# 비교표

| 경로 | 비용 성격 | 선택 기준 |
|---|---|---|
| NAT Gateway | 시간 + 처리 데이터 | 범용 외부 통신 |
| Interface Endpoint | 시간 + 처리 데이터 | 지원 서비스 사설 연결 |
| S3 Gateway Endpoint | 라우팅 기반 | S3 사설 경로 |
| Public IPv4 | 주소 사용 | 공인 주소 필요성 |

비교표는 절대적인 우열이 아니라 워크로드에 맞는 선택 기준을 제공한다.

---

# Cost Explorer와 AWS CLI

다음 명령은 예시 기간의 데이터를 조회하며 실행 계정의 권한과 실제 날짜를 확인해야 한다.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-0123456789abcdef0 \
  --statistics Sum --period 3600 \
  --start-time 2026-08-01T00:00:00Z --end-time 2026-08-02T00:00:00Z
aws ec2 describe-vpc-endpoints
```

CLI 결과는 청구 확정 시점과 데이터 지연을 고려해 해석한다.

특정 가격 수치는 문서에 고정하지 않고 Region, 시점과 구매 모델에 맞는 공식 가격 정보를 확인한다.

---

# Spring Boot 운영자가 통제할 수 있는 요소

외부 API 호출 응답을 필요한 범위에서 캐시하고 불필요한 재시도와 큰 응답 전송을 줄인다.

S3 업로드는 애플리케이션 프록시 대신 Presigned URL을 검토해 서버와 NAT 경로를 줄인다.

HTTP 연결 풀, Keep-Alive와 압축을 조정하되 CPU와 지연 변화도 함께 측정한다.

---

# 실무 적용

## 1단계: 기준선 만들기

변경 전 최소 한 주기의 비용, 사용량, 처리량과 SLO를 같은 시간 범위로 저장한다.

## 2단계: 원인 분리하기

VPC Flow Logs와 서비스 지표로 source, destination, AZ와 바이트 흐름을 그린다.

## 3단계: 작은 변경 적용하기

S3 트래픽에는 S3 Gateway Endpoint와 라우팅 적용 범위를 우선 검토한다.

## 4단계: 가격 조건 확인하기

Interface Endpoint는 NAT 처리량 절감과 endpoint별 고정비 및 트래픽 비용을 함께 비교한다.

## 5단계: 효과 검증하기

변경 후 총비용뿐 아니라 단위 비용, 지연, 오류율과 운영 부담이 어떻게 달라졌는지 확인한다.

---

# 운영 함정

VPC Endpoint는 만들기만 하면 항상 절감되는 무료 경로가 아니다.

고가용성을 위해 AZ별 NAT Gateway를 두는 설계와 단일 NAT의 비용만 비교하면 장애 위험을 누락한다.

ALB 요청 수만 보고 LCU를 추정하면 연결 수나 처리 바이트가 지배하는 워크로드를 놓친다.

---

# 비용과 성능의 Trade-off

같은 AZ 배치는 전송을 줄일 수 있지만 장애 격리와 균형 배치 요구를 훼손하면 안 된다.

압축은 전송량을 줄이지만 CPU 사용량과 응답 지연을 늘릴 수 있다.

네트워크 최적화는 경로를 단축하면서 보안, 가용성, 운영 복잡도를 함께 지켜야 한다.

비용 최적화는 단순한 최저가 선택이 아니라 비즈니스 가치, 가용성, 성능과 운영 가능성 사이의 균형을 찾는 일이다.

---

# 점검 체크리스트

- 비용 변화의 시작 시각과 배포 시각을 비교했는가.

- 서비스, 계정, 태그와 사용 유형으로 비용을 분해했는가.

- 변경 전후의 단위 비용과 SLO를 함께 비교했는가.

- 되돌리기 절차와 담당 owner가 정해져 있는가.

- 최신 Region과 구매 모델의 가격 조건을 확인했는가.

---

# 핵심 요약

NAT Gateway는 시간과 처리 데이터 양을 따로 관찰한다.

AZ 경계, Public IPv4와 ALB LCU를 비용 흐름에 표시한다.

S3 Gateway Endpoint와 Interface Endpoint의 비용 모델을 구분한다.

---

# 다음 장

→ **[Chapter 05. 스토리지와 데이터베이스 비용](/aws-backend/part-14/05-storage-database-cost/)**
