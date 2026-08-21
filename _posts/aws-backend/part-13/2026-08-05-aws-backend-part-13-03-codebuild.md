---
title: "Chapter 03. CodeBuild"
permalink: /aws-backend/part-13/03-codebuild/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 03. CodeBuild
## CodeBuild와 buildspec
> **학습 목표**
>
> - CodeBuild Project, Service Role, Build Environment의 역할을 설명할 수 있다.
> - buildspec의 네 Phase를 구분할 수 있다.
> - Artifact와 Cache를 서로 다른 목적으로 구성할 수 있다.
---
# 수동 배포 실패 시나리오
팀원이 공용 빌드 서버에 접속해 의존성과 Docker를 수동 갱신하면 빌드 환경이 누적 변경되어 재현성이 사라진다.
담당자의 기억과 로컬 상태에 의존하는 배포는 반복 가능하지 않고 감사 가능한 증거도 남기기 어렵다.
자동화의 목적은 명령을 빠르게 실행하는 데 그치지 않고 동일한 입력에 동일한 절차와 검증을 적용하는 데 있다.
---
# CodeBuild란?
CodeBuild는 Source를 임시 Build Environment에서 실행하고 로그, Artifact와 상태를 외부 단계에 전달하는 AWS 관리형 빌드 서비스이다.
배포 가능한 단위는 테스트를 통과한 Source Commit과 연결되고 환경별 설정은 Artifact 밖에서 주입되어야 한다.
같은 Artifact를 여러 환경에서 승격해야 검증 결과와 운영 결과 사이의 인과관계를 보존할 수 있다.
---
# 전체 흐름
```text
Source -> CodeBuild Project -> Ephemeral Environment
                              |
          install -> pre_build -> build -> post_build
                              |
                 Artifact + Logs + Image Digest
```
각 단계는 입력 식별자와 출력 식별자를 기록하고 실패하면 다음 단계로 진행하지 않는다.
운영 배포 기록에는 Commit SHA, Image URI와 Digest, 실행 주체, 승인자, 시작과 종료 시각을 남긴다.
---
# 핵심 구성 비교
| 요소 | 책임 | 주의점 |
|---|---|---|
| Project | Source와 환경 연결 | 변수와 Timeout 관리 |
| Service Role | ECR와 Log 접근 | 최소 권한 |
| Artifact | 단계 간 결과 전달 | 불변성과 보존 |
| Cache | 재사용 가능한 의존성 | 정답 Artifact가 아님 |
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
version: 0.2
phases:
  install:
    runtime-versions:
      java: corretto17
  pre_build:
    commands:
      - ./gradlew test
      - aws ecr get-login-password | docker login --username AWS --password-stdin "$ECR_REGISTRY"
      - IMAGE_TAG="$CODEBUILD_RESOLVED_SOURCE_VERSION"
  build:
    commands:
      - DOCKER_BUILDKIT=1 docker build -t "$ECR_REPOSITORY:$IMAGE_TAG" .
  post_build:
    commands:
      - docker push "$ECR_REPOSITORY:$IMAGE_TAG"
      - printf '[{"name":"order-api","imageUri":"%s"}]' "$ECR_REPOSITORY:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
cache:
  paths:
    - '/root/.gradle/caches/**/*'
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
`install`은 Runtime 준비, `pre_build`는 인증과 테스트, `build`는 컴파일과 Image 생성, `post_build`는 Push와 배포 Manifest 생성을 담당한다.

Docker Build가 필요한 환경은 권한과 격리를 검토하고 ECR 권한을 Build Service Role에 최소 범위로 부여한다.

S3 또는 Pipeline Artifact는 다음 단계에 전달할 결과이고 Cache는 Gradle 의존성처럼 다시 내려받아도 되는 데이터이다.

배포 전에는 변경 위험도와 복구 가능성을 검토하고 배포 후에는 사용자 관점의 핵심 경로를 확인한다.
---
# 장애와 롤백
Cache 오염이 의심되면 Cache 없이 재빌드해 Source와 Lockfile만으로 결과가 재현되는지 확인한다.

Source Version이 Branch 이름이면 가변적이므로 `CODEBUILD_RESOLVED_SOURCE_VERSION`의 Commit SHA를 Tag에 사용한다.

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
- CodeBuild는 Source를 임시 Build Environment에서 실행하고 로그, Artifact와 상태를 외부 단계에 전달하는 AWS 관리형 빌드 서비스이다.
- CI와 CD는 책임과 실패 경계를 분리한다.
- Commit SHA Tag와 Image Digest로 Artifact를 불변하게 식별한다.
- 한 번 빌드한 Artifact를 환경마다 다시 빌드하지 않고 승격한다.
- 배포 성공은 명령 성공이 아니라 Health, Smoke와 Alarm 검증까지 포함한다.
- 이전 Artifact와 실행 가능한 Runbook이 있어야 롤백이 복구 수단이 된다.
---









# 다음 Chapter
다음 Chapter는 Source, Build, Approval, Deploy를 CodePipeline Stage로 연결하는 방법을 다룬다.
