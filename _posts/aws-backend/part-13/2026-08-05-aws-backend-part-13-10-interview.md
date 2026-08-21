---
title: "Chapter 10. Interview Questions"
permalink: /aws-backend/part-13/10-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 10. Interview Questions
## CI/CD 면접 질문
---
## CI와 CD의 차이는 무엇인가요?
**모범 답변:** CI는 코드 변경을 Test와 Build로 지속 검증해 통합 위험을 줄이고 CD는 검증된 Artifact를 환경에 전달하거나 배포해 운영 변경을 관리합니다.
**꼬리 질문:** CI 성공 후에도 CD가 실패할 수 있는 사례는 무엇인가요?
**꼬리 질문 답변:** Image는 정상이어도 IAM, Task Definition, Health Check, Capacity 또는 Runtime 설정 문제로 배포가 실패할 수 있습니다.
---
## Build Once, Deploy Many가 왜 필요한가요?
**모범 답변:** 한 번 검증한 동일 Artifact를 환경에 승격해야 재빌드 차이를 없애고 어떤 결과물을 운영에 배포했는지 증명할 수 있기 때문입니다.
**꼬리 질문:** Artifact를 어떻게 식별하나요?
**꼬리 질문 답변:** Commit SHA Tag와 Registry Digest를 함께 기록하고 배포 Manifest에는 가능하면 Digest를 고정합니다.
---
## GitHub Actions에서 AWS Access Key 대신 무엇을 사용하나요?
**모범 답변:** OIDC Token으로 AWS STS의 Role을 Assume해 작업 단위의 단기 Credential을 사용합니다.
**꼬리 질문:** 권한은 어떻게 최소화하나요?
**꼬리 질문 답변:** `id-token: write`와 `contents: read`만 선언하고 AWS Trust Policy의 저장소, Branch 또는 Environment Claim을 제한합니다.
---
## GitHub Environment Approval의 역할은 무엇인가요?
**모범 답변:** 운영 Job 실행 전에 지정 Reviewer의 승인을 요구해 자동 Build와 운영 변경 권한 사이에 통제 지점을 만듭니다.
**꼬리 질문:** 승인자는 무엇을 확인해야 하나요?
**꼬리 질문 답변:** Commit, Image Digest, 변경 내역, Test와 Smoke 결과, Alarm 상태와 Rollback 계획을 확인해야 합니다.
---
## CodeBuild buildspec의 Phase를 설명해 주세요.
**모범 답변:** install은 Runtime, pre_build는 인증과 Test, build는 Compile과 Image 생성, post_build는 Push와 Manifest 생성을 담당합니다.
**꼬리 질문:** Cache와 Artifact의 차이는 무엇인가요?
**꼬리 질문 답변:** Cache는 다시 생성 가능한 속도 최적화 데이터이고 Artifact는 다음 Stage가 소비하는 식별된 결과물입니다.
---
## CodePipeline은 어떤 역할을 하나요?
**모범 답변:** Source, Build, Approval, Deploy Action의 순서와 Artifact 전달, 상태와 이력을 관리합니다.
**꼬리 질문:** Role을 하나로 합치면 왜 안 되나요?
**꼬리 질문 답변:** Pipeline, Build와 Deploy의 책임과 필요한 Resource가 달라 하나의 Role은 침해 범위와 오용 가능성을 키웁니다.
---
## CodeDeploy의 EC2와 ECS 배포 차이는 무엇인가요?
**모범 답변:** EC2/On-Premises는 Agent가 파일과 Script Hook을 실행하고 ECS는 새 Task Set과 Load Balancer Traffic 전환을 관리합니다.
**꼬리 질문:** ECS AppSpec에는 무엇이 있나요?
**꼬리 질문 답변:** Task Definition, Container 이름과 Port, 그리고 Traffic 전후 검증용 Lambda Lifecycle Hook이 있습니다.
---
## ECS Rolling Deployment를 설명해 주세요.
**모범 답변:** 새 Task를 점진적으로 늘리고 Health를 확인한 뒤 기존 Task를 줄이는 같은 Service 안의 교체 방식입니다.
**꼬리 질문:** minimumHealthyPercent와 maximumPercent는 무엇인가요?
**꼬리 질문 답변:** 배포 중 유지할 최소 정상 비율과 허용할 최대 Task 비율로 Capacity와 배포 속도를 함께 제한합니다.
---
## Deployment Circuit Breaker는 무엇인가요?
**모범 답변:** 새 ECS 배포가 안정 상태에 도달하지 못하는 것을 감지하고 설정에 따라 이전 완료 배포로 롤백합니다.
**꼬리 질문:** 자동 롤백이 실패하는 경우는 무엇인가요?
**꼬리 질문 답변:** 이전 Image가 삭제되었거나 Capacity와 IAM 문제가 계속되거나 DB 변경이 하위 호환되지 않으면 복구되지 않을 수 있습니다.
---
## Health Check Grace Period가 필요한 이유는 무엇인가요?
**모범 답변:** Spring Boot 초기화 중 일시적인 실패 판정을 무시해 정상 Task가 시작 전에 반복 종료되는 것을 막기 위해서입니다.
**꼬리 질문:** 너무 길면 어떤 문제가 있나요?
**꼬리 질문 답변:** 실제 실패 Task의 감지와 배포 실패 판정이 늦어집니다.
---
## Graceful Shutdown은 배포와 어떻게 연결되나요?
**모범 답변:** ALB가 Target을 등록 해제한 뒤 ECS Stop Signal을 보내면 Spring Boot가 진행 중 요청을 마치고 제한 시간 안에 종료합니다.
**꼬리 질문:** 어떤 값을 함께 보나요?
**꼬리 질문 답변:** ALB Deregistration Delay, 애플리케이션 종료 유예와 ECS `stopTimeout`을 함께 검증합니다.
---
## Blue/Green Deployment를 설명해 주세요.
**모범 답변:** 기존 Blue와 새 Green Task Set을 동시에 유지하고 Green을 검증한 뒤 Load Balancer Traffic을 전환하는 방식입니다.
**꼬리 질문:** Target Group이 왜 두 개인가요?
**꼬리 질문 답변:** 두 Task Set을 격리해 Test Listener 검증과 운영 Traffic 복귀를 독립적으로 수행하기 위해서입니다.
---
## Traffic Shifting 중 무엇을 관찰하나요?
**모범 답변:** ALB 오류율과 지연, Task Health, 애플리케이션 예외, 핵심 비즈니스 성공률을 관찰합니다.
**꼬리 질문:** 문제가 생기면 어떻게 하나요?
**꼬리 질문 답변:** 추가 전환을 멈추고 Traffic을 Blue로 복귀한 뒤 Green의 로그와 배포 증거를 보존합니다.
---
## Blue/Green의 비용 단점은 무엇인가요?
**모범 답변:** 전환과 복구 대기 동안 Blue와 Green Capacity를 동시에 유지하고 Log와 Test 자원도 추가로 사용합니다.
**꼬리 질문:** 그래도 선택할 때는 언제인가요?
**꼬리 질문 답변:** 변경 위험이 높고 사전 검증과 빠른 Traffic 복구의 가치가 추가 자원보다 클 때 선택합니다.
---
## DB Migration은 어떻게 안전하게 하나요?
**모범 답변:** Expand 단계로 하위 호환 Schema를 추가하고 양쪽 버전이 동작한 뒤 새 코드 전환과 검증 후 Contract 단계로 제거합니다.
**꼬리 질문:** Rollback과의 관계는 무엇인가요?
**꼬리 질문 답변:** 구버전이 새 Schema에서도 동작해야 Traffic 또는 코드 Rollback이 실제 복구 수단이 됩니다.
---
## Feature Flag와 배포의 차이는 무엇인가요?
**모범 답변:** 배포는 코드를 Runtime에 전달하고 Feature Flag는 이미 배포한 기능의 노출을 Runtime에서 제어합니다.
**꼬리 질문:** Flag의 운영 위험은 무엇인가요?
**꼬리 질문 답변:** 오래된 분기가 복잡도를 키우므로 소유자, 관측 지표와 제거 기한을 정해야 합니다.
---
## Canary와 Smoke Test의 목적은 무엇인가요?
**모범 답변:** Canary는 노출 범위를 제한하고 Smoke Test는 새 버전의 핵심 사용자 경로가 실제 환경에서 작동하는지 빠르게 확인합니다.
**꼬리 질문:** Health Check만으로 부족한 이유는 무엇인가요?
**꼬리 질문 답변:** 프로세스 생존은 확인해도 인증, 주문, DB 쓰기 같은 비즈니스 흐름의 정상 여부는 보장하지 못합니다.
---
## Alarm 기반 Rollback은 어떻게 설계하나요?
**모범 답변:** 배포 구간에 오류율, 지연, Task 실패와 비즈니스 지표 Alarm을 연결하고 임계 초과 시 Traffic 복구를 자동 실행합니다.
**꼬리 질문:** Alarm이 너무 민감하면 어떻게 되나요?
**꼬리 질문 답변:** 일시 변동으로 정상 배포가 취소되므로 기간, 데이터 포인트와 Missing Data 처리 방식을 부하 특성에 맞춰 조정합니다.
---
## 배포 Runbook에는 무엇이 있어야 하나요?
**모범 답변:** 중단과 롤백 기준, 담당자, Dashboard, 명령, 데이터 정합성 확인, 커뮤니케이션과 사후 조치를 포함해야 합니다.
**꼬리 질문:** Runbook의 신뢰성을 어떻게 확인하나요?
**꼬리 질문 답변:** 정기 Rollback Drill에서 권한, Artifact, 명령과 연락 경로가 실제로 작동하는지 검증합니다.
---
## CI/CD 비용을 어떻게 최적화하나요?
**모범 답변:** Gradle과 BuildKit Cache, Test 병렬화, 적절한 Runner 크기와 Artifact 보존 정책을 측정 기반으로 조정합니다.
**꼬리 질문:** 고정 가격을 답하지 않는 이유는 무엇인가요?
**꼬리 질문 답변:** Region, Runner와 실행 시간, 저장량과 전송량에 따라 달라지므로 사용 동인을 기준으로 추정해야 하기 때문입니다.
---
# 답변할 때 기억할 점
도구 이름만 나열하지 않고 실패 경계, Artifact 식별자와 롤백 가능성을 함께 설명한다.
정확한 가격이나 계정별 한도를 단정하지 않고 비용 동인과 확인 방법을 말한다.
배포 성공을 명령 완료가 아니라 사용자 경로와 Alarm 검증까지 포함해 설명한다.
