---
title: "Chapter 08. Deployment Safety"
permalink: /aws-backend/part-13/08-deployment-safety/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 08. Deployment Safety
## 배포 안전장치
> **학습 목표**
>
> - Expand and Contract DB Migration을 설명할 수 있다.
> - Feature Flag, Canary, Smoke Test와 Alarm Rollback을 조합할 수 있다.
> - 실패 시 실행 가능한 Runbook을 작성할 수 있다.
---
# 수동 배포 실패 시나리오
코드 배포와 동시에 기존 Column을 삭제하면 구버전 Task가 남아 있는 동안 SQL 오류가 발생하고 Traffic 롤백도 DB를 복구하지 못한다.
담당자의 기억과 로컬 상태에 의존하는 배포는 반복 가능하지 않고 감사 가능한 증거도 남기기 어렵다.
자동화의 목적은 명령을 빠르게 실행하는 데 그치지 않고 동일한 입력에 동일한 절차와 검증을 적용하는 데 있다.
---
# Deployment Safety란?
Deployment Safety는 배포 전후의 호환성, 제한 노출, 자동 검증, 관측과 복구 절차를 계층적으로 구성하는 운영 원칙이다.
배포 가능한 단위는 테스트를 통과한 Source Commit과 연결되고 환경별 설정은 Artifact 밖에서 주입되어야 한다.
같은 Artifact를 여러 환경에서 승격해야 검증 결과와 운영 결과 사이의 인과관계를 보존할 수 있다.
---
# 전체 흐름
```text
Expand Schema -> Deploy Compatible Code -> Feature Flag Off
                                           |
                                  Canary Traffic + Smoke
                                           |
                                Alarm Gate -> Full Traffic
                                           |
                                  Contract Schema Later
```
각 단계는 입력 식별자와 출력 식별자를 기록하고 실패하면 다음 단계로 진행하지 않는다.
운영 배포 기록에는 Commit SHA, Image URI와 Digest, 실행 주체, 승인자, 시작과 종료 시각을 남긴다.
---
# 핵심 구성 비교
| 장치 | 목적 | 실패 시 |
|---|---|---|
| Expand/Contract | 버전 간 DB 호환 | Contract 연기 |
| Feature Flag | 기능과 배포 분리 | Flag 비활성화 |
| Canary | 영향 범위 제한 | Traffic 복귀 |
| Smoke Test | 핵심 경로 확인 | 승격 차단 |
| Alarm | 자동 이상 감지 | 자동 롤백 |
표의 선택 기준은 팀 규모보다 장애 영향, 복구 시간 목표, 변경 빈도와 운영 역량이다.
---
# Spring Boot 3.x와 Java 17 연동
Spring Boot 3.x 애플리케이션은 Java 17 Toolchain을 고정하고 Gradle Wrapper로 Runner 간 버전 차이를 줄인다.
```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

tasks.test {
    useJUnitPlatform()
}
```
CI에서는 `./gradlew clean test build`를 실행하고 테스트 실패 시 Image Build와 배포를 중단한다.
Dockerfile은 빌드 결과를 복사하는 Runtime Stage와 최소 JRE Base Image를 사용해 공격 표면과 전송량을 줄인다.
---
# 설정과 자동화 예시
```yaml
deployment:
  feature-flags:
    new-checkout: false
  smoke:
    endpoints:
      - /actuator/health/readiness
      - /api/orders/probe
  rollback:
    alarms:
      - order-api-5xx-rate
      - order-api-p95-latency
      - order-api-task-failures
```
예시의 Account ID, Role ARN, Repository와 Resource 이름은 환경에 맞게 바꾸고 권한은 대상 Resource로 제한한다.
Secret 값은 로그에 출력하지 않고 민감 설정은 AWS Secrets Manager 또는 Parameter Store에서 Runtime에 주입한다.
---
# AWS CLI 확인 예시
```bash
COMMIT_SHA=$(git rev-parse HEAD)
aws ecr describe-images \
  --repository-name order-api \
  --image-ids imageTag="$COMMIT_SHA"

aws ecs describe-services \
  --cluster production-app \
  --services order-api
```
CLI 출력에서 Image Digest와 현재 Task Definition Revision을 대조하면 Source부터 Runtime까지의 연결을 확인할 수 있다.
---
# 실무에서는 어떻게 사용할까
Expand 단계에서 새 Column이나 Table을 추가하고 구버전과 신버전이 모두 동작하는 Dual Read 또는 Dual Write 전환을 설계한다.

Feature Flag는 배포 없이 기능을 끌 수 있게 하지만 오래된 Flag와 분기 코드는 소유자와 제거 일정을 가져야 한다.

Canary 대상에서 인증, 주문 생성, DB 읽기 같은 핵심 Smoke Test를 실행하고 CloudWatch Alarm이 안정된 뒤 Traffic을 확대한다.

배포 전에는 변경 위험도와 복구 가능성을 검토하고 배포 후에는 사용자 관점의 핵심 경로를 확인한다.
---
# 장애와 롤백
Runbook에는 중단 기준, 담당자, Dashboard, Rollback 명령, 데이터 정합성 확인과 사후 조치가 포함되어야 한다.

Rollback Drill을 정기적으로 실행해 IAM 권한, 이전 Artifact, 명령과 연락 경로가 실제로 작동하는지 확인한다.

| 증상 | 확인 지점 | 대응 |
|---|---|---|
| 배포 단계 정지 | Pipeline 또는 Workflow 로그 | 실패 원인을 수정하고 같은 Commit으로 재검증 |
| 새 Task Unhealthy | ECS Event와 Target Health | 전환 중단 후 이전 Revision 유지 |
| 오류율 증가 | ALB, 애플리케이션 Alarm | Traffic 또는 Task Definition 롤백 |
| 데이터 오류 | Migration 이력과 정합성 지표 | 쓰기 중단과 데이터 복구 Runbook 실행 |
롤백은 이전 코드를 다시 빌드하는 작업이 아니라 이미 검증된 이전 Image Digest로 Traffic을 복구하는 작업이어야 한다.
---
# 비용과 성능 고려사항
Runner, CodeBuild Build Minute, Artifact 저장소, ECR Image, CloudWatch Log와 배포 중 추가 Task가 주요 비용 요소이다.
정확한 가격은 Region과 실행 환경에 따라 바뀌므로 고정 숫자보다 사용 시간, 저장량, 전송량과 동시 Capacity를 기준으로 추정한다.
Gradle과 BuildKit Cache는 반복 빌드 시간을 줄이지만 Cache Miss와 오염을 관측하고 Cache 없이도 재현 가능한 빌드를 유지한다.
병렬 Test는 Feedback을 빠르게 하지만 Runner 자원과 외부 테스트 의존성의 동시 처리 한도를 함께 고려한다.
---
# 보안과 Provenance
사람의 IAM User Key 대신 Workload Identity와 단기 Credential을 사용하고 Build Role과 Deploy Role을 분리한다.
Artifact Metadata에는 Commit SHA, Builder, Build Run, 생성 시각, Image Digest와 검증 결과를 연결한다.
승인자는 가변 Tag가 아니라 Digest와 변경 내역을 보고 승인해야 Build Once, Deploy Many 원칙이 유지된다.
Audit Log와 CloudTrail을 보존해 누가 어떤 Role로 어느 Artifact를 배포했는지 추적한다.
---
# 운영 체크리스트
- Source Commit이 Review와 필수 Test를 통과했는지 확인한다.
- Image Tag가 전체 Commit SHA이고 ECR Digest가 기록되었는지 확인한다.
- 스테이징과 운영이 같은 Artifact Digest를 사용하는지 확인한다.
- 배포 Role과 Runtime Role의 권한이 분리되었는지 확인한다.
- Health Check, Smoke Test와 Alarm이 정상 동작하는지 확인한다.
- 이전 Revision과 Image가 롤백 기간 동안 보존되는지 확인한다.
- DB 변경이 구버전과 신버전의 동시 실행을 허용하는지 확인한다.
- Runbook의 명령과 담당자 연락 경로가 최신인지 확인한다.
---
# 기억해야 할 내용
- Deployment Safety는 배포 전후의 호환성, 제한 노출, 자동 검증, 관측과 복구 절차를 계층적으로 구성하는 운영 원칙이다.
- CI와 CD는 책임과 실패 경계를 분리한다.
- Commit SHA Tag와 Image Digest로 Artifact를 불변하게 식별한다.
- 한 번 빌드한 Artifact를 환경마다 다시 빌드하지 않고 승격한다.
- 배포 성공은 명령 성공이 아니라 Health, Smoke와 Alarm 검증까지 포함한다.
- 이전 Artifact와 실행 가능한 Runbook이 있어야 롤백이 복구 수단이 된다.
---


















# 다음 Chapter
다음 Chapter는 Part 13의 CI/CD 구성 요소와 안전한 배포 원칙을 하나의 흐름으로 정리한다.
