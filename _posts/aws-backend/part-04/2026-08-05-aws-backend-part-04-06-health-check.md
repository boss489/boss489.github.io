---
title: "Chapter 06. Health Check"
permalink: /aws-backend/part-04/06-health-check/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Health Check
## 요청을 받을 수 있는 서버인지 확인하기

> **학습 목표**
>
> - Health Check의 목적을 설명할 수 있다.
> - Health Check 실패가 트래픽 흐름에 미치는 영향을 이해한다.
> - Spring Boot에서 Health Check 엔드포인트를 설계하는 기준을 설명할 수 있다.

---

# Health Check란?

Health Check는 ALB가 Target이 정상인지 주기적으로 확인하는 기능이다.

정상 대상에게만 사용자 요청을 전달한다.

---

# Spring Boot 예시

보통 다음 같은 엔드포인트를 사용한다.

```text
/actuator/health
```

단순히 프로세스가 살아 있는지와 핵심 의존성이 정상인지 구분해서 설계해야 한다.

---

# 배포와 Health Check

새 버전을 배포했는데 Health Check가 실패하면 ALB는 해당 대상에 트래픽을 보내지 않는다.

이 덕분에 잘못된 서버가 사용자 요청을 받는 것을 줄일 수 있다.

---

# 기억해야 할 내용

- Health Check는 트래픽을 받을 수 있는 대상인지 확인한다.
- 경로, 포트, 응답 코드가 맞아야 정상으로 판단된다.
- Health Check 실패는 배포 실패와 장애 분석의 핵심 신호다.


