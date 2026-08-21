---
title: "Part 3. Compute"
permalink: /aws-backend/part-03/
date: 2026-08-05T09:14:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 3. Compute
## 애플리케이션 실행 환경

> 목표
>
> Spring Boot 애플리케이션이 실행되는 AWS 컴퓨팅 환경을 이해한다.

---

# 학습 내용

- EC2
- AMI
- EBS
- ENI
- Linux
- SSH
- Docker
- Systemd

---

# 이 Part에서 연결할 실행 흐름

```text
AMI → EC2 생성 → EBS/ENI 연결 → Linux 부팅
    → Docker 또는 Systemd로 Spring Boot 실행 → ALB Health Check 응답
```

EC2는 단순히 서버 한 대를 만드는 서비스가 아니다. 어떤 이미지로 시작할지, 디스크를 어떻게 보존할지, 어느 네트워크에 붙일지, 프로세스가 죽었을 때 어떻게 다시 시작할지를 함께 결정해야 한다.

# 실무 판단 기준

- 서버 접속은 운영 편의보다 노출 범위를 먼저 고려한다. 가능하면 SSH 공개보다 Session Manager를 우선 검토한다.
- 애플리케이션 설정과 로그를 서버 파일에만 남기지 않는다. 재배포와 인스턴스 교체에도 동작해야 한다.
- 인스턴스 수와 배포 빈도가 늘면 EC2 직접 운영 대신 ECS를 검토할 시점이다.

---

# 완료 후 설명할 수 있어야 하는 것

- EC2 인스턴스가 생성되고 실행되는 흐름
- AMI, EBS, ENI의 역할
- SSH로 서버에 접속하는 방식
- Docker와 Systemd로 애플리케이션을 실행하는 방식

---

다음 Part

→ **Part 4. Request Flow**
