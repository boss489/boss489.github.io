---
title: "Chapter 10. Interview Questions"
permalink: /aws-backend/part-09/10-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Interview Questions
## Kubernetes 면접 질문

---

## Pod란 무엇인가요?

Kubernetes에서 컨테이너를 실행하는 최소 단위입니다.

## ReplicaSet과 Deployment의 차이는 무엇인가요?

ReplicaSet은 Pod 개수를 유지하고, Deployment는 ReplicaSet을 관리하며 배포 전략을 제공합니다.

## Service는 왜 필요한가요?

Pod IP는 바뀔 수 있으므로, Pod 앞에 안정적인 접근 지점을 제공하기 위해 필요합니다.

## Ingress와 Ingress Controller의 차이는 무엇인가요?

Ingress는 라우팅 규칙이고, Ingress Controller는 그 규칙을 실제 로드밸런서나 프록시 설정으로 반영하는 컴포넌트입니다.

## ECS와 EKS 선택 기준은 무엇인가요?

단순한 AWS 컨테이너 운영이면 ECS가 유리하고, Kubernetes 표준과 생태계가 필요하면 EKS를 검토합니다.


