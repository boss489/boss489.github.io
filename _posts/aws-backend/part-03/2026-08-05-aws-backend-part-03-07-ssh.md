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

---

# SSH란?

SSH(Secure Shell)는 원격 서버에 안전하게 접속하기 위한 프로토콜이다.

EC2 Linux 서버에 접속할 때 일반적으로 SSH를 사용한다.

![SSH access](/assets/aws-backend/ssh-access.png)

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

# 기억해야 할 내용

- SSH는 원격 서버 접속 프로토콜이다.
- 접속에는 키와 네트워크 허용이 모두 필요하다.
- Security Group에서 22번 포트를 아무 곳에나 열면 안 된다.
- 실무에서는 Session Manager를 쓰면 SSH 포트를 줄일 수 있다.


