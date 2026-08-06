---
title: "Chapter 09. Elastic IP"
permalink: /aws-backend/part-02/09-elastic-ip/
date: 2026-08-05T09:17:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Elastic IP
## 고정 Public IP

Elastic IP는 AWS에서 사용하는 고정 Public IPv4 주소다.

EC2나 NAT Gateway처럼 고정 IP가 필요한 리소스에 연결한다.

![Elastic IP with NAT Gateway](/assets/aws-backend/elastic-ip.png)

---

# 왜 필요한가

일반 Public IP는 리소스를 중지하거나 재생성하면 바뀔 수 있다.

하지만 외부 시스템이 특정 IP를 허용해야 하는 경우가 있다.

- 외부 결제사 IP Allowlist
- 사내 방화벽 허용
- 고정 출구 IP가 필요한 API 호출

이때 Elastic IP를 사용한다.

---

# NAT Gateway와 Elastic IP

Private Subnet 서버들이 NAT Gateway를 통해 외부 API를 호출하면 외부에서는 NAT Gateway의 Elastic IP로 보인다.

```
Spring Boot
  │
NAT Gateway + Elastic IP
  │
External API
```

외부 API 제공자에게는 이 Elastic IP만 허용해달라고 요청하면 된다.

---

# 주의할 점

Elastic IP는 공짜 고정 IP가 아니다.

할당만 하고 사용하지 않거나, 연결 상태에 따라 비용이 발생할 수 있다.

필요 없는 Elastic IP는 정리해야 한다.

---

# 핵심 요약

Elastic IP는 바뀌지 않는 Public IP가 필요할 때 사용한다.

백엔드에서는 보통 NAT Gateway의 고정 출구 IP를 만들 때 자주 사용한다.
