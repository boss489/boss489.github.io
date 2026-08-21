---
title: "Chapter 02. Docker Image"
permalink: /aws-backend/part-08/02-docker-image/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 02. Docker Image
## 실행 가능한 애플리케이션 패키지
> **학습 목표**
>
> - Docker Image의 역할을 설명할 수 있다.
> - Image Tag가 배포에 미치는 영향을 이해한다.
> - Spring Boot Image를 만들 때 고려할 점을 설명할 수 있다.
> - 멀티 스테이지 빌드와 Layer Cache를 활용할 수 있다.
> - Tag와 Digest의 차이를 설명할 수 있다.
---
# 왜 Docker Image가 필요한가
개발자가 빌드한 JAR를 EC2에 직접 복사했는데 운영 서버의 Java 버전이 달라 `UnsupportedClassVersionError`가 발생하는 상황을 생각해 보자.

서버에서 직접 JAR를 교체하면 이전 파일이 덮어써져 어떤 바이너리가 배포되었는지 증명하기 어렵고, 롤백할 때도 과거 파일을 별도로 보관해야 한다.

Docker Image는 애플리케이션, Java Runtime, 실행 명령을 하나의 버전 있는 산출물로 묶어 환경 차이와 수동 배포를 줄인다.

CI에서 검증한 Image Digest를 그대로 운영에 사용하면 같은 코드가 아니라 **같은 바이트의 산출물**을 배포했다는 사실을 확인할 수 있다.
---
# Docker Image란?
Docker Image는 애플리케이션 실행에 필요한 파일과 메타데이터를 Layer로 구성한 읽기 전용 템플릿이다.

Image 자체는 실행 중인 프로세스가 아니며 Runtime이 Image 위에 쓰기 가능한 Layer를 추가해 Container를 만든다.

각 Layer는 내용 기반으로 식별되므로 변경되지 않은 Layer는 로컬 빌드와 Registry 전송에서 재사용될 수 있다.
```text
Image
├── Base Layer: eclipse-temurin:17-jre
├── Dependencies Layer
├── Spring Boot Loader Layer
├── Application Layer
└── Metadata: ENTRYPOINT, ENV, EXPOSE
        │
        └── docker run
              ▼
          Container
```
[Part 3의 Docker Chapter](/aws-backend/part-03/08-docker/)가 Image와 Container의 기본 차이를 다뤘다면, 여기서는 ECS 배포를 위한 Image 설계와 식별에 집중한다.
---
# Layer와 빌드 캐시
Dockerfile의 각 명령은 대체로 새로운 Layer를 만들며 앞 단계가 바뀌면 이후 단계의 캐시도 무효화될 수 있다.

소스 코드보다 변경 빈도가 낮은 의존성 정보를 먼저 복사하면 Maven 의존성 다운로드 Layer를 재사용하기 쉽다.
| 대상 | 변경 빈도 | 권장 Layer 위치 |
|---|---:|---|
| Base JRE | 낮음 | 가장 아래 |
| Maven 의존성 | 낮음 | 애플리케이션 코드보다 아래 |
| Spring Boot Loader | 낮음 | 의존성 다음 |
| 애플리케이션 클래스 | 높음 | 가장 위 |
Layer 수를 무조건 줄이는 것보다 변경 특성에 맞게 분리해 Build와 Pull 효율을 높이는 것이 중요하다.

`COPY . .`를 너무 이른 단계에 두면 README 한 줄 변경도 Maven 의존성 캐시를 무효화할 수 있다.
---
# Tag와 Digest
Tag는 사람이 읽기 쉬운 Image 별칭이고 Digest는 Image Manifest 내용에서 계산되는 불변 식별자이다.
| 구분 | Tag | Digest |
|---|---|---|
| 예 | `order-api:2026.08.05` | `order-api@sha256:...` |
| 변경 가능성 | 같은 이름을 다른 Image에 다시 지정 가능 | 내용이 같으면 동일 |
| 목적 | 릴리스 검색과 관리 | 정확한 산출물 고정 |
| 운영 권장 | Git SHA 또는 릴리스 버전 | 배포 증적과 승격에 활용 |
`latest`는 특별한 최신 버전 선택 기능이 아니라 Tag가 생략될 때 사용되는 관례적인 이름일 뿐이다.

같은 `latest`를 계속 덮어쓰면 이미 실행 중인 Task와 새 Task가 서로 다른 Image Digest를 사용할 수 있어 배포 상태가 불명확해진다.

ECR Repository에서 Tag 불변성을 활성화하면 기존 Tag를 다른 Image에 덮어쓰는 실수를 줄일 수 있다.
---
# Image 빌드 흐름
```text
Developer
   │ Source Commit
   ▼
CI Test
   │
   ▼
Docker Build ──> Image Tag + Digest
   │
   ▼
ECR Push
   │
   ▼
Task Definition Revision
   │
   ▼
ECS Service ──> Task ──> ALB Target Group
```
1. CI는 같은 Commit에서 테스트와 패키징을 수행한다.
2. Docker Build는 Dockerfile과 Build Context로 Image를 생성한다.
3. Image에 Git SHA나 릴리스 버전 Tag를 부여한다.
4. [ECR](/aws-backend/part-08/04-ecr/)에 Push한 뒤 Digest를 기록한다.
5. 새 Task Definition Revision이 해당 Image를 참조한다.
6. ECS Service가 새 Revision으로 Task를 교체한다.
---
# 실습: Image 빌드와 확인
`.dockerignore`에는 빌드에 필요 없는 파일을 제외해 Build Context 크기와 Secret 유출 위험을 줄인다.
```bash
docker build \
  --tag order-api:$(git rev-parse --short HEAD) \
  .

docker image ls order-api
docker image inspect order-api:$(git rev-parse --short HEAD)
docker history order-api:$(git rev-parse --short HEAD)
```
플랫폼이 다른 개발 머신에서 Fargate Linux Task용 Image를 만들 때는 대상 아키텍처를 명시할 수 있다.
```bash
docker buildx build \
  --platform linux/amd64 \
  --tag order-api:2026.08.05 \
  --load \
  .
```
운영 Capacity가 ARM 기반이면 `linux/arm64` Image를 빌드하고 Task Definition의 Runtime Platform도 일치시켜야 한다.
---
# Spring Boot에서는 어떻게 쓰는가
멀티 스테이지 빌드는 Maven과 JDK가 필요한 빌드 단계와 JRE만 필요한 실행 단계를 분리한다.

Spring Boot Layered JAR는 의존성과 애플리케이션 코드를 서로 다른 Image Layer로 복사해 변경량을 줄인다.
```dockerfile
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /workspace
COPY . .
RUN ./mvnw -q -DskipTests package
RUN java -Djarmode=layertools \
    -jar target/*.jar extract --destination extracted

FROM eclipse-temurin:17-jre
RUN useradd --system --uid 10001 spring
WORKDIR /app
COPY --from=builder /workspace/extracted/dependencies/ ./
COPY --from=builder /workspace/extracted/spring-boot-loader/ ./
COPY --from=builder /workspace/extracted/snapshot-dependencies/ ./
COPY --from=builder /workspace/extracted/application/ ./
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```
Spring Boot와 Maven Plugin 버전에 따라 `layertools` 실행 방식과 추출 결과가 달라질 수 있으므로 현재 프로젝트의 Plugin 문서를 확인해야 한다.

Image에는 운영 비밀번호를 넣지 않고 Profile과 Secret은 ECS Task 실행 시 주입한다.
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

management:
  endpoint:
    health:
      probes:
        enabled: true
```
`SPRING_PROFILES_ACTIVE=prod`는 Task Definition의 `environment`에 두고 데이터베이스 비밀번호는 `secrets`로 참조한다.
---
# 실무에서는 어떻게 사용할까
Image Tag에는 Git SHA를 포함하고 배포 기록에는 ECR Digest와 Task Definition Revision을 함께 남긴다.

Base Image의 보안 패치는 기존 Image에 자동 반영되지 않으므로 소스가 변하지 않아도 주기적으로 다시 빌드하고 검증해야 한다.

Root 사용자를 피하고 쓰기 가능한 디렉터리를 최소화하며 Image에 셸과 패키지 관리자를 남길지는 진단 편의성과 공격 표면을 비교해 결정한다.

Image는 환경 중립적으로 만들고 `dev`, `stage`, `prod`의 차이는 환경 변수, Secret, 외부 설정으로 표현한다.
---
# 장애 사례
| 증상 | 원인 | 대응 |
|---|---|---|
| ECS에서 `exec format error` | CPU 아키텍처 불일치 | Build Platform과 Runtime Platform 일치 |
| 시작 시 Java 버전 오류 | 빌드 JDK와 실행 JRE 불일치 | Java 17 계열 조합 검증 |
| Image가 지나치게 큼 | JDK와 빌드 도구가 최종 Stage에 포함 | 멀티 스테이지 빌드 적용 |
| 새 Task에 과거 코드 실행 | 재사용한 Tag 또는 Pull 결과 혼동 | 고유 Tag와 Digest 사용 |
| Image에 인증 정보 노출 | `COPY` 또는 `ENV`로 Secret 포함 | 이력 폐기 후 Secret 교체 |
Image Layer에서 파일을 나중에 삭제해도 이전 Layer에는 내용이 남을 수 있으므로 Secret이 포함된 Image는 다시 빌드하는 것만으로 충분하지 않다.
---
# 주의할 점
- `latest`를 운영 배포 식별자로 사용하지 않는다.
- Base Image는 신뢰할 수 있는 출처와 지원되는 Java 버전을 선택한다.
- 최종 Stage에는 소스 코드와 Maven Cache를 포함하지 않는다.
- 컨테이너를 Root가 아닌 사용자로 실행한다.
- `ENTRYPOINT`는 신호 전달과 종료 동작을 검증한다.
- Image 아키텍처와 ECS Runtime Platform을 일치시킨다.
---
# 비용과 성능 고려사항
Image가 크면 ECR 저장량과 전송량이 늘고 Task 시작 시 Pull 시간이 길어진다.

Layer Cache는 CI Build 시간을 줄이지만 Cache 저장소의 보존 비용과 오래된 의존성 재사용 위험도 관리해야 한다.

Fargate의 vCPU와 메모리 과금은 Image 크기로 직접 결정되지 않지만 큰 Image로 인한 시작 지연은 Scale-out 대응 시간에 영향을 준다.

Private Subnet에서 NAT Gateway를 통해 Image를 Pull하면 NAT 처리와 데이터 전송이 비용 요소가 되므로 ECR과 S3 VPC Endpoint를 검토한다.
---
# 기억해야 할 내용
- Docker Image는 읽기 전용 Layer와 실행 메타데이터로 구성된다.
- Image는 Container가 아니라 Container를 만드는 불변 템플릿이다.
- Tag는 변경 가능한 별칭이고 Digest는 내용 기반 식별자이다.
- 멀티 스테이지 빌드는 빌드 도구를 최종 Image에서 제외한다.
- Layered JAR는 변경이 잦은 애플리케이션 Layer를 분리한다.
- Secret은 Image가 아니라 실행 시점에 주입한다.
- 운영 배포는 Git SHA, Digest, Task Definition Revision으로 추적한다.
---
# 다음 Chapter
다음 Chapter에서는 [Container Runtime](/aws-backend/part-08/03-container-runtime/)을 학습한다.

Image가 격리된 프로세스로 실행될 때 포트, 환경 변수, 로그, 자원 제한과 종료 신호가 어떻게 적용되는지 살펴본다.
---
# Tag
Tag는 Image 버전을 식별한다.

`latest`만 사용하면 어떤 코드가 배포됐는지 추적하기 어렵다.

실무에서는 Git SHA, 빌드 번호, 릴리스 버전을 Tag로 사용한다.
---
# Spring Boot Image
Spring Boot Image에는 다음이 포함된다.
- JRE 또는 JDK
- 애플리케이션 JAR
- 실행 명령
- 환경 변수 기본값
---
# 기억해야 할 내용
Image는 배포 단위다.

운영 배포에는 추적 가능한 Tag를 사용해야 한다.


