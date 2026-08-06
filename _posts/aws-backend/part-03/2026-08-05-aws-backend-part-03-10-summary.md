---
title: "Chapter 10. Part 3 Summary"
permalink: /aws-backend/part-03/10-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Part 3 Summary
## Compute 핵심 정리

Part 3의 핵심은 EC2가 어떤 구성 요소로 만들어지고, Spring Boot 애플리케이션이 어떻게 실행되는지 설명하는 것이다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| EC2 | AWS의 가상 서버 |
| AMI | EC2 시작 이미지 |
| EBS | EC2에 붙는 블록 스토리지 |
| ENI | EC2의 네트워크 인터페이스 |
| Linux | EC2 위에서 애플리케이션을 실행하는 OS |
| SSH | 서버 원격 접속 방식 |
| Docker | 실행 환경을 이미지로 포장 |
| Systemd | Linux 서비스 관리자 |

---

# 실행 흐름

```
AMI
  -> EC2 Instance
  -> EBS attached
  -> ENI attached
  -> Linux boot
  -> Docker or Systemd
  -> Spring Boot running
```

---

# 실무 기본 설계

- EC2는 Private Subnet에 둔다.
- ALB를 통해서만 애플리케이션에 접근한다.
- SSH는 최소한으로 열거나 Session Manager를 사용한다.
- 애플리케이션 로그는 CloudWatch Logs로 모은다.
- 직접 운영이 커지면 ECS로 이전을 고려한다.

---

# 장애 확인 순서

1. EC2가 `running` 상태인가?
2. Security Group이 필요한 포트를 허용하는가?
3. 애플리케이션 프로세스가 떠 있는가?
4. 포트가 열려 있는가?
5. 로그에 에러가 있는가?
6. 디스크, 메모리, CPU가 부족하지 않은가?


