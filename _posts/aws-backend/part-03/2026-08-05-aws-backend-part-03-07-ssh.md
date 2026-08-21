---
title: "Chapter 07. SSH"
permalink: /aws-backend/part-03/07-ssh/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. SSH
## 서버에 안전하게 접속하는 방식

> **학습 목표**
>
> - SSH 접속 흐름을 설명할 수 있다.
> - Key Pair와 Security Group의 역할을 이해한다.
> - SSH를 열 때의 보안 주의사항을 설명할 수 있다.
> - Bastion Host와 Session Manager 방식을 비교할 수 있다.

---

# 왜 안전한 원격 접속이 필요한가

운영 중인 Spring Boot 서버에서 로그와 프로세스를 확인하려면 원격 관리 경로가 필요하다.

비밀번호를 인터넷으로 그대로 보내거나 관리 포트를 모든 주소에 공개하면 탈취와 무차별 대입 공격에 노출된다.

SSH는 통신을 암호화하고 공개 키 인증으로 접속자를 확인한다.

그러나 SSH 프로토콜이 안전하더라도 키 관리와 네트워크 규칙이 부실하면 서버는 안전하지 않다.

---

# SSH란?

SSH(Secure Shell)는 원격 서버에 안전하게 접속하기 위한 프로토콜이다.

EC2 Linux 서버에 접속할 때 일반적으로 SSH를 사용한다.

![SSH access](/assets/aws-backend/ssh-access.png)

---

# SSH 접속 구조

클라이언트는 Private Key를 보관하고 EC2에는 대응하는 Public Key가 등록된다.

```
Operator
├── Private Key
└── SSH Client
       │ TCP 22
       ▼
Security Group
       │
EC2 sshd
└── ~/.ssh/authorized_keys
      Public Key
```

Private Key 자체가 네트워크로 전송되는 것은 아니며 서버가 보낸 과제에 서명해 키 보유를 증명한다.

서버의 Host Key는 접속 대상 서버가 바뀌지 않았는지 확인하여 중간자 공격을 줄인다.

---

# 접속에 필요한 것

SSH 접속에는 다음이 필요하다.

- EC2 Public IP 또는 접근 가능한 Private IP
- 사용자 이름
- Private Key
- Security Group의 22번 포트 허용
- 네트워크 경로

예시는 다음과 같다.

```bash
ssh -i key.pem ec2-user@1.2.3.4
```

---

# 보안 주의사항

SSH를 `0.0.0.0/0`에 열어두면 위험하다.

가능하면 다음 방식으로 제한한다.

- 회사 고정 IP만 허용
- Bastion Host 사용
- Session Manager 사용
- 키 파일 권한 제한

---

# 접속 방식 비교

| 방식 | EC2 Public IP | 22번 포트 | 특징 |
|---|---:|---:|---|
| 직접 SSH | 필요 | 운영자 IP 허용 | 구성이 단순하지만 공개 면이 생김 |
| Bastion Host | 대상에는 불필요 | Bastion 경유 허용 | 중앙 진입점 운영 필요 |
| VPN·전용 연결 | 불필요 | 내부망 허용 | 네트워크 단위 통제 |
| Session Manager | 불필요 | 불필요 | IAM과 세션 감사 활용 |

Private Subnet의 인스턴스에 접속하려고 무조건 Public IP를 부여하기보다 Session Manager나 관리 네트워크를 검토한다.

---

# 키 파일과 SSH 설정

Private Key는 소유자만 읽을 수 있도록 권한을 제한한다.

```bash
chmod 400 key.pem
ssh -i key.pem ec2-user@203.0.113.10
```

사용자 이름은 AMI에 따라 다르므로 `ec2-user`, `ubuntu` 등을 정확히 구분해야 한다.

Key Pair는 인스턴스 시작 시 Public Key를 등록하는 수단이며 AWS가 내려받은 Private Key를 다시 제공하지 않는다.

Private Key는 Git 저장소, AMI, 메신저에 저장하지 않고 안전한 비밀 관리 절차로 배포·폐기한다.

---

# AWS Session Manager

Systems Manager Session Manager는 인스턴스의 관리 에이전트와 IAM 권한을 사용해 셸 세션을 제공한다.

```
Operator
   │ IAM 인증
AWS Systems Manager
   │ 아웃바운드 연결
SSM Agent on EC2
```

인스턴스에는 SSM Agent, 적절한 IAM Role, Systems Manager 서비스로 나가는 네트워크 경로가 필요하다.

Session Manager는 인바운드 22번 포트를 없앨 수 있지만 IAM 최소 권한과 세션 감사 설정은 별도로 구성해야 한다.

```bash
aws ssm start-session --target i-0123456789abcdef0
```

---

# 접속 실패 진단

| 메시지·증상 | 원인 후보 | 확인할 곳 |
|---|---|---|
| 연결 시간 초과 | 경로·SG·NACL·주소 오류 | VPC와 ENI |
| Connection refused | `sshd` 미실행 | Systemd와 포트 |
| Permission denied | 사용자·키 불일치 | AMI 사용자와 Public Key |
| Host key changed | 인스턴스 교체 또는 공격 가능성 | 대상 식별과 `known_hosts` |
| SSM 대상 없음 | Agent·IAM·네트워크 문제 | Agent 로그와 Role |

타임아웃은 인증 전의 네트워크 문제일 가능성이 크고 Permission denied는 서버 도달 후의 인증 문제이다.

`StrictHostKeyChecking=no`를 습관적으로 사용하면 서버 신원 검증을 우회하므로 신뢰할 Host Key를 관리해야 한다.

---

# 실무에서는 어떻게 사용할까

운영 서버를 직접 수정하기보다 배포 자동화와 중앙 로그를 우선 사용한다.

긴급 접속은 승인된 사용자에게 제한된 시간 동안만 허용하고 접속 기록을 감사 가능하게 만든다.

Bastion을 사용한다면 대상 Security Group은 Bastion Security Group만 소스로 허용한다.

공유 키는 개인별 추적과 폐기가 어려우므로 IAM 또는 개인별 인증 체계를 사용한다.

---

# 비용과 운영 고려사항

Bastion Host는 인스턴스와 EBS, 패치와 가용성 관리 비용이 지속적으로 발생한다.

Session Manager도 로그 저장이나 VPC Endpoint 구성에 관련 비용이 생길 수 있다.

선택 기준은 편의뿐 아니라 공격 표면, 감사 요구, 장애 시 접근 가능성을 포함해야 한다.

---

# 기억해야 할 내용

- SSH는 원격 서버 접속 프로토콜이다.
- 접속에는 키와 네트워크 허용이 모두 필요하다.
- Security Group에서 22번 포트를 아무 곳에나 열면 안 된다.
- 실무에서는 Session Manager를 쓰면 SSH 포트를 줄일 수 있다.
- Private Key는 네트워크로 전송되지 않고 서명에 사용된다.
- 타임아웃과 인증 실패는 서로 다른 계층의 문제이다.
- 운영 접속은 최소 권한과 감사 로그를 갖춰야 한다.

---

# 다음 Chapter

다음 Chapter에서는 [Docker](/aws-backend/part-03/08-docker/)를 학습한다.

서버마다 달라지는 Java와 파일 구성을 이미지로 고정해 동일한 실행 환경을 만드는 방법을 알아본다.


