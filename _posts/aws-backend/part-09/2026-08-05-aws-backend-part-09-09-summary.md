---
title: "Chapter 09. Part 9 Summary"
permalink: /aws-backend/part-09/09-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Part 9 Summary
## Kubernetes 핵심 정리

![Kubernetes workload](/assets/aws-backend/kubernetes-workload.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| Pod | 최소 실행 단위 |
| ReplicaSet | Pod 개수 유지 |
| Deployment | 배포와 ReplicaSet 관리 |
| Service | Pod 앞의 안정적 접근 지점 |
| Ingress | HTTP 라우팅 규칙 |
| ALB Controller | Ingress를 AWS ALB로 연결 |
| EKS | AWS 관리형 Kubernetes |

---

# 실무 포인트

- 직접 Pod를 만들기보다 Deployment를 사용한다.
- Service는 Pod IP 변화로부터 호출자를 보호한다.
- Ingress는 Controller가 있어야 실제로 동작한다.
- ECS보다 EKS 운영 복잡도가 높다.

---

# 트래픽과 상태의 분리

| 역할 | Kubernetes 리소스 |
|---|---|
| 원하는 Pod 수와 배포 이력 | Deployment |
| 실제 실행 중인 컨테이너 | Pod |
| 내부의 안정적인 호출 주소 | Service |
| 외부 HTTP 라우팅 규칙 | Ingress |
| AWS ALB 생성·동기화 | ALB Controller |

Pod가 실행 중이어도 readiness가 실패하면 Service의 대상이 되지 않는다. “Pod 상태”와 “트래픽 수신 가능 상태”를 구분하는 것이 운영의 시작이다.

