---
title: "Chapter 04. ECR"
permalink: /aws-backend/part-08/04-ecr/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 04. ECR
## Docker Image 저장소
> **학습 목표**
>
> - ECR의 역할을 설명할 수 있다.
> - Image Push/Pull 흐름을 이해한다.
> - Image Scan과 Lifecycle 정책을 설명할 수 있다.
> - ECR 인증과 IAM 권한의 관계를 설명할 수 있다.
> - ECS가 Private Subnet에서 Image를 Pull하는 경로를 설계할 수 있다.
---
# 왜 ECR이 필요한가
개발자 노트북에서 만든 Image를 파일로 압축해 운영 서버마다 전달한다면 배포 이력과 접근 권한을 일관되게 관리하기 어렵다.

ECR은 IAM 기반 인증과 정책을 제공하는 AWS 관리형 Container Registry로서 CI와 ECS 사이의 신뢰할 수 있는 Image 전달 지점이 된다.

CI에서 한 번 검증한 Image를 ECR에 저장하고 같은 Digest를 각 환경에 배포하면 서버마다 다시 빌드하는 차이를 없앨 수 있다.
---
# ECR이란?
ECR(Elastic Container Registry)은 OCI 호환 Container Image와 관련 Artifact를 Repository 단위로 저장하는 AWS 서비스이다.

ECR Private Registry는 계정과 Region을 기준으로 사용하며 Repository URI에는 Registry ID, Region, Repository 이름이 포함된다.
```text
123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api
└── Registry / Region                         └── Repository
```
Repository 안에는 여러 Image Manifest와 Layer가 저장되고 Tag는 특정 Manifest를 가리킨다.

ECR은 컨테이너를 실행하지 않으며 실행에 필요한 Image를 보관하고 배포하는 역할만 담당한다.
---
# Push와 Pull 동작 흐름
```text
Developer
   │ Git Push
   ▼
CI ── Docker Build
   │
   ├── IAM 인증
   ▼
ECR Repository
   │ Image Pull
   ▼
ECS Service
   └── Task Execution Role
       └── Task ── Spring Boot
                    ▲
                    │
              ALB Target Group
```
![ECS deployment](/assets/aws-backend/ecs-deployment.png)
1. CI가 테스트 후 Image를 빌드하고 고유 Tag를 부여한다.
2. AWS 자격 증명으로 ECR Authorization Token을 받아 Docker Client가 로그인한다.
3. Docker Client가 Image Manifest와 누락된 Layer를 ECR에 Push한다.
4. Task Definition이 ECR Image URI를 참조한다.
5. ECS가 Task Execution Role을 사용해 Image를 Pull한다.
6. Runtime이 Image를 Container로 실행하고 ALB Health Check를 기다린다.
ECR 로그인에 사용하는 Authorization Token은 영구 자격 증명이 아니며 만료되므로 CI 작업마다 새로 발급받는 방식이 안전하다.
---
# Repository와 Image 식별
| 항목 | 역할 | 운영 기준 |
|---|---|---|
| Registry | 계정과 Region의 저장 영역 | 계정 및 환경 분리 전략 적용 |
| Repository | 애플리케이션별 Image 모음 | 서비스 단위 이름 사용 |
| Tag | 사람이 읽는 Image 별칭 | Git SHA 또는 릴리스 버전 |
| Digest | 내용 기반 불변 식별자 | 배포 증적과 정확한 롤백 |
| Lifecycle Policy | 오래된 Image 자동 정리 | 최근 릴리스와 롤백 범위 보존 |
Digest로 배포하면 Tag가 변경되더라도 정확히 같은 Image를 가리킬 수 있다.
---
# 실습: Repository 생성과 Push
서울 Region에 Repository를 만들고 Image Scan과 Tag 불변성을 활성화하는 예시이다.
```bash
aws ecr create-repository \
  --repository-name order-api \
  --region ap-northeast-2 \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true
```
ECR 인증 토큰을 Docker에 전달한다.
```bash
aws ecr get-login-password --region ap-northeast-2 \
  | docker login \
    --username AWS \
    --password-stdin \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com
```
Image를 빌드하고 고유 Tag로 Push한다.
```bash
IMAGE_TAG=$(git rev-parse --short HEAD)
REGISTRY=123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

docker build -t order-api:${IMAGE_TAG} .
docker tag order-api:${IMAGE_TAG} ${REGISTRY}/order-api:${IMAGE_TAG}
docker push ${REGISTRY}/order-api:${IMAGE_TAG}

aws ecr describe-images \
  --repository-name order-api \
  --image-ids imageTag=${IMAGE_TAG}
```
CLI 예시의 계정 ID와 Region은 실제 배포 계정에 맞게 변경해야 한다.
---
# Image Scan과 Lifecycle Policy
Image Scan은 운영체제 패키지와 언어 패키지에서 알려진 취약점을 탐지하는 데 도움을 주지만 애플리케이션의 모든 보안 결함을 보장하지는 않는다.

심각도 기준으로 배포를 중단할지, 예외 승인할지, 언제 Base Image를 다시 빌드할지 조직 정책을 정해야 한다.

Lifecycle Policy는 규칙에 맞는 Image를 자동 만료시켜 저장소가 무한히 커지는 문제를 줄인다.
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "untagged image cleanup",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 14
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```
정리 기간은 예시이며 실제 롤백 보존 기간과 감사 요구를 기준으로 결정해야 한다.

현재 Task Definition이 참조하거나 긴급 롤백에 필요한 Image를 너무 일찍 삭제하지 않도록 정책 적용 전 Preview를 확인한다.
---
# IAM과 역할 구분
CI Role은 Repository에 Image를 Push할 권한이 필요하고 ECS Task Execution Role은 Image를 Pull할 권한이 필요하다.

애플리케이션 Container가 실행 중 S3나 SQS에 접근하는 권한은 Task Role에 부여하며 ECR Pull 권한과 분리한다.
| 주체 | 대표 책임 | 사용하는 역할 |
|---|---|---|
| CI | ECR 로그인, Layer와 Manifest Push | CI IAM Role |
| ECS Agent/Fargate | ECR Pull, 로그 전송, Secret 주입 | Task Execution Role |
| Spring Boot | S3, SQS, DynamoDB 등 호출 | Task Role |
권한은 Repository ARN과 필요한 작업으로 최소화하고 계정 간 Pull이 필요하면 Repository Policy와 호출 주체 IAM Policy를 함께 검토한다.
---
# Private Subnet에서의 Image Pull
Public IP가 없는 Task도 ECR API와 Registry Endpoint 및 실제 Layer 저장 경로에 도달할 네트워크가 있어야 한다.

일반적인 방법은 NAT Gateway를 통한 아웃바운드 연결 또는 ECR API, ECR DKR Interface Endpoint와 S3 Gateway Endpoint를 구성하는 것이다.
```text
Private Subnet Task
   │
   ├── NAT Gateway ──> ECR / S3
   │
   └── VPC Endpoints
       ├── ecr.api
       ├── ecr.dkr
       └── s3
```
---
# Spring Boot에서는 어떻게 쓰는가
Spring Boot Image는 환경별 설정을 포함하지 않은 채 ECR에 저장하고 Task Definition에서 Profile과 Secret을 주입한다.
```dockerfile
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /workspace
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /workspace/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```
```yaml
server:
  shutdown: graceful

management:
  endpoint:
    health:
      probes:
        enabled: true
```
Task Definition의 Image URI에는 `latest` 대신 CI가 Push한 Git SHA Tag 또는 Digest를 사용한다.

`SPRING_PROFILES_ACTIVE=prod`는 환경 변수로 주입하고, DB 비밀번호는 Secrets Manager 참조로 전달한다.
---
# 실무에서는 어떻게 사용할까
개발과 운영 계정을 분리한다면 검증된 Image를 복제하거나 승격하는 흐름을 만들고 각 단계의 Digest를 기록한다.

Repository 이름, Tag 규칙, Scan 실패 기준, 보존 기간을 조직 표준으로 정해 서비스마다 서로 다른 관행이 생기지 않게 한다.

Base Image 취약점이 발견되면 애플리케이션 코드 변경이 없어도 Image를 다시 빌드하고 새 Digest로 배포한다.

CI에는 장기 Access Key를 저장하기보다 OIDC 같은 임시 자격 증명 방식을 사용해 노출 위험을 줄인다.
---
# 장애 사례
| 증상 | 주요 원인 | 확인 지점 |
|---|---|---|
| `no basic auth credentials` | ECR 로그인 누락 또는 토큰 만료 | CI 로그인 단계와 Region |
| `CannotPullContainerError` | Execution Role 또는 네트워크 문제 | ECS Event, IAM, NAT/Endpoint |
| Image를 찾지 못함 | Repository, Tag, Region 불일치 | Task Definition Image URI |
| Tag Push 거부 | Tag Immutability 활성화 후 재사용 | 새 고유 Tag 발급 |
| 저장량 지속 증가 | Lifecycle Policy 미설정 | Untagged Image와 정책 Preview |
ECR 로그인 Token은 일정 시간이 지나면 만료되므로 로컬에서 과거 로그인 상태를 신뢰하지 말고 Push 직전에 다시 인증한다.
---
# 주의할 점
- `latest` Tag를 운영 배포의 유일한 식별자로 사용하지 않는다.
- Lifecycle Policy가 롤백 Image를 삭제하지 않도록 보존 범위를 정한다.
- Scan 결과를 확인하지 않은 채 기능만 활성화하고 끝내지 않는다.
- CI Push 권한과 ECS Pull 권한을 분리한다.
- Repository Region과 ECS Task가 참조하는 Region을 확인한다.
- Private Subnet의 ECR 및 S3 네트워크 경로를 검증한다.
---
# 비용과 성능 고려사항
ECR은 저장된 Image Layer 용량과 데이터 전송 등이 주요 비용 요소이므로 불필요한 Tag와 Untagged Image를 정리한다.

동일한 Layer는 전송과 저장에서 재사용될 수 있으므로 안정적인 Base Layer와 Spring Boot Layered JAR 구성이 효율적이다.

Private Subnet에서 NAT Gateway를 거쳐 Image를 Pull하면 NAT 처리와 데이터 전송 비용이 생길 수 있으며 VPC Endpoint도 자체 비용과 운영 요소가 있다.

Image 크기는 Task 시작과 Scale-out 속도에 영향을 주므로 최종 Image에서 Maven, JDK, 소스 코드를 제외한다.
---
# 기억해야 할 내용
- ECR은 IAM으로 보호되는 AWS의 Container Registry이다.
- Repository는 Image를 저장하고 ECS 실행 자체는 담당하지 않는다.
- Tag는 변경 가능한 별칭이고 Digest는 불변 식별자이다.
- CI Role은 Push하고 Task Execution Role은 Pull한다.
- Task Role은 실행 중인 애플리케이션의 AWS API 권한이다.
- Image Scan과 Lifecycle Policy는 보안과 저장량 관리에 필요하다.
- Private Subnet에서는 NAT 또는 적절한 VPC Endpoint 경로가 필요하다.
---
# 다음 Chapter
다음 Chapter에서는 [ECS Cluster](/aws-backend/part-08/05-ecs-cluster/)를 학습한다.

ECR의 Image가 실행될 논리적 공간을 만들고 Fargate와 EC2 Launch Type 중 적절한 Capacity 방식을 선택하는 기준을 살펴본다.
---
# 기억해야 할 내용
ECR은 컨테이너 배포의 이미지 저장소다.

오래된 Image는 Lifecycle 정책으로 정리한다.


