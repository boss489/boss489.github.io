---
title: "Chapter 01. CI/CD Overview"
permalink: /aws-backend/part-13/01-cicd-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 01. CI/CD Overview
## CI/CD 전체 흐름
> **학습 목표**
>
> - CI와 CD의 책임을 분리해 설명할 수 있다.
> - Build Once, Deploy Many가 필요한 이유를 설명할 수 있다.
> - Commit SHA와 Image Digest로 배포 출처를 추적할 수 있다.
---
# 수동 배포 실패 시나리오
개발자가 로컬에서 테스트와 빌드를 생략한 채 JAR를 서버에 복사하면 동일한 Commit이어도 머신마다 결과가 달라질 수 있다.
담당자의 기억과 로컬 상태에 의존하는 배포는 반복 가능하지 않고 감사 가능한 증거도 남기기 어렵다.
자동화의 목적은 명령을 빠르게 실행하는 데 그치지 않고 동일한 입력에 동일한 절차와 검증을 적용하는 데 있다.
---
# CI/CD Overview란?
CI는 변경을 통합하기 전에 테스트와 빌드로 품질을 검증하는 과정이고 CD는 검증된 Artifact를 환경에 전달하거나 배포하는 과정이다.
배포 가능한 단위는 테스트를 통과한 Source Commit과 연결되고 환경별 설정은 Artifact 밖에서 주입되어야 한다.
같은 Artifact를 여러 환경에서 승격해야 검증 결과와 운영 결과 사이의 인과관계를 보존할 수 있다.
---
# 전체 흐름
```text
Developer -> Git Push -> CI Test -> Build -> Immutable Artifact
                                             |
                                             v
                           Staging -> Approval -> Production
                                  CD and deployment evidence
```
각 단계는 입력 식별자와 출력 식별자를 기록하고 실패하면 다음 단계로 진행하지 않는다.
운영 배포 기록에는 Commit SHA, Image URI와 Digest, 실행 주체, 승인자, 시작과 종료 시각을 남긴다.
---
# 핵심 구성 비교
| 구분 | CI | CD |
|---|---|---|
| 주요 책임 | 테스트와 빌드 | 환경별 배포와 검증 |
| 입력 | Source Commit | 불변 Artifact |
| 실패 대응 | 통합 차단 | 중단 또는 롤백 |
| 증거 | Test 결과와 SBOM | 배포 이력과 Alarm |
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
name: ci
on: [pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: gradle
      - run: ./gradlew test build
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
한 번 빌드한 Image를 스테이징과 운영에 같은 Digest로 배포해야 환경별 재빌드 차이를 제거할 수 있다.

Image Tag에는 Git Commit SHA를 사용하고 배포 기록에는 ECR Digest, Workflow Run, Builder Role을 함께 남긴다.

Provenance는 어떤 Source와 Builder가 어떤 절차로 Artifact를 만들었는지 검증할 수 있게 하는 공급망 증거이다.

배포 전에는 변경 위험도와 복구 가능성을 검토하고 배포 후에는 사용자 관점의 핵심 경로를 확인한다.
---
# 장애와 롤백
CI 성공을 배포 성공으로 오해하면 Health Check 이후의 런타임 장애를 놓친다.

운영에서 다시 빌드하면 검증한 Artifact와 배포한 Artifact가 달라져 롤백 기준이 무너진다.

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
- CI는 변경을 통합하기 전에 테스트와 빌드로 품질을 검증하는 과정이고 CD는 검증된 Artifact를 환경에 전달하거나 배포하는 과정이다.
- CI와 CD는 책임과 실패 경계를 분리한다.
- Commit SHA Tag와 Image Digest로 Artifact를 불변하게 식별한다.
- 한 번 빌드한 Artifact를 환경마다 다시 빌드하지 않고 승격한다.
- 배포 성공은 명령 성공이 아니라 Health, Smoke와 Alarm 검증까지 포함한다.
- 이전 Artifact와 실행 가능한 Runbook이 있어야 롤백이 복구 수단이 된다.
---



















# 다음 Chapter
다음 Chapter는 GitHub Actions에서 OIDC로 AWS 권한을 안전하게 얻고 ECR까지 Image를 전달하는 방법을 다룬다.
