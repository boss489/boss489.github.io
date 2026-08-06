---
title: "Chapter 06. NAT Gateway"
date: 2026-08-05T09:20:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. NAT Gateway
## Private Subnet의 외부 요청 통로

NAT Gateway는 Private Subnet의 리소스가 인터넷으로 나갈 수 있게 해준다.

반대로 인터넷에서 Private Subnet으로 직접 들어오는 요청은 허용하지 않는다.

![Route table flow](/assets/aws-backend/route-table.png)

---

# 왜 필요한가

Private Subnet의 서버도 외부 통신이 필요할 수 있다.

- 패키지 다운로드
- 외부 API 호출
- OS 업데이트
- 컨테이너 이미지 Pull

서버를 Public Subnet에 두면 단순하지만 공격 면적이 커진다.

NAT Gateway를 사용하면 서버는 숨긴 채 외부 요청만 보낼 수 있다.

---

# 배치 위치

NAT Gateway는 Public Subnet에 둔다.

```
Private Subnet
  │
Route Table
  │
NAT Gateway (Public Subnet)
  │
Internet Gateway
  │
Internet
```

Private Subnet의 Route Table에는 다음 Route를 둔다.

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | NAT Gateway |

---

# 비용 주의

NAT Gateway는 비용이 자주 문제가 된다.

비용은 크게 두 가지로 발생한다.

- 시간당 비용
- 처리한 데이터 비용

트래픽이 많거나 AZ마다 NAT Gateway를 두면 비용이 커질 수 있다.

개발 환경에서는 꼭 필요한지 먼저 확인해야 한다.

---

# 핵심 요약

NAT Gateway는 Private Subnet 리소스의 외부 요청을 대신 보내는 통로다.

보안에는 유리하지만 비용이 있으므로 운영 환경과 개발 환경의 구성을 다르게 가져갈 수 있다.
