---
title: "Chapter 08. Docker"
permalink: /aws-backend/part-03/08-docker/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Docker
## 애플리케이션 실행 환경을 이미지로 포장하기

> **학습 목표**
>
> - Docker Image와 Container의 차이를 설명할 수 있다.
> - Spring Boot를 컨테이너로 실행하는 이유를 이해한다.
> - EC2에서 Docker를 사용할 때의 장단점을 설명할 수 있다.
> - 이미지 빌드와 컨테이너 실행 시 운영 원칙을 적용할 수 있다.

---

# Docker가 필요한 이유

서버마다 Java 버전, 환경 변수, 파일 경로가 다르면 배포가 불안정해진다.

Docker는 애플리케이션 실행 환경을 이미지로 묶어 같은 방식으로 실행하게 만든다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

개발 환경에서는 성공했지만 운영 서버에서는 Java 버전과 라이브러리가 달라 실패하는 문제를 줄일 수 있다.

Docker는 서버를 없애는 기술이 아니라 서버 위에서 프로세스를 격리하고 배포 단위를 표준화하는 기술이다.

---

# Docker의 구조

Docker Image는 읽기 전용 Layer로 구성되고 Container는 그 위에 쓰기 가능한 Layer와 격리된 프로세스를 추가한다.

```
EC2 Linux Kernel
└── Docker Engine
    ├── Image Layers
    │   ├── Base OS
    │   ├── JRE
    │   └── app.jar
    └── Container
        ├── Java Process
        ├── Writable Layer
        └── Network Namespace
```

컨테이너는 별도 가상 머신이 아니며 호스트 Linux 커널을 공유한다.

Image가 같아도 환경 변수와 Secret, 볼륨, 네트워크 설정이 다르면 컨테이너 동작은 달라질 수 있다.

---

# Image와 Container

Docker Image는 실행 가능한 템플릿이다.

Container는 Image를 실제로 실행한 프로세스다.

```
Docker Image
  -> docker run
  -> Container
```

| 구분 | Image | Container |
|---|---|---|
| 의미 | 실행 템플릿 | 실행 중이거나 중지된 인스턴스 |
| 변경 가능성 | 내용 기반의 불변 산출물 | Writable Layer에 변경 가능 |
| 식별 | 태그와 Digest | 이름과 Container ID |
| 저장 위치 | Registry에 배포 | Docker Host에서 실행 |

운영 배포는 변경 가능한 `latest` 태그만 신뢰하지 않고 버전 태그나 Digest로 정확한 이미지를 식별해야 한다.

---

# Spring Boot 실행

예시는 다음과 같다.

```bash
docker run -p 8080:8080 my-api:latest
```

EC2에서 Docker를 쓰면 서버에 직접 Java와 애플리케이션 파일을 복잡하게 설치하지 않아도 된다.

---

# Spring Boot 이미지 만들기

멀티 스테이지 빌드는 빌드 도구와 소스 코드를 최종 실행 이미지에서 제외할 수 있다.

```dockerfile
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /workspace
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:17-jre
RUN useradd --system --uid 10001 spring
WORKDIR /app
COPY --from=builder /workspace/target/*.jar app.jar
USER spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Base Image는 신뢰할 수 있는 출처와 지원되는 Java 17 버전을 사용하고 정기적으로 재빌드해 보안 패치를 반영한다.

Secret을 `Dockerfile`의 `ENV`나 Image Layer에 복사하면 이미지 기록에 남으므로 실행 시 외부에서 주입한다.

---

# 컨테이너 실행과 네트워크

호스트의 `8080` 포트를 컨테이너의 `8080` 포트로 전달한다.

```bash
docker run -d \
  --name my-api \
  --restart unless-stopped \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  my-api:2026.08.05
```

`EXPOSE`는 문서화 성격의 메타데이터이며 실제 호스트 포트 공개는 `-p`로 설정한다.

Spring Boot가 컨테이너 내부의 `127.0.0.1`에만 바인딩되면 포트 매핑이 있어도 외부에서 접근할 수 없다.

---

# 데이터와 로그

컨테이너의 Writable Layer는 컨테이너 삭제와 함께 사라질 수 있으므로 영속 데이터 저장소로 사용하지 않는다.

| 데이터 | 권장 위치 |
|---|---|
| 관계형 업무 데이터 | RDS |
| 업로드 파일 | S3 |
| 공유 세션·캐시 | Redis |
| 애플리케이션 로그 | 표준 출력 후 중앙 수집 |
| 임시 파일 | 크기 제한과 정리 정책을 둔 임시 영역 |

애플리케이션 로그는 파일보다 표준 출력과 표준 오류로 보내면 `docker logs`와 수집 에이전트가 처리하기 쉽다.

---

# 주의할 점

Docker만 사용한다고 운영이 완성되는 것은 아니다.

다음도 함께 필요하다.

- 로그 수집
- 컨테이너 재시작 정책
- 이미지 버전 관리
- 환경 변수와 Secret 관리
- Health Check

---

# 장애 사례

| 증상 | 원인 후보 | 확인 방법 |
|---|---|---|
| 시작 직후 `Exited` | ENTRYPOINT·환경 변수 오류 | `docker logs`, `docker inspect` |
| 외부 접속 실패 | 포트 매핑·바인딩·SG 오류 | `docker ps`, `ss` |
| 반복 재시작 | 앱 종료와 Restart Policy | 종료 코드와 로그 |
| 디스크 가득 참 | 이미지·로그 누적 | `docker system df` |
| 호스트 OOM | 메모리 제한 부재 | 컨테이너와 커널 지표 |

재시작 정책은 일시 장애를 복구할 수 있지만 설정 오류로 죽는 컨테이너를 무한 반복하게 만들 수도 있다.

Health Check는 프로세스 존재가 아니라 실제 요청 처리 가능 여부를 검사하도록 설계한다.

---

# 실무에서는 어떻게 사용할까

CI에서 테스트 후 이미지를 한 번 빌드하고 Registry에 Push한 뒤 개발·스테이징·운영에서 같은 Digest를 사용한다.

```
Git Commit
   │
Test & Build
   │
Docker Image
   │
Registry
   │
EC2 / ECS
```

EC2 한두 대에서는 Docker와 Systemd로 관리할 수 있지만 서버 수와 배포 빈도가 늘면 ECS 같은 오케스트레이션이 유리하다.

컨테이너를 직접 수정하지 않고 새 이미지를 빌드해 교체해야 재현성과 롤백 가능성을 유지할 수 있다.

---

# 비용과 성능 고려사항

Docker 자체보다 기반 EC2, EBS, Registry 저장량과 데이터 전송이 비용 요소이다.

작은 컨테이너는 Pull 시간과 공격 표면을 줄이지만 진단 도구가 부족할 수 있으므로 운영 방식에 맞는 Base Image를 선택한다.

CPU와 메모리 제한이 없으면 하나의 컨테이너가 호스트 전체 자원을 고갈시킬 수 있다.

---

# 기억해야 할 내용

- Image는 실행 템플릿이고 Container는 실행 중인 프로세스다.
- Docker는 실행 환경 차이를 줄인다.
- EC2에서 Docker를 쓰면 배포 단위가 단순해진다.
- 운영에는 로그, 재시작, Secret 관리가 함께 필요하다.
- 컨테이너는 호스트 Linux 커널을 공유한다.
- 운영 이미지는 버전 태그나 Digest로 식별한다.
- 영속 데이터와 Secret을 Image나 Writable Layer에 넣지 않는다.

---

# 다음 Chapter

다음 Chapter에서는 [Systemd](/aws-backend/part-03/09-systemd/)를 학습한다.

JAR 프로세스나 Docker 컨테이너를 부팅 시 자동 시작하고 실패 시 관리하는 Linux 서비스 방식을 알아본다.


