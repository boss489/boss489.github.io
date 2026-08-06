---
title: "Chapter 02. Pod"
permalink: /aws-backend/part-09/02-pod/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Pod
## Kubernetes의 최소 실행 단위

> **학습 목표**
>
> - Pod의 역할을 설명할 수 있다.
> - Pod와 Container의 관계를 이해한다.
> - Pod가 영구적인 서버가 아님을 설명할 수 있다.

---

# Pod란?

Pod는 Kubernetes에서 컨테이너를 실행하는 최소 단위다.

보통 하나의 Pod에 하나의 애플리케이션 컨테이너를 둔다.

---

# 특징

- 고유 IP를 가진다.
- 죽으면 새 Pod로 교체될 수 있다.
- 상태를 오래 보존하는 단위가 아니다.
- 로그와 파일은 외부 저장소로 보내야 한다.

---

# 기억해야 할 내용

Pod는 일회성 실행 단위다.

직접 Pod를 오래 관리하기보다 Deployment로 관리한다.


