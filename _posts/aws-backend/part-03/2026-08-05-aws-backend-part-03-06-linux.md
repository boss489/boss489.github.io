---
title: "Chapter 06. Linux"
permalink: /aws-backend/part-03/06-linux/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Linux
## EC2 위에서 애플리케이션을 실행하는 OS

> **학습 목표**
>
> - Linux가 EC2 운영에서 왜 중요한지 이해한다.
> - 프로세스, 포트, 로그, 파일 권한의 기본 개념을 설명할 수 있다.
> - 서버 장애 확인 시 기본 명령을 사용할 수 있다.

---

# 왜 Linux를 알아야 하는가

Spring Boot를 EC2에 직접 배포하면 결국 Linux 위에서 프로세스로 실행된다.

Docker를 사용해도 컨테이너는 Linux 커널 기능 위에서 동작한다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

---

# 기본 확인 항목

서버 장애를 볼 때는 먼저 다음을 확인한다.

```bash
ps -ef
df -h
free -m
top
ss -lntp
```

각 명령은 프로세스, 디스크, 메모리, CPU, 포트 상태를 확인하는 데 사용한다.

---

# 파일 권한

Linux 파일은 소유자와 권한을 가진다.

애플리케이션이 로그 파일을 쓰지 못하거나 실행 파일을 실행하지 못하면 권한 문제일 수 있다.

---

# 로그

EC2에서 문제를 볼 때 로그는 가장 중요하다.

확인 대상은 다음과 같다.

- 애플리케이션 로그
- Systemd journal
- Nginx 로그
- Docker 로그
- CloudWatch Logs

---

# 기억해야 할 내용

- EC2 운영은 Linux 운영과 연결된다.
- 프로세스, 포트, 디스크, 메모리 확인이 기본이다.
- Docker를 써도 Linux 이해가 필요하다.
- 운영 로그는 CloudWatch로 모으는 것이 좋다.


