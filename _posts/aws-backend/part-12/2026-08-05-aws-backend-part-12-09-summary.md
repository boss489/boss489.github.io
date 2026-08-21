---
title: "Chapter 09. Part 12 Summary"
permalink: /aws-backend/part-12/09-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Part 12 Summary
## AWS Security 핵심 정리

---

# 보안 모델

Shared Responsibility Model에서 AWS는 클라우드 자체를 보호하고 고객은 데이터, Identity와 서비스 설정을 보호한다.

Defense in Depth는 네트워크, Identity, 암호화, 탐지와 복구 통제를 여러 계층으로 겹친다.

최소 권한은 필요한 Action, Resource와 Condition만 허용하고 사용 기록으로 계속 줄이는 과정이다.

장기 액세스 키는 피하고 Workload에는 Role과 STS 임시 자격 증명을 사용한다.

---

# Identity 선택

| 주체 | 권장 방식 |
|---|---|
| 사람 | Identity Center 또는 개별 IAM User와 MFA |
| 같은 직무 | Group 또는 Permission Set |
| ECS | ECS Task Role |
| EC2 | Instance Profile |
| EKS | IRSA |
| 교차 계정 | AssumeRole |

Root user는 MFA로 보호하고 Root 액세스 키를 만들지 않으며 일상 작업에 사용하지 않는다.

---

# Policy 평가

모든 요청은 Implicit Deny에서 시작하고 Explicit Allow가 있어야 허용된다.

Explicit Deny는 모든 Allow보다 우선한다.

`Action`은 작업, `Resource`는 대상, `Condition`은 적용 조건, `Principal`은 주체다.

Identity Policy와 Resource Policy는 권한을 부여할 수 있다.

Permissions Boundary와 SCP는 권한의 최대 범위를 제한할 뿐 권한을 직접 부여하지 않는다.

Trust Policy는 누가 Role을 맡는지 결정하고 Permission Policy는 맡은 뒤 무엇을 하는지 결정한다.

---

# 암호화와 비밀

KMS Envelope Encryption은 Data Key로 데이터를 암호화하고 KMS Key로 Data Key를 보호한다.

Key Policy와 IAM Policy를 함께 검토하고 Key 관리자와 사용자를 분리한다.

Rotation은 새 암호화에 사용할 Key Material을 바꾸지만 기존 데이터를 자동 재암호화하지 않는다.

Grant는 AWS 서비스나 Principal에 제한된 KMS 작업을 위임한다.

암호화와 데이터 접근 권한은 별개이므로 둘 다 최소화해야 한다.

Secrets Manager는 비밀 저장, 버전과 Rotation에 적합하고 Parameter Store는 일반 설정과 계층형 파라미터에도 적합하다.

Secret 원문은 코드, 환경 변수, CLI 기록, 로그와 예외에 남기지 않는다.

캐싱은 조회 비용과 지연을 줄이지만 Rotation 반영을 늦추므로 TTL과 실패 정책이 필요하다.

---

# Spring Boot 원칙

AWS SDK v2의 `DefaultCredentialsProvider`로 실행 환경 Role의 임시 자격 증명을 사용한다.

SDK Client는 Spring Bean으로 재사용하고 Service에는 생성자 주입한다.

설정 파일에는 Secret 값이 아니라 Secret ID와 Resource 이름만 둔다.

Spring Cloud AWS는 설정 통합을 위한 선택지이며 호환성, 갱신과 장애 동작을 확인한다.

권한 오류를 장기 관리자 키로 우회하지 않는다.

---

# Security Operations

| 서비스 | 역할 |
|---|---|
| CloudTrail | API 활동 기록 |
| Config | Resource 구성 이력과 준수 평가 |
| IAM Access Analyzer | 외부 접근과 미사용 권한 분석 |
| GuardDuty | 위협 행위 탐지 |
| Security Hub | Finding과 보안 상태 집계 |

Incident Response는 준비, 탐지, Triage, 격리, 조사, 제거, 복구와 회고로 이어진다.

자동 대응에는 최소 권한, 오탐 보호, 사람 승인과 Rollback이 필요하다.

---

# 최종 점검

- Root MFA와 Root 사용 경보가 구성되어 있는가.
- 사람과 Workload의 Identity가 분리되어 있는가.
- 장기 액세스 키가 코드와 환경 변수에 없는가.
- Role의 Trust Policy와 Permission Policy가 최소화되어 있는가.
- SCP와 Permissions Boundary를 권한 부여 정책으로 오해하지 않는가.
- KMS Key 삭제와 비활성화 절차가 있는가.
- Secret Rotation과 애플리케이션 캐시가 함께 테스트되었는가.
- CloudTrail 로그가 중앙에서 보호되는가.
- Finding 소유자와 대응 Runbook이 정해져 있는가.
- 사고 복구 뒤 권한과 탐지 규칙을 개선하는가.

---

# 다음 Chapter

다음 Chapter에서는 AWS Security 면접 질문과 모범 답변을 통해 핵심 개념을 점검한다.
