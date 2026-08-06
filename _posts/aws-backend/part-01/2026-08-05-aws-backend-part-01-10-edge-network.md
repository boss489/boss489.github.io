---
title: "Chapter 10. Edge Network"
permalink: /aws-backend/part-01/10-edge-network/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Edge Network
## 사용자 가까이에서 응답하는 네트워크

> **학습 목표**
>
> - Edge Location의 역할을 설명할 수 있다.
> - CloudFront, Global Accelerator, DNS의 목적을 이해한다.
> - Region과 Edge Network의 차이를 설명할 수 있다.

---

# Edge Location

Edge Location은 사용자와 가까운 위치에서 네트워크 기능을 제공하는 AWS 거점이다.

Region이 애플리케이션과 데이터를 실행하는 곳이라면, Edge Location은 사용자 요청을 더 빠르게 처리하기 위한 앞단 네트워크에 가깝다.

![AWS global infrastructure](/assets/aws-backend/aws-global-infra.png)

---

# CloudFront

CloudFront는 CDN 서비스다.

정적 파일이나 캐시 가능한 응답을 Edge Location에 저장해 사용자에게 빠르게 전달한다.

예시는 다음과 같다.

- 이미지
- JavaScript
- CSS
- 다운로드 파일
- 캐시 가능한 API 응답

---

# Global Accelerator

Global Accelerator는 사용자의 트래픽을 AWS 글로벌 네트워크로 빠르게 진입시키는 서비스다.

고정 Anycast IP를 제공하고, 가장 적절한 엔드포인트로 트래픽을 보낼 수 있다.

---

# DNS

DNS는 도메인 이름을 IP 주소나 AWS 리소스로 연결한다.

AWS에서는 Route 53이 대표 DNS 서비스다.

---

# 기억해야 할 내용

- Edge Network는 사용자 가까이에서 성능을 높이기 위한 구조다.
- CloudFront는 콘텐츠 캐싱에 사용한다.
- Route 53은 도메인 이름을 리소스로 연결한다.
- 일반 백엔드 요청은 Region에서 처리되고, 캐시 가능한 콘텐츠는 Edge에서 처리할 수 있다.


