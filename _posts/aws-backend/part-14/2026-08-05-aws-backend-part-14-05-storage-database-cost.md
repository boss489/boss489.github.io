---
title: "Chapter 05. 스토리지와 데이터베이스 비용"
permalink: /aws-backend/part-14/05-storage-database-cost/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. 스토리지와 데이터베이스 비용
## 용량과 I/O와 보존 정책을 함께 설계하기

> **학습 목표**
>
> - EBS 볼륨과 Snapshot의 비용 발생 지점을 설명할 수 있다.
> - S3 Lifecycle로 접근 패턴에 맞는 보존 정책을 설계할 수 있다.
> - RDS와 Aurora의 인스턴스, 저장, I/O와 백업 요소를 구분할 수 있다.

---

# 비용 폭증 시나리오

서비스 트래픽은 줄었지만 스토리지와 데이터베이스 비용이 매달 증가하는 상황을 가정한다.

종료한 EC2의 미연결 EBS, 오래된 Snapshot과 무기한 보관된 S3 버전이 계속 용량을 차지할 수 있다.

RDS Read Replica와 백업 보존 정책도 목적과 사용량을 검토하지 않으면 비용이 누적된다.

---

# 핵심 정의

## EBS Volume

프로비저닝한 용량과 유형별 성능 설정이 비용에 영향을 주는 블록 스토리지다.

## EBS Snapshot

변경 블록을 기반으로 저장되지만 논리적 보존 관계와 삭제 정책을 이해해야 하는 백업이다.

## S3 Lifecycle

객체의 수명과 접근 패턴에 따라 스토리지 클래스 전환 또는 만료를 자동화한다.

## RDS 비용

DB 인스턴스 실행, 저장 공간, I/O 모델, 백업, 전송과 복제 구성이 영향을 준다.

## Aurora I/O 모델

선택한 구성과 엔진 옵션에 따라 I/O 비용 반영 방식이 달라질 수 있으므로 실제 워크로드로 비교해야 한다.

---

# 비용 흐름

```
Spring Boot
  -> EBS ─ volume 성능·snapshot
  -> S3 ─ request·capacity·lifecycle
  -> RDS Instance
       ├ storage / I/O
       ├ backup retention
       └ read replica
  -> 단위 저장·쿼리 비용
```

이 흐름에서 사용량이 생성되는 지점과 청구 데이터에서 보이는 지점을 연결해야 한다.

---

# 비교표

| 대상 | 주요 비용 요소 | 점검 항목 |
|---|---|---|
| EBS | 용량·성능·Snapshot | 미연결 볼륨 |
| S3 | 용량·요청·전환·복원 | 접근 패턴 |
| RDS | 인스턴스·저장·I/O·백업 | 유휴와 쿼리 |
| Read Replica | 복제 인스턴스와 저장 | 실제 읽기 분산 |

비교표는 절대적인 우열이 아니라 워크로드에 맞는 선택 기준을 제공한다.

---

# Cost Explorer와 AWS CLI

다음 명령은 예시 기간의 데이터를 조회하며 실행 계정의 권한과 실제 날짜를 확인해야 한다.

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query "Volumes[].{Id:VolumeId,Size:Size,Type:VolumeType}"
aws rds describe-db-instances \
  --query "DBInstances[].{Id:DBInstanceIdentifier,Class:DBInstanceClass,Storage:AllocatedStorage}"
aws s3api get-bucket-lifecycle-configuration --bucket example-bucket
```

CLI 결과는 청구 확정 시점과 데이터 지연을 고려해 해석한다.

특정 가격 수치는 문서에 고정하지 않고 Region, 시점과 구매 모델에 맞는 공식 가격 정보를 확인한다.

---

# Spring Boot 운영자가 통제할 수 있는 요소

N+1 쿼리와 불필요한 전체 조회를 제거해 DB CPU와 I/O 사용량을 줄인다.

커넥션 풀 크기를 DB 용량에 맞추고 무제한 동시 쿼리로 확장되는 상황을 막는다.

업로드 객체의 보존 기간과 법적 보존 요구를 도메인 정책으로 명확히 정의한다.

---

# 실무 적용

## 1단계: 기준선 만들기

변경 전 최소 한 주기의 비용, 사용량, 처리량과 SLO를 같은 시간 범위로 저장한다.

## 2단계: 원인 분리하기

미연결 EBS와 오래된 Snapshot은 owner와 복구 요구를 확인한 뒤 삭제한다.

## 3단계: 작은 변경 적용하기

S3 Storage Lens와 접근 지표로 Lifecycle 전환 시점과 복원 요구를 검증한다.

## 4단계: 가격 조건 확인하기

RDS는 CPU, 메모리, 연결 수, IOPS, 지연과 쿼리 성능을 함께 보고 Right Sizing한다.

## 5단계: 효과 검증하기

변경 후 총비용뿐 아니라 단위 비용, 지연, 오류율과 운영 부담이 어떻게 달라졌는지 확인한다.

---

# 운영 함정

Snapshot을 무조건 오래 보관하면 복구 선택지는 늘지만 비용과 관리 복잡도도 증가한다.

S3 저빈도 또는 아카이브 계층은 최소 보관, 전환과 복원 조건을 확인하지 않으면 오히려 비효율적일 수 있다.

Read Replica를 추가하면 읽기 성능은 늘 수 있지만 쓰기 병목이나 비효율 쿼리가 자동으로 해결되지는 않는다.

---

# 비용과 성능의 Trade-off

DB 인스턴스를 줄이면 비용은 낮아지지만 메모리 캐시 감소로 I/O와 지연이 늘 수 있다.

백업 보존을 줄이면 저장량은 감소하지만 복구 시점 목표와 감사 요구를 해칠 수 있다.

Aurora의 I/O 관련 선택은 특정 모델이 항상 유리하다고 단정하지 말고 쿼리 패턴과 최신 가격 조건으로 비교한다.

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

스토리지 비용은 현재 연결 여부와 보존 목적부터 확인한다.

RDS 비용은 인스턴스, 저장, I/O, 백업과 복제를 분리해 본다.

Lifecycle과 백업 최적화는 복구 요구와 접근 패턴을 먼저 정의한다.

---

# 다음 장

→ **[Chapter 06. CloudFront와 S3 비용](/aws-backend/part-14/06-cloudfront-s3-cost/)**
