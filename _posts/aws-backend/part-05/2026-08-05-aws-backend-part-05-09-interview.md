---
title: "Chapter 09. Interview Questions"
permalink: /aws-backend/part-05/09-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Interview Questions
## Database 면접 질문

---

## RDS란 무엇인가요?

관계형 데이터베이스를 관리형으로 제공하는 AWS 서비스입니다.

## RDS와 Aurora의 차이는 무엇인가요?

RDS는 여러 관계형 DB 엔진을 관리형으로 제공하고, Aurora는 AWS가 만든 MySQL/PostgreSQL 호환 엔진입니다.

## Multi-AZ와 Read Replica의 차이는 무엇인가요?

Multi-AZ는 장애 대비용 고가용성 구성이고, Read Replica는 읽기 부하 분산용 구성입니다.

## Backup과 Restore에서 중요한 기준은 무엇인가요?

RPO와 RTO입니다. 얼마나 최근 시점까지, 얼마나 빨리 복구할 수 있어야 하는지 정해야 합니다.

## Failover 중 애플리케이션은 무엇을 고려해야 하나요?

DB 연결 실패와 재연결을 고려해야 하며, Connection Pool과 재시도 정책을 점검해야 합니다.

## RPO와 RTO는 각각 무엇인가요?

RPO는 장애 시 허용할 수 있는 데이터 손실 시점이고, RTO는 서비스를 복구하기까지 허용되는 시간입니다. 백업 주기와 복구 절차는 이 두 목표에서 결정합니다.

## Read Replica로 쓰기 장애에 대비할 수 있나요?

아닙니다. Read Replica의 주된 목적은 읽기 확장입니다. 쓰기 가용성은 Multi-AZ나 Aurora의 장애 조치 구성을 기준으로 설계합니다.

## Replica Lag는 왜 주의해야 하나요?

쓰기 직후 Replica에서 읽으면 아직 복제되지 않은 이전 값을 볼 수 있습니다. 결제 결과나 방금 변경한 정보처럼 최신 값이 필요한 요청은 Writer에서 읽는 기준을 둡니다.

