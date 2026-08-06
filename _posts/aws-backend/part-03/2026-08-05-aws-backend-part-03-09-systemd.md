---
title: "Chapter 09. Systemd"
permalink: /aws-backend/part-03/09-systemd/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Systemd
## Linux에서 서비스를 관리하는 기본 방식

> **학습 목표**
>
> - Systemd의 역할을 설명할 수 있다.
> - Spring Boot나 Docker 컨테이너를 서비스로 관리하는 이유를 이해한다.
> - 재시작 정책과 로그 확인 방식을 설명할 수 있다.

---

# Systemd란?

Systemd는 Linux에서 서비스 프로세스를 시작하고 관리하는 시스템이다.

EC2가 부팅될 때 애플리케이션을 자동으로 시작하거나, 프로세스가 죽었을 때 재시작하도록 설정할 수 있다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

---

# 왜 필요한가

터미널에서 직접 실행한 프로세스는 세션이 끊기면 함께 종료될 수 있다.

운영 서버에서는 애플리케이션을 서비스로 등록해 관리해야 한다.

Systemd는 다음을 제공한다.

- 부팅 시 자동 시작
- 프로세스 상태 확인
- 실패 시 재시작
- 로그 확인

---

# 기본 명령

```bash
sudo systemctl start my-api
sudo systemctl stop my-api
sudo systemctl restart my-api
sudo systemctl status my-api
journalctl -u my-api
```

---

# Docker와 Systemd

EC2에서 Docker를 직접 쓴다면 Docker 실행 명령을 Systemd 서비스로 감싸는 경우가 있다.

규모가 커지면 ECS 같은 컨테이너 오케스트레이션 서비스로 넘어가는 것이 자연스럽다.

---

# 기억해야 할 내용

- Systemd는 Linux 서비스 관리자다.
- 운영 애플리케이션은 수동 실행보다 서비스 등록이 안전하다.
- 로그와 재시작 정책을 함께 고려해야 한다.
- 컨테이너 운영이 커지면 ECS를 고려한다.


