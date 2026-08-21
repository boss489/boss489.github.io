---
title: "Chapter 11. Interview Questions"
permalink: /aws-backend/part-15/11-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 11. Interview Questions
## Troubleshooting 면접 질문

---

# 공통 원칙

## 장애가 발생하면 가장 먼저 무엇을 확인하는가?

영향받는 사용자, 기능, 지역, 오류율과 시작 시각을 확인해 범위와 severity를 정의한다.

그다음 최근 변경을 확인하고 메트릭, 로그와 trace로 검증 가능한 가설을 세운다.

## 서버 재시작을 첫 조치로 하지 않는 이유는 무엇인가?

재시작은 메모리와 연결 상태 같은 증거를 지우고 남은 인스턴스에 부하를 집중시킬 수 있다.

원인과 영향을 확인한 뒤 재시작이 안전한 완화인지 판단해야 한다.

## 메트릭, 로그와 trace의 역할은 어떻게 다른가?

메트릭은 이상 시각과 범위를 보여주고 로그는 사건의 상세 맥락을 제공한다.

trace는 하나의 요청이 여러 서비스와 dependency를 지나는 시간을 연결한다.

---

# ALB

## ALB 502는 무엇을 의미하는가?

ALB가 Target과 정상 HTTP 교환을 완료하지 못했거나 Target이 502를 반환했음을 의미할 수 있다.

Access Log의 `elb_status_code`, `target_status_code`와 `error_reason`으로 두 경우를 구분한다.

## ALB 502의 대표 원인은 무엇인가?

연결 reset, 연결 거부, malformed response, idle timeout 불일치와 Target HTTPS의 TLS 문제가 있다.

## ALB 503과 애플리케이션 503은 어떻게 구분하는가?

ALB 자체 503은 사용할 수 있는 Target 부재와 연관되며 `target_status_code`가 없을 수 있다.

Target 503은 요청이 애플리케이션에 도달했으므로 과부하, rate limit과 dependency 보호 상태를 확인한다.

## 모든 Target이 unhealthy이면 무엇을 확인하는가?

등록 상태, health check 포트와 경로, 성공 코드, 보안 그룹, readiness와 최근 배포를 확인한다.

---

# Compute

## Compute 장애에서 CPU 외에 무엇을 확인하는가?

memory, OOM, disk, inode, file descriptor, socket, health check와 배포 상태를 확인한다.

## ECS Task 종료 원인은 어디서 확인하는가?

`describe-tasks`의 stop code, stopped reason, 컨테이너 reason과 exit code를 확인한다.

## EKS의 CrashLoopBackOff는 어떻게 진단하는가?

`kubectl describe pod`, event, 현재 로그와 `--previous` 로그로 직전 컨테이너 종료 원인을 확인한다.

---

# RDS

## RDS connection exhaustion은 어떻게 확인하는가?

RDS DBConnections와 모든 애플리케이션의 Hikari active, pending, max 및 acquisition timeout을 비교한다.

## connection pool 크기는 어떻게 정하는가?

애플리케이션 인스턴스 수와 관리 및 batch 연결을 합산하고 DB 한도와 실제 workload를 기준으로 정한다.

## lock과 deadlock의 차이는 무엇인가?

lock wait는 다른 transaction의 lock 해제를 기다리는 상태이고 deadlock은 순환 대기를 DB가 감지해 일부 transaction을 중단하는 상태다.

## DB 오류를 무조건 재시도하면 안 되는 이유는 무엇인가?

재시도 폭주는 connection과 CPU를 더 소진하고 멱등하지 않은 쓰기를 중복 실행할 수 있다.

## RDS failover 후 무엇을 확인하는가?

endpoint DNS 재해석, stale connection 제거, 읽기와 쓰기 성공 및 데이터 정합성을 확인한다.

---

# Redis

## Redis 장애에서 가장 먼저 물어야 할 질문은 무엇인가?

Redis 데이터가 재생성 가능한 cache인지 session이나 lock 또는 원본 데이터인지 확인한다.

## eviction은 항상 장애인가?

eviction은 memory policy에 따른 동작이므로 증가율, hit ratio, 원본 부하와 사용자 영향을 함께 판단한다.

## cache stampede는 무엇인가?

인기 key가 동시에 만료되거나 cache가 비워질 때 많은 요청이 원본 저장소로 몰리는 현상이다.

## cache stampede를 어떻게 예방하는가?

TTL jitter, request coalescing, stale cache와 rate limit으로 동시 원본 조회를 줄인다.

---

# DNS

## authoritative DNS와 recursive resolver의 차이는 무엇인가?

authoritative 서버는 zone 원본에 권한 있는 응답을 제공하고 recursive resolver는 대신 조회해 TTL 동안 cache한다.

## NXDOMAIN과 SERVFAIL의 차이는 무엇인가?

NXDOMAIN은 이름이 존재하지 않는다는 응답이고 SERVFAIL은 resolver가 유효한 응답을 완성하지 못했다는 뜻이다.

## `dig +trace`는 언제 사용하는가?

root부터 TLD와 authoritative 서버까지 위임 경로가 어디에서 끊기는지 확인할 때 사용한다.

## DNS 변경이 반영됐는데 JVM이 이전 주소를 쓰는 이유는 무엇인가?

JVM DNS cache와 HTTP client의 기존 keep-alive connection이 이전 주소를 유지할 수 있다.

---

# Network

## Security Group과 NACL의 핵심 차이는 무엇인가?

Security Group은 stateful이고 NACL은 stateless이므로 NACL은 요청과 응답 방향 규칙을 각각 확인해야 한다.

## private subnet에서 internet 연결이 실패하면 무엇을 확인하는가?

subnet route, NAT 상태, NAT subnet의 IGW route, Security Group, NACL과 DNS를 확인한다.

## Reachability Analyzer와 Flow Logs의 차이는 무엇인가?

Reachability Analyzer는 AWS 구성을 모델링해 경로를 분석하고 Flow Logs는 ENI에서 관찰된 network 흐름의 단서를 제공한다.

---

# Incident Response

## Incident Commander의 역할은 무엇인가?

사용자 영향 완화를 최우선으로 두고 역할, 우선순위, 의사결정과 다음 업데이트 시각을 관리한다.

## mitigation-first는 무엇을 의미하는가?

완전한 원인 규명 전에 되돌릴 수 있는 조치로 사용자 영향을 먼저 줄이고 안정화 후 분석을 계속한다는 뜻이다.

## blameless postmortem은 책임을 없애는가?

개인 비난 대신 시스템 조건과 방어 계층을 분석하지만 후속 조치의 owner와 deadline은 명확히 둔다.

## RTO와 RPO의 차이는 무엇인가?

RTO는 목표 복구 시간이고 RPO는 복구 시 허용 가능한 데이터 손실 시점이다.

## incident 종료 전에 무엇을 확인하는가?

오류율과 지연뿐 아니라 핵심 사용자 여정, backlog, 재시도와 데이터 정합성을 확인한다.

---

# 면접 답변의 핵심

도구 이름만 나열하지 않고 영향, 증거, 가설, 안전한 완화와 복구 검증 순서로 설명한다.

확실하지 않은 원인을 단정하지 않고 어떤 증거로 확인할지 말한다.
