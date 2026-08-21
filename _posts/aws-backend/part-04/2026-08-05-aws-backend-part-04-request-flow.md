---
title: "Part 4. Request Flow"
permalink: /aws-backend/part-04/
date: 2026-08-05T09:13:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 4. Request Flow
## 인터넷 요청이 API까지 도달하는 과정

> 목표
>
> 사용자의 요청이 DNS, 로드밸런서, 애플리케이션까지 이동하는 흐름을 이해한다.

---

# 학습 내용

- DNS
- Route53
- ALB
- Target Group
- Health Check
- Auto Scaling
- Session
- Keep Alive

---

# 이 Part에서 추적할 요청 흐름

```text
Browser → DNS / Route 53 → ALB Listener → Rule → Target Group
        → Spring Boot → Session / Keep-Alive 처리 → Response
```

요청 장애는 한 컴포넌트만 보고 해결하기 어렵다. DNS부터 애플리케이션 포트까지 각 경계에서 “다음 단계로 전달되었는가”를 확인하는 순서가 중요하다.

# 실무 판단 기준

- 도메인은 Route 53 Alias로 ALB에 연결하고, 애플리케이션 인스턴스를 직접 공개하지 않는다.
- 배포 가능 여부는 Target Group Health Check가 판단한다. 상태 확인 URL은 인증 없이 빠르고 안정적으로 응답해야 한다.
- 세션을 특정 서버에 묶어야 한다면 Sticky Session의 장애·확장 한계를 먼저 고려한다.

---

# 완료 후 설명할 수 있어야 하는 것

- 도메인 요청이 서버까지 도달하는 과정
- ALB와 Target Group의 관계
- Health Check가 배포와 장애 대응에 미치는 영향
- Auto Scaling이 트래픽 증가에 대응하는 방식

---

다음 Part

→ **Part 5. Database**
