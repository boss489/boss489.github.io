---
title: "Chapter 07. Security Group"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Security Group
## 리소스 단위 방화벽

Security Group은 EC2, ALB, RDS 같은 리소스에 붙는 가상 방화벽이다.

인바운드와 아웃바운드 트래픽을 제어한다.

![Security controls](/assets/aws-backend/security-controls.svg)

---

# 특징

- 리소스 단위로 적용된다.
- 허용 규칙만 작성한다.
- 상태 저장 방식이다.
- 다른 Security Group을 Source로 지정할 수 있다.

상태 저장이므로 인바운드 요청을 허용하면 응답 트래픽은 자동으로 허용된다.

---

# 기본 예시

ALB Security Group:

| Type | Port | Source |
|---|---:|---|
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |

Spring Boot Security Group:

| Type | Port | Source |
|---|---:|---|
| Custom TCP | 8080 | ALB Security Group |

RDS Security Group:

| Type | Port | Source |
|---|---:|---|
| MySQL | 3306 | Spring Boot Security Group |

---

# 좋은 규칙

IP보다 Security Group 참조를 우선 사용한다.

예를 들어 Spring Boot 서버는 `0.0.0.0/0:8080`을 열지 않고 ALB Security Group에서만 접근하도록 둔다.

이렇게 하면 서버 IP가 바뀌어도 권한 구조가 깨지지 않는다.

---

# 자주 하는 실수

- EC2에 SSH를 `0.0.0.0/0`으로 여는 것
- RDS를 Public으로 만들고 전체 IP를 허용하는 것
- ALB가 아닌 외부에서 애플리케이션 포트로 직접 접근하게 두는 것
- 테스트용 규칙을 지우지 않는 것

---

# 핵심 요약

Security Group은 리소스 단위 접근 제어다.

실무에서는 IP 직접 허용보다 Security Group 간 참조로 흐름을 제한하는 방식이 안전하다.
