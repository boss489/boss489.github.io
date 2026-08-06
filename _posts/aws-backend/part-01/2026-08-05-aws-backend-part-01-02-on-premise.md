---
title: "Chapter 02. On-Premise Infrastructure"
permalink: /aws-backend/part-01/02-on-premise/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. On-Premise Infrastructure
## 직접 소유하고 직접 운영하는 인프라

> **학습 목표**
>
> - On-Premise 인프라의 구성 요소를 설명할 수 있다.
> - IDC, Rack, Power, Cooling, Network의 역할을 이해한다.
> - On-Premise 증설 과정이 왜 느린지 설명할 수 있다.

---

# On-Premise란?

On-Premise는 회사가 서버와 네트워크 장비를 직접 소유하고 운영하는 방식이다.

보통 회사 내부 서버실이나 IDC에 장비를 둔다.

```
Company
└── IDC
    ├── Rack
    ├── Server
    ├── Switch
    ├── Storage
    └── Firewall
```

---

# IDC

IDC(Internet Data Center)는 서버를 안정적으로 운영하기 위한 시설이다.

IDC는 다음을 제공한다.

- 전원
- 냉각
- 네트워크 회선
- 물리 보안
- 랙 공간
- 장애 대응 절차

서버 애플리케이션은 코드만으로 동작하지 않는다. 전원, 온도, 네트워크가 안정적이어야 한다.

---

# 증설 과정

On-Premise에서 서버를 늘리는 과정은 느리다.

1. 필요한 서버 사양을 산정한다.
2. 구매 승인을 받는다.
3. 장비를 주문한다.
4. IDC에 반입한다.
5. 랙에 장착한다.
6. 네트워크와 보안을 설정한다.
7. OS와 애플리케이션을 설치한다.

트래픽이 갑자기 증가했을 때 이 과정을 즉시 수행하기 어렵다.

---

# 비용 구조

On-Premise는 대부분 선투자 비용이다.

서버를 30%만 사용해도 100% 비용을 이미 지불한 상태다.

반대로 갑자기 트래픽이 증가하면 미리 사둔 서버가 부족할 수 있다.

---

# 기억해야 할 내용

- On-Premise는 인프라를 직접 소유하고 운영하는 방식이다.
- 서버 운영에는 전원, 냉각, 네트워크, 물리 보안이 필요하다.
- 증설이 느리고 초기 비용이 크다.
- Cloud는 이 운영 부담을 API 기반 리소스 사용으로 바꾼다.


