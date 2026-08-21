---
title: "Chapter 09. Part 4 Summary"
permalink: /aws-backend/part-04/09-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Part 4 Summary
## Request Flow 핵심 정리

![Request flow](/assets/aws-backend/request-flow.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| DNS | 도메인을 접속 대상으로 변환 |
| Route 53 | AWS DNS 서비스 |
| ALB | HTTP/HTTPS 요청 분산 |
| Listener | ALB의 요청 입구 |
| Target Group | 요청을 받을 대상 묶음 |
| Health Check | 정상 대상 판별 |
| Auto Scaling | 부하에 따라 대상 수 조절 |

---

# 장애 확인 순서

1. DNS가 올바른 ALB를 가리키는가?
2. ALB Listener가 포트를 열고 있는가?
3. Rule이 올바른 Target Group을 향하는가?
4. Target Group Health Check가 성공하는가?
5. Security Group이 ALB에서 애플리케이션 포트를 허용하는가?
6. 애플리케이션이 해당 포트에서 실행 중인가?

---

# 상태 코드로 좁히기

| 증상 | 먼저 볼 위치 |
|---|---|
| 도메인이 다른 IP를 반환 | DNS Record, TTL, Route 53 Alias |
| ALB 4xx | Listener Rule, 애플리케이션 요청 검증 |
| ALB 5xx | Target Group 상태, 포트, 애플리케이션 로그 |
| 특정 Target만 실패 | Health Check 경로와 해당 인스턴스 |

장애를 “ALB 문제”로 묶지 말고, 클라이언트·DNS·ALB·Target·애플리케이션 중 마지막으로 성공한 지점부터 확인한다.

