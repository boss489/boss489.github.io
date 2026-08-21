---
title: "Chapter 09. Summary"
permalink: /aws-backend/part-13/09-summary/
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
## CI/CD 핵심 정리
> **학습 목표**
>
> - Part 13의 자동화 흐름을 하나의 Pipeline으로 설명할 수 있다.
> - 배포 전략과 안전장치를 변경 위험에 맞게 선택할 수 있다.
> - Source Commit부터 ECS Task까지 Provenance를 추적할 수 있다.
---
# 전체 아키텍처
```text
GitHub Commit
     |
     v
CI: Gradle Test -> BuildKit Image -> ECR SHA Tag + Digest
     |
     v
CD: Staging -> Smoke Test -> Approval -> Production
     |
     +-> Rolling: New Task -> Healthy -> Old Task Stop
     |
     +-> Blue/Green: Green Test -> Traffic Shift -> Blue Stop
```
CI는 변경을 검증하고 불변 Artifact를 생산한다.

CD는 검증된 Artifact를 환경에 승격하고 배포 결과를 확인한다.

두 책임을 분리하면 Build 실패와 Runtime 배포 실패를 다른 경계에서 처리할 수 있다.
---
# Build Once, Deploy Many
운영용 Image를 스테이징 이후 다시 빌드하면 검증 대상과 배포 대상이 달라진다.

Commit SHA를 Image Tag로 사용하고 ECR Digest를 배포 Manifest에 기록한다.

같은 Digest를 스테이징, Canary와 운영으로 승격한다.

Provenance에는 Source, Builder, Workflow Run, Role과 Digest를 연결한다.
---
# 도구별 책임
| 도구 | 핵심 책임 | 출력 |
|---|---|---|
| GitHub Actions | 저장소 이벤트와 CI | Test 결과와 Image |
| CodeBuild | 관리형 Build 실행 | Artifact와 Manifest |
| CodePipeline | Stage 오케스트레이션 | 승인과 실행 이력 |
| CodeDeploy | Hook과 Traffic 전환 | Deployment 상태 |
| ECS | Task 교체와 정상 상태 유지 | 실행 중 Task |
GitHub Actions OIDC는 장기 Access Key 없이 AWS Role을 Assume한다.

`id-token: write`와 `contents: read`만 부여하고 Trust Policy Claim을 제한한다.

운영 GitHub Environment에는 Reviewer 승인을 적용할 수 있다.
---
# Build 단계
Java 17 Toolchain과 Gradle Wrapper로 Build Runtime을 고정한다.

`./gradlew clean test build`가 실패하면 Docker Build를 실행하지 않는다.

BuildKit Cache는 속도를 높이지만 불변 Image 식별자를 대신하지 않는다.

CodeBuild buildspec은 install, pre_build, build, post_build 순서로 역할을 분리한다.

Artifact는 다음 Stage의 입력이고 Cache는 다시 생성 가능한 최적화 데이터이다.
---
# 배포 전략
| 기준 | Rolling | Blue/Green |
|---|---|---|
| 교체 | 같은 Service에서 점진 교체 | 두 Task Set 간 Traffic 전환 |
| 검증 | Health Check 중심 | Test Listener와 Hook |
| 복구 | 이전 Revision 재배포 | Blue로 Traffic 복귀 |
| 자원 | 비율만큼 추가 | 두 환경 동시 유지 |
Rolling은 `minimumHealthyPercent`와 `maximumPercent`로 배포 중 Capacity 범위를 정한다.

Circuit Breaker는 안정화 실패를 감지하고 이전 완료 배포로 롤백할 수 있다.

Health Check Grace Period는 느린 초기화가 반복 종료로 이어지는 것을 막는다.

ALB Deregistration Delay, ECS `stopTimeout`과 Spring Boot graceful shutdown을 함께 시험한다.

Blue/Green은 Blue와 Green Target Group 및 선택적 Test Listener를 사용한다.

Traffic Shifting 중 오류율과 지연 Alarm을 평가하고 실패하면 Blue로 되돌린다.
---
# 데이터와 기능 안전성
Expand 단계에서 하위 호환 Column과 Table을 추가한다.

구버전과 신버전이 동시에 동작하는 동안 Dual Read 또는 Dual Write를 운영한다.

새 코드가 안정된 뒤 별도 배포에서 Contract 단계로 기존 Schema를 제거한다.

Feature Flag는 기능 활성화와 코드 배포 시점을 분리한다.

Canary는 새 버전에 노출되는 Traffic과 장애 영향을 제한한다.

Smoke Test는 인증, 핵심 API, DB 읽기와 쓰기 같은 사용자 경로를 확인한다.
---
# 롤백 기준
배포 명령 성공만으로 배포 완료를 선언하지 않는다.

Task Health, ALB 오류율, 지연, 애플리케이션 예외와 핵심 비즈니스 지표를 확인한다.

Alarm 기준을 넘으면 자동 Traffic 복귀 또는 이전 Task Definition 배포를 실행한다.

DB의 파괴적 변경은 Code Rollback으로 복구되지 않으므로 Expand and Contract가 필요하다.

Runbook에는 중단 기준, 담당자, Dashboard, 명령과 데이터 검증 절차를 포함한다.
---
# 비용과 성능
Build Runner, Artifact 저장, ECR, Log와 배포 중 추가 Task가 비용을 만든다.

가격은 Region과 실행 방식에 따라 달라지므로 고정 금액을 문서에 박아 두지 않는다.

Cache와 Test 병렬화는 Feedback 시간을 줄이지만 자원 사용과 재현성을 함께 관측한다.

Blue/Green은 복구 속도를 얻는 대신 전환 중 두 환경의 Capacity를 사용한다.
---
# 최종 체크리스트
- CI와 CD 실패 경계가 분리되어 있다.
- 장기 AWS Access Key를 GitHub Secret에 두지 않는다.
- Commit SHA Tag와 Image Digest가 기록된다.
- 스테이징과 운영이 같은 Digest를 사용한다.
- Health Check와 Smoke Test가 사용자 경로를 반영한다.
- Alarm Rollback이 실제로 시험되었다.
- DB Migration이 하위 호환된다.
- 이전 Artifact와 Task Definition이 보존된다.
---
# 다음 Chapter
다음 Chapter는 **Chapter 10. Interview Questions**이다.

CI/CD 설계 의도와 장애 대응을 면접 답변 형태로 정리한다.
