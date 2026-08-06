---
title: "Chapter 04. Service"
permalink: /aws-backend/part-09/04-service/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Service
## Pod 앞의 안정적인 접근 지점

> **학습 목표**
>
> - Service의 역할을 설명할 수 있다.
> - Pod IP가 바뀌어도 접근 가능한 이유를 이해한다.
> - ClusterIP, NodePort, LoadBalancer의 차이를 구분할 수 있다.

---

# Service란?

Service는 여러 Pod 앞에 고정된 접근 지점을 제공하는 Kubernetes 객체다.

Pod는 죽고 다시 생성되면 IP가 바뀔 수 있다.

Service는 Label Selector로 Pod를 찾아 트래픽을 보낸다.

---

# 타입

| Type | 설명 |
|---|---|
| ClusterIP | 클러스터 내부 접근 |
| NodePort | 노드 포트로 외부 접근 |
| LoadBalancer | 클라우드 로드밸런서 생성 |

---

# 기억해야 할 내용

Service는 Pod IP 변화로부터 호출자를 보호한다.

Ingress와 함께 외부 HTTP 요청을 Pod로 연결한다.


