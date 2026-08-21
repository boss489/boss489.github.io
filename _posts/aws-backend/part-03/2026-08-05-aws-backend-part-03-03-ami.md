---
title: "Chapter 03. AMI"
permalink: /aws-backend/part-03/03-ami/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. AMI
## EC2를 시작하기 위한 이미지

> **학습 목표**
>
> - AMI의 역할을 설명할 수 있다.
> - OS 이미지와 애플리케이션 이미지의 차이를 이해한다.
> - Golden Image 전략의 장단점을 설명할 수 있다.
> - AMI의 생성과 복사, 폐기 흐름을 이해한다.

---

# 왜 AMI가 필요한가

Spring Boot 서버 열 대를 만들 때마다 운영체제를 설치하고 보안 패치와 에이전트를 수동 설정하면 결과가 조금씩 달라진다.

서버마다 Java 버전이나 사용자 권한이 다르면 같은 애플리케이션도 일부 서버에서만 실패할 수 있다.

AMI는 검증된 시작 상태를 복제하여 EC2를 일관되게 만드는 기준점 역할을 한다.

장애가 난 서버를 직접 수리하는 대신 같은 AMI로 새 서버를 만들 수 있어야 복구와 확장이 빨라진다.

---

# AMI란?

AMI(Amazon Machine Image)는 EC2 인스턴스를 만들 때 사용하는 시작 이미지다.

AMI에는 보통 다음 정보가 포함된다.

- OS
- 기본 패키지
- 파일 시스템 구성
- 시작 설정

EC2는 AMI를 기반으로 루트 볼륨을 만들고 부팅한다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# AMI의 구성과 시작 흐름

AMI는 실행 중인 가상 머신 자체가 아니라 EC2를 시작하기 위한 템플릿이다.

AMI에는 루트 볼륨을 만들 Snapshot 매핑, 가상화 정보, 부팅 권한 같은 메타데이터가 포함된다.

```
AMI
├── Root Volume Snapshot
├── Block Device Mapping
└── Launch Permissions
          │
          ▼
    새 EBS Root Volume
          │
          ▼
      EC2 Boot
```

AMI에서 여러 인스턴스를 시작해도 각 인스턴스의 루트 EBS 볼륨은 독립적으로 생성된다.

AMI는 리전 리소스이므로 다른 리전에서 사용하려면 대상 리전으로 복사해야 한다.

---

# AMI 선택

가장 흔한 선택은 다음과 같다.

- Amazon Linux
- Ubuntu
- Red Hat Enterprise Linux
- Windows Server

백엔드 서버에서는 Amazon Linux 또는 Ubuntu를 자주 사용한다.

| 기준 | 확인할 내용 |
|---|---|
| 출처 | AWS, 신뢰할 수 있는 공급자, 조직 내부 이미지 |
| 아키텍처 | `x86_64` 또는 `arm64`와 인스턴스 타입 호환성 |
| 운영체제 | 패키지 관리자와 지원 기간 |
| 루트 장치 | EBS 기반 여부와 볼륨 설정 |
| 보안 | 최신 패치와 불필요한 서비스 제거 여부 |

Marketplace나 공개 AMI는 출처와 비용, 유지보수 주체를 확인해야 한다.

---

# Golden Image

Golden Image는 필요한 패키지와 보안 설정을 미리 반영한 표준 AMI다.

장점은 인스턴스 생성 후 설정 시간이 줄어든다는 것이다.

단점은 이미지 관리 프로세스가 필요하다는 것이다.

작은 팀에서는 처음부터 복잡한 Golden Image 체계를 만들기보다 User Data, Docker Image, IaC로 시작하는 편이 단순하다.

---

# 구성 전략 비교

| 전략 | 장점 | 단점 |
|---|---|---|
| 최소 OS AMI + User Data | 이미지 수가 적고 변경이 단순함 | 부팅 시간이 길고 외부 저장소에 의존 |
| Golden AMI | 시작이 빠르고 결과가 일관됨 | 패치마다 이미지 재생성 필요 |
| OS AMI + Docker Image | OS와 앱 생명주기를 분리함 | Docker 운영 체계가 필요 |

애플리케이션 변경이 잦다면 OS·보안 에이전트는 AMI에, 애플리케이션은 Docker Image에 두는 방식이 관리하기 쉽다.

AMI에 비밀번호, API Key, 데이터베이스 자격 증명을 넣으면 복사본 전체에 Secret이 확산되므로 금지해야 한다.

---

# AWS 콘솔과 CLI에서는

인스턴스에서 AMI를 생성하면 연결된 EBS의 Snapshot이 함께 만들어질 수 있다.

```bash
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "backend-base-2026-08-05"
```

소유한 AMI를 조회할 때는 `self` 필터로 공개 이미지와 구분한다.

```bash
aws ec2 describe-images \
  --owners self \
  --query "Images[].{Id:ImageId,Name:Name,State:State}"
```

AMI 등록을 해제해도 연결된 Snapshot이 자동으로 모두 삭제되는 것은 아니므로 폐기 절차에서 함께 확인해야 한다.

---

# 이미지 생성 파이프라인

운영에서는 사람이 서버를 수정한 뒤 이미지를 만드는 방식보다 코드로 이미지를 생성하고 검사하는 방식이 안전하다.

```
Base AMI
   │
Packer / Image Builder
   │
패치와 에이전트 설치
   │
보안·부팅 테스트
   │
Versioned AMI
   │
Launch Template 배포
```

새 AMI는 바로 전체 서버에 적용하지 않고 테스트 인스턴스나 일부 트래픽으로 검증한다.

이미지 이름과 태그에 생성일, OS 버전, 파이프라인 버전을 남기면 추적과 롤백이 쉬워진다.

---

# 실무에서는 어떻게 사용할까

Auto Scaling은 지정된 AMI와 Launch Template을 사용해 동일한 인스턴스를 반복 생성한다.

배포가 실패하면 이전 AMI나 이전 Docker Image를 가리키는 Launch Template 버전으로 되돌릴 수 있다.

오래된 AMI를 무조건 삭제하면 실행 중인 인스턴스에는 즉시 영향이 없더라도 장애 복구와 롤백이 어려워질 수 있다.

현재 배포와 이전 안정 버전을 보존하고 사용 관계를 확인한 뒤 수명 주기 정책으로 정리한다.

---

# 장애 사례와 주의할 점

| 증상 | 원인 | 대응 |
|---|---|---|
| 새 인스턴스가 부팅되지 않음 | 아키텍처·부트 설정 불일치 | 타입 호환성과 시스템 로그 확인 |
| 일부 서버만 설정이 다름 | 수동 변경으로 구성 Drift 발생 | 이미지 재생성과 인스턴스 교체 |
| 시작 시간이 너무 김 | User Data 작업 과다 | 공통 패키지를 AMI에 포함 |
| 새 이미지에 취약점 존재 | 오래된 Base AMI 사용 | 정기 재빌드와 취약점 검사 |
| AMI 삭제 후 비용 지속 | Snapshot 잔존 | 참조 확인 후 Snapshot 정리 |

실행 중인 서버를 임의 수정하는 Snowflake Server 방식은 다음 교체 때 변경이 사라져 장애를 반복하게 한다.

---

# 비용과 성능 고려사항

AMI 자체보다 AMI가 참조하는 EBS Snapshot 저장량이 주요 비용 요소이다.

리전 간 AMI 복사에는 Snapshot 복사와 데이터 전송이 관련되므로 필요한 리전에만 배포한다.

Golden Image는 부팅 시간을 줄일 수 있지만 이미지 생성·검증·보존 자동화 비용이 생긴다.

변경 빈도와 인스턴스 생성 빈도를 비교해 이미지에 포함할 범위를 정해야 한다.

---

# 기억해야 할 내용

- AMI는 EC2를 시작하기 위한 이미지다.
- AMI는 OS와 초기 파일 시스템 구성을 제공한다.
- Golden Image는 반복 설정을 줄이지만 관리 비용이 생긴다.
- 애플리케이션 배포 단위는 AMI보다 Docker Image가 더 자주 사용된다.
- AMI는 리전 리소스이며 다른 리전에서는 복사가 필요하다.
- Secret을 AMI에 포함하면 안 된다.
- AMI 폐기 시 연결된 Snapshot과 롤백 필요성을 함께 확인한다.

---

# 다음 Chapter

다음 Chapter에서는 [EBS](/aws-backend/part-03/04-ebs/)를 학습한다.

AMI로부터 생성된 루트 볼륨과 추가 데이터 볼륨이 EC2의 생명주기에서 어떻게 동작하는지 알아본다.


