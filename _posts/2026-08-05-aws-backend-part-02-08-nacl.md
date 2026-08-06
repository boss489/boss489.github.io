---
title: "Chapter 08. NACL"
date: 2026-08-05T09:18:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. NACL
## Subnet 단위 접근 제어

NACL(Network Access Control List)은 Subnet 단위로 적용되는 네트워크 접근 제어다.

Security Group이 리소스 단위라면, NACL은 Subnet 단위다.

![Security controls](/assets/aws-backend/security-controls.png)

---

# 특징

- Subnet 단위로 적용된다.
- 허용과 거부 규칙을 모두 작성할 수 있다.
- 상태 비저장 방식이다.
- 규칙 번호가 낮은 것부터 평가한다.

상태 비저장이므로 요청과 응답 트래픽을 각각 허용해야 한다.

---

# Security Group과 차이

| 구분 | Security Group | NACL |
|---|---|---|
| 적용 범위 | 리소스 | Subnet |
| 규칙 | 허용만 | 허용, 거부 |
| 상태 | 상태 저장 | 상태 비저장 |
| 평가 방식 | 모든 규칙 | 낮은 번호부터 |

대부분의 접근 제어는 Security Group으로 처리한다.

NACL은 Subnet 전체를 막아야 하는 경우에 사용한다.

---

# 사용 예시

특정 악성 IP를 Subnet 단위로 차단해야 할 때 NACL을 사용할 수 있다.

```
Rule 100: deny  203.0.113.10/32
Rule 200: allow 0.0.0.0/0
```

낮은 번호부터 평가되므로 차단 규칙을 허용 규칙보다 앞에 둔다.

---

# 주의할 점

NACL을 과하게 쓰면 장애 분석이 어려워진다.

특히 Ephemeral Port 응답 트래픽을 막으면 요청은 나갔는데 응답이 돌아오지 않는 문제가 생긴다.

기본은 Security Group으로 제어하고, NACL은 명확한 이유가 있을 때만 사용한다.

---

# 핵심 요약

NACL은 Subnet 단위의 상태 비저장 접근 제어다.

백엔드 운영에서는 Security Group을 주로 쓰고, NACL은 넓은 범위 차단이 필요할 때 보조로 쓴다.
