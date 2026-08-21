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
> - CPU, 메모리, 디스크 문제를 구분해 점검할 수 있다.

---

# 왜 Linux를 알아야 하는가

Spring Boot를 EC2에 직접 배포하면 결국 Linux 위에서 프로세스로 실행된다.

Docker를 사용해도 컨테이너는 Linux 커널 기능 위에서 동작한다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

쇼핑몰 API의 응답이 느릴 때 Java 코드만 보면 CPU 고갈, 메모리 부족, 디스크 가득 참 같은 운영체제 문제를 놓칠 수 있다.

Linux 기본 지식은 장애 원인을 애플리케이션과 실행 환경으로 나누는 출발점이다.

---

# Linux 실행 구조

Linux 커널은 CPU 스케줄링, 메모리, 파일 시스템, 네트워크 장치를 관리한다.

```
EC2
└── Linux Kernel
    ├── systemd
    │   └── java -jar app.jar
    ├── File System
    │   ├── /opt/my-api
    │   └── /var/log
    └── Network
        └── TCP :8080
```

Spring Boot는 PID를 가진 프로세스이며 파일을 열고 메모리를 사용하며 TCP 포트를 Listen한다.

프로세스가 실행 중이라는 사실만으로 요청을 정상 처리한다는 뜻은 아니므로 자원과 포트, 로그를 함께 확인해야 한다.

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

# 프로세스와 포트

프로세스는 실행 중인 프로그램의 인스턴스이며 PID로 식별한다.

```bash
pgrep -af java
ps -o pid,ppid,user,%cpu,%mem,cmd -p 1234
sudo ss -lntp
curl -v http://127.0.0.1:8080/actuator/health
```

`ss`에 `8080`이 없다면 Security Group보다 먼저 애플리케이션 시작과 바인딩 설정을 확인한다.

Loopback 요청은 성공하지만 외부 요청이 실패하면 ENI, Route Table, Security Group 계층을 점검한다.

`kill -9`는 종료 정리 작업을 건너뛰므로 먼저 Systemd나 정상 종료 신호를 사용한다.

---

# CPU와 메모리

| 자원 | 대표 증상 | 확인 항목 |
|---|---|---|
| CPU | 응답 지연, Load Average 상승 | `top`, 프로세스 CPU, 스레드 |
| 메모리 | OOM, Swap 증가, 강제 종료 | `free`, kernel log, JVM heap |
| 디스크 | 쓰기 실패, 로그 중단 | `df`, inode, I/O 대기 |
| 네트워크 | 연결 지연·실패 | `ss`, DNS, 패킷 경로 |

Linux의 여유 메모리가 작아 보여도 Page Cache로 사용 중일 수 있으므로 `available` 값을 함께 본다.

JVM Heap뿐 아니라 Metaspace, Direct Memory, 스레드 스택과 OS 메모리도 인스턴스 용량에 포함된다.

OOM Killer가 Java 프로세스를 종료했는지는 커널 로그에서 확인할 수 있다.

```bash
journalctl -k --since "30 minutes ago"
```

---

# 파일 권한

Linux 파일은 소유자와 권한을 가진다.

애플리케이션이 로그 파일을 쓰지 못하거나 실행 파일을 실행하지 못하면 권한 문제일 수 있다.

권한은 소유자, 그룹, 기타 사용자에 대한 읽기·쓰기·실행 비트로 구성된다.

```bash
ls -l /opt/my-api
sudo chown -R my-api:my-api /opt/my-api
sudo chmod 750 /opt/my-api
```

문제를 빠르게 피하려고 `chmod 777`을 사용하면 모든 사용자가 수정할 수 있어 보안 위험이 커진다.

애플리케이션은 `root`가 아닌 전용 사용자로 실행하고 필요한 디렉터리에만 최소 권한을 부여한다.

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

# 로그를 읽는 방법

먼저 장애가 시작된 시간을 정하고 해당 시간 전후의 로그를 좁혀 본다.

```bash
journalctl -u my-api --since "2026-08-21 09:00:00"
journalctl -u my-api -f
sudo du -xhd1 /var/log
```

로그 파일이 삭제됐는데 디스크 공간이 돌아오지 않으면 프로세스가 삭제된 파일을 계속 열고 있는지 확인한다.

```bash
sudo lsof +L1
```

운영 로그는 인스턴스 교체 뒤에도 조회할 수 있도록 CloudWatch Logs 같은 중앙 시스템으로 전송한다.

---

# 패키지와 업데이트

Amazon Linux와 Ubuntu는 패키지 관리 명령이 다르므로 운영체제를 확인한 뒤 작업한다.

```bash
cat /etc/os-release
uname -r
```

보안 업데이트는 테스트 환경에서 애플리케이션 호환성을 검증한 뒤 이미지 재생성과 인스턴스 교체로 반영하는 것이 안전하다.

운영 서버마다 직접 명령을 실행하면 구성 차이가 누적되므로 변경 과정을 스크립트나 이미지 파이프라인으로 관리한다.

---

# 실무 장애 점검 순서

1. 장애 시간과 영향 범위를 확인한다.
2. 인스턴스와 프로세스가 실행 중인지 확인한다.
3. 포트 Listen과 로컬 Health Check를 확인한다.
4. CPU, 메모리, 디스크와 inode를 확인한다.
5. 애플리케이션과 Systemd, 커널 로그를 연결해 본다.
6. OS가 정상이라면 VPC 경로와 외부 의존성을 확인한다.

명령 결과를 기록하지 않고 바로 재시작하면 원인 증거가 사라질 수 있으므로 긴급성이 허용하는 범위에서 상태를 먼저 수집한다.

---

# 주의할 점

서버의 시간대가 서로 다르면 분산 로그의 사건 순서를 맞추기 어렵기 때문에 UTC 또는 조직 표준을 정한다.

디스크 사용률 알람과 로그 순환을 구성하지 않으면 무제한 로그가 루트 볼륨을 채울 수 있다.

SSH에서 수동으로 수정한 설정은 다음 인스턴스 교체 때 사라지므로 반드시 코드와 배포 절차에 반영한다.

---

# 기억해야 할 내용

- EC2 운영은 Linux 운영과 연결된다.
- 프로세스, 포트, 디스크, 메모리 확인이 기본이다.
- Docker를 써도 Linux 이해가 필요하다.
- 운영 로그는 CloudWatch로 모으는 것이 좋다.
- JVM 메모리와 OS 메모리를 함께 고려해야 한다.
- 애플리케이션은 전용 사용자와 최소 파일 권한으로 실행한다.
- 재시작 전에 가능한 범위에서 장애 증거를 수집한다.

---

# 다음 Chapter

다음 Chapter에서는 [SSH](/aws-backend/part-03/07-ssh/)를 학습한다.

Linux 서버에 원격 접속할 때 공개 키 인증과 네트워크 경로가 어떻게 결합되는지 알아본다.


