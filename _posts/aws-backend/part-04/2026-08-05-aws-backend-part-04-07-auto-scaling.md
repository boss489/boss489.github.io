---
title: "Chapter 07. Auto Scaling"
permalink: /aws-backend/part-04/07-auto-scaling/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Auto Scaling
## 트래픽 변화에 맞춰 서버 수 조절하기

> **학습 목표**
>
> - Auto Scaling의 목적을 설명할 수 있다.
> - Desired, Min, Max Capacity의 의미를 이해한다.
> - Scaling Policy가 어떤 지표를 기준으로 동작하는지 설명할 수 있다.
> - Scale Out과 Scale In의 생명주기를 안전하게 설계할 수 있다.

---

# 왜 Auto Scaling이 필요한가

쇼핑몰의 평상시 요청은 서버 두 대로 처리할 수 있지만 할인 행사 때 트래픽이 열 배로 증가한다고 가정해 보자.

최대 트래픽에 맞춰 서버를 항상 실행하면 대부분의 시간에 자원이 낭비된다.

반대로 평상시 용량만 유지하면 행사 시작 후 CPU와 응답 시간이 급증하고 장애가 발생할 수 있다.

Auto Scaling은 수요에 따라 처리 용량을 늘리고 줄여 가용성과 비용 효율을 함께 관리한다.

다만 새 서버 시작에는 시간이 걸리므로 정책만 활성화한다고 모든 급증을 즉시 흡수할 수 있는 것은 아니다.

---

# Auto Scaling이란?

Auto Scaling은 부하에 따라 서버나 Task 수를 자동으로 조절하는 기능이다.

트래픽이 늘면 인스턴스나 Task를 늘리고, 줄면 다시 줄인다.

EC2에서는 Auto Scaling Group이 인스턴스 수를 관리하고 ECS에서는 Service Auto Scaling이 Task 수를 관리한다.

```
CloudWatch Metric
        │
        ▼
Scaling Policy
   ┌────┴────┐
   │         │
Scale Out  Scale In
   │         │
Target 추가 Target 제거
```

Auto Scaling은 서버의 수를 조절하며 애플리케이션 코드의 병목이나 데이터베이스 한계를 자동으로 해결하지 않는다.

---

# Capacity

| 값 | 의미 |
|---|---|
| Min | 최소 유지 수 |
| Desired | 현재 유지하려는 수 |
| Max | 최대 확장 수 |

Max를 너무 낮게 잡으면 트래픽 증가에 대응하지 못한다.

Min은 장애와 배포 중에도 유지해야 할 최소 가용 용량을 기준으로 정한다.

Desired는 정책과 운영 명령에 따라 Min과 Max 범위 안에서 변한다.

```
Min = 2
Desired = 4
Max = 10

허용 범위: 2 ───── 4 ────────── 10
            최소    현재 목표       최대
```

Max를 크게 설정하는 것만으로 안전해지는 것은 아니며 계정 할당량과 하위 시스템 용량도 확인해야 한다.

---

# Scaling Policy

Scaling Policy는 어떤 조건에서 늘리고 줄일지 정한다.

| 정책 | 동작 | 대표 용도 |
|---|---|---|
| Target Tracking | 지표를 목표값 근처로 유지 | CPU, 요청 수 기반 일반 확장 |
| Step Scaling | 경보 크기에 따라 단계별 조정 | 부하 수준별 확장 폭 제어 |
| Scheduled Scaling | 지정 시간에 용량 변경 | 정기 행사, 업무 시작 시간 |
| Predictive Scaling | 과거 패턴으로 필요 용량 예측 | 반복되는 트래픽 패턴 |

Target Tracking은 지표를 목표값 근처로 유지하며 일반적인 API 확장에 적합하다.

Scheduled Scaling은 예측 가능한 트래픽 전에 용량을 확보하고 반응형 정책은 예상 밖 증가에 대응한다.

---

# 어떤 지표를 선택할까

대표적인 지표는 다음과 같다.

- CPU 사용률
- 메모리 사용률
- ALB Target당 Request Count
- Queue Length 또는 처리 지연

| 워크로드 | 유용한 지표 | 이유 |
|---|---|---|
| CPU 연산 API | CPU 사용률 | 처리량과 CPU가 함께 증가 |
| 일반 HTTP API | Target당 요청 수 | ALB 부하를 직접 반영 |
| 비동기 Worker | Queue 적체량 | 대기 작업을 용량과 연결 |
| 메모리 집약 처리 | 메모리 사용률 | CPU만으로 병목을 찾기 어려움 |

EC2 기본 지표만으로 메모리 사용률을 얻지 못하는 경우 CloudWatch Agent나 애플리케이션 지표가 필요하다.

평균 CPU가 낮아도 데이터베이스 연결 수나 외부 API 제한이 병목이면 서버를 늘려도 처리량이 증가하지 않을 수 있다.

---

# Scale Out 내부 흐름

Scale Out은 지표가 임계값을 넘는 순간 바로 사용자 처리 용량이 늘어나는 작업이 아니다.

```
지표 상승
  └── 정책 평가
      └── EC2 또는 Task 시작
          └── 애플리케이션 초기화
              └── Target Group 등록
                  └── Health Check 통과
                      └── 요청 처리
```

전체 확장 시간은 인스턴스 시작, 이미지 다운로드, JVM 시작, 캐시 준비와 [Health Check](/aws-backend/part-04/06-health-check/)에 영향을 받는다.

급격한 트래픽이 예상되면 Min Capacity, Scheduled Scaling, 애플리케이션 시작 시간을 함께 개선해야 한다.

---

# Scale In 내부 흐름

Scale In은 비용을 줄이지만 잘못 설계하면 처리 중인 요청을 끊거나 세션을 잃게 한다.

```
지표 하락
  └── 정책 평가
      └── Target 등록 해제
          └── 새 요청 중단
              └── 진행 중 요청 완료
                  └── 프로세스 종료
```

[Target Group](/aws-backend/part-04/05-target-group/)의 등록 해제 지연과 애플리케이션의 Graceful Shutdown이 조화를 이루어야 한다.

Scale In 정책은 Scale Out보다 보수적으로 설정하여 짧은 트래픽 감소에 서버 수가 반복해서 오르내리는 현상을 줄인다.

---

# Spring Boot에서는 어떻게 쓰는가

종료 신호를 받으면 새 요청 수락을 중단하고 처리 중인 요청을 마치도록 Graceful Shutdown을 활성화한다.

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

애플리케이션은 서버 로컬 메모리에만 세션이나 필수 작업 상태를 저장하지 않아야 한다.

긴 비동기 작업은 종료 신호를 처리하고 재시도 가능한 Queue 기반 구조를 사용하는 것이 안전하다.

시작 과정에서는 무거운 동기 초기화를 줄여 Scale Out 후 실제 처리 용량이 빠르게 증가하게 한다.

---

# 실무에서는 어떻게 사용할까

일반 API는 두 개 이상의 AZ에 최소 용량을 분산하여 단일 인스턴스나 AZ 장애에 대비한다.

ALB Target당 요청 수를 기준으로 Target Tracking을 구성하고 CPU와 응답 시간을 보조 지표로 모니터링한다.

예정된 행사 전에는 Scheduled Scaling으로 용량을 미리 확보한다.

배포 시 새 인스턴스가 정상 상태가 된 뒤 기존 인스턴스를 순차 종료하여 가용 용량을 유지한다.

데이터베이스 연결 수와 외부 API Rate Limit도 최대 인스턴스 수를 감당할 수 있는지 검증한다.

---

# 장애 사례와 주의할 점

## 트래픽이 늘어도 확장되지 않는 경우

Max Capacity 도달, CloudWatch Alarm 미발생, 잘못된 지표 차원, 인스턴스 시작 실패를 확인한다.

정책만 보지 말고 Scaling Activity의 실패 사유를 확인해야 한다.

## 확장되었지만 응답 시간이 줄지 않는 경우

데이터베이스 Lock, Connection Pool, 외부 API, 단일 Queue Consumer 같은 공유 병목일 수 있다.

요청 처리 구간별 지표를 확인하여 수평 확장 가능한 병목인지 판단한다.

## 서버 수가 계속 오르내리는 경우

Scale Out과 Scale In 임계값이 너무 가깝거나 준비 시간이 실제보다 짧을 수 있다.

평가 기간과 Warmup을 조정하고 Scale In을 더 보수적으로 만든다.

## Scale In 중 요청이 실패하는 경우

등록 해제 전에 프로세스가 종료되거나 Graceful Shutdown 제한 시간이 짧을 수 있다.

ALB 등록 해제 시간과 종료 순서를 실제 장시간 요청으로 테스트한다.

---

# 기억해야 할 내용

- Auto Scaling은 수요에 따라 처리 용량을 조절하여 가용성과 비용을 함께 다룬다.
- Min, Desired, Max를 구분해야 한다.
- 워크로드 처리량과 함께 변하는 지표를 Scaling Policy에 선택한다.
- Scale Out에는 시작과 Health Check 시간이 필요하므로 급증 전에 준비해야 한다.
- Scale In에는 Target 등록 해제와 Graceful Shutdown이 필요하다.
- 공유 데이터베이스와 외부 API가 Max Capacity를 감당하는지 확인한다.
- 확장 활동의 성공 여부와 정상 Target 수를 지속해서 모니터링한다.

---

# 다음 Chapter

다음 Chapter에서는 서버가 여러 대로 늘어날 때 필요한 **Session과 Keep Alive**를 학습한다.

서버 로컬 세션의 한계, Redis를 통한 세션 외부화, 연결 Timeout 불일치로 발생하는 장애를 살펴본다.

