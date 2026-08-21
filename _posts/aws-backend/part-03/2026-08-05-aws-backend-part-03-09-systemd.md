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
> - 안전한 Unit 파일을 작성하고 장애를 진단할 수 있다.

---

# Systemd란?

Systemd는 Linux에서 서비스 프로세스를 시작하고 관리하는 시스템이다.

EC2가 부팅될 때 애플리케이션을 자동으로 시작하거나, 프로세스가 죽었을 때 재시작하도록 설정할 수 있다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

---

# 왜 필요한가

터미널에서 직접 실행한 프로세스는 세션이 끊기면 함께 종료될 수 있다.

운영 서버에서는 애플리케이션을 서비스로 등록해 관리해야 한다.

쇼핑몰 API를 `java -jar`로 수동 실행하면 서버 재부팅 뒤 자동으로 복구되지 않고 실행 환경도 사람마다 달라질 수 있다.

Systemd는 실행 방법을 Unit 파일로 선언하여 시작, 종료, 재시작과 로그 확인을 표준화한다.

Systemd는 다음을 제공한다.

- 부팅 시 자동 시작
- 프로세스 상태 확인
- 실패 시 재시작
- 로그 확인

---

# Systemd의 구조와 동작

Systemd는 Unit 사이의 의존 관계를 계산하고 서비스 프로세스를 지정된 사용자로 실행한다.

```
Linux Boot
   │
systemd
   ├── network-online.target
   │          │
   │          ▼
   └── my-api.service
          ├── Environment
          ├── ExecStart
          ├── Restart Policy
          └── journald
```

Unit 파일을 `enable`하면 부팅 Target에 연결되고 `start`하면 현재 세션에서 즉시 실행된다.

`enable`과 `start`는 다른 작업이므로 하나만 수행하면 기대한 시점에 서비스가 실행되지 않을 수 있다.

---

# Spring Boot Unit 파일

운영 애플리케이션은 `root`가 아닌 전용 사용자로 실행한다.

```ini
[Unit]
Description=Spring Boot Backend API
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=my-api
Group=my-api
WorkingDirectory=/opt/my-api
EnvironmentFile=-/etc/my-api/my-api.env
ExecStart=/usr/bin/java -XX:MaxRAMPercentage=75.0 -jar /opt/my-api/app.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

`ExecStart`에는 셸 문법이 자동 적용되지 않으므로 실행 파일의 절대 경로를 사용한다.

`EnvironmentFile`의 Secret은 파일 권한을 제한하고 더 큰 규모에서는 전용 Secret 관리 서비스를 검토한다.

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

# Unit 적용과 확인

Unit 파일을 수정한 뒤에는 Systemd 구성을 다시 읽고 문법을 검증한다.

```bash
sudo systemd-analyze verify /etc/systemd/system/my-api.service
sudo systemctl daemon-reload
sudo systemctl enable --now my-api
```

상태 출력의 첫 줄뿐 아니라 종료 코드와 최근 Journal을 함께 확인해야 한다.

```bash
systemctl show my-api -p ActiveState -p SubState -p ExecMainStatus
journalctl -u my-api --since "30 minutes ago" --no-pager
```

---

# 시작과 종료 흐름

Systemd는 `ExecStart`로 프로세스를 시작하고 Main PID의 상태를 추적한다.

서비스 중지 시 종료 신호를 보내고 `TimeoutStopSec` 안에 종료되지 않으면 강제 종료할 수 있다.

Spring Boot의 Graceful Shutdown 시간을 Systemd 종료 제한과 ALB Deregistration 흐름에 맞춰야 처리 중인 요청이 끊기지 않는다.

`Restart=always`는 정상 종료도 재시작하고 `Restart=on-failure`는 실패 종료를 중심으로 재시작한다.

설정 오류가 지속되면 재시작이 반복되므로 `RestartSec`과 시작 제한을 함께 고려한다.

---

# Docker 서비스 관리

Systemd가 Docker CLI를 호출한다면 기존 컨테이너 정리와 종료 동작을 명확하게 정의해야 한다.

Docker Restart Policy와 Systemd 재시작 정책을 동시에 복잡하게 설정하면 실제 재시작 주체를 파악하기 어렵다.

한쪽에 책임을 두고 이미지 Pull, 컨테이너 교체, 로그 정리를 배포 절차에 포함한다.

---

# 로그와 journald

표준 출력과 표준 오류는 기본적으로 Journal에 수집할 수 있다.

```bash
journalctl -u my-api -f
journalctl -u my-api -p err
journalctl -u my-api --since today
```

Journal은 로컬 분석에 유용하지만 인스턴스가 교체되면 함께 사라질 수 있다.

CloudWatch Agent 등으로 중앙 로그 시스템에 전달하고 보존 기간과 디스크 사용 제한을 설정한다.

---

# 장애 사례

| 증상 | 원인 후보 | 확인 방법 |
|---|---|---|
| `203/EXEC` | 경로·실행 권한 오류 | `ExecStart`, `ls -l` |
| `status=1/FAILURE` | 애플리케이션 시작 실패 | Journal과 앱 로그 |
| 재부팅 후 미실행 | `enable` 누락 | `systemctl is-enabled` |
| 환경 변수가 없음 | 파일 경로·권한 오류 | `systemctl show` |
| Start request repeated | 반복 실패로 시작 제한 | 최초 실패 로그 |
| 중지 중 강제 종료 | 앱 종료가 제한보다 느림 | 종료 시간과 신호 |

서비스를 무조건 재시작하기 전에 `systemctl status`와 Journal로 최초 실패 원인을 보존해야 한다.

Shell에서는 되지만 Systemd에서는 실패한다면 작업 디렉터리, PATH, 사용자, 환경 변수 차이를 확인한다.

---

# 실무에서는 어떻게 사용할까

Unit 파일을 서버에서 직접 편집하지 않고 배포 저장소나 이미지 생성 코드로 버전 관리한다.

배포는 새 JAR 배치, 파일 소유권 확인, 서비스 재시작, Health Check 검증 순서로 자동화한다.

인스턴스 부팅 완료와 애플리케이션 준비 완료는 다르므로 ALB는 Health Check 성공 뒤 트래픽을 전달해야 한다.

EC2 수가 늘어나면 개별 Systemd 관리보다 ECS Service가 원하는 실행 개수를 유지하게 하는 편이 효율적이다.

---

# 비용과 성능 고려사항

Systemd 자체의 별도 AWS 비용은 없지만 재시작 루프는 CPU, 로그, 외부 API 호출을 증가시킬 수 있다.

상세 로그를 무기한 로컬에 저장하면 EBS 용량이 증가하므로 보존 정책을 둔다.

서비스 시작 시간이 길면 Auto Scaling 교체와 배포도 느려지므로 초기화 작업을 최소화한다.

---

# 기억해야 할 내용

- Systemd는 Linux 서비스 관리자다.
- 운영 애플리케이션은 수동 실행보다 서비스 등록이 안전하다.
- 로그와 재시작 정책을 함께 고려해야 한다.
- 컨테이너 운영이 커지면 ECS를 고려한다.
- `enable`은 부팅 자동 시작이고 `start`는 현재 실행이다.
- Unit은 전용 사용자, 절대 경로, 명확한 환경 설정으로 작성한다.
- 재시작 전에 최초 실패의 종료 코드와 Journal을 확인한다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 10. Part 3 Summary**이다.

EC2, AMI, EBS, ENI와 Linux, SSH, Docker, Systemd가 하나의 Spring Boot 실행 환경을 구성하는 흐름을 연결해 정리한다.


