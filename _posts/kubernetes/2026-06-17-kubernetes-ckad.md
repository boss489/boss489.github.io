---
title: "CKAD 취득 후기"
permalink: /kubernetes/ckad/
categories:
  - kubernetes
tags:
  - kubernetes
  - ckad
  - certification
last_modified_at: 2026-06-17T09:00:00+09:00
---

# CKAD(Certified Kubernetes Application Developer) 취득 후기

최근 CKAD(Certified Kubernetes Application Developer) 자격증을 취득하였다.

백엔드 개발자로 일하면서 Kafka, Redis, Spring Boot, AWS 기반의 서비스를 운영해 왔지만, Kubernetes에 대해서는 단편적인 지식만 가지고 있었다. 최근 MSA 환경과 클라우드 네이티브 아키텍처가 일반화되면서 Kubernetes에 대한 이해가 필요하다고 판단했고, CKAD 취득을 목표로 학습을 시작했다.

## 학습 방법

CKAD는 이론 시험이 아니라 실제 Kubernetes 환경에서 문제를 해결하는 실습형 시험이다.

따라서 단순히 강의를 듣는 방식보다는 직접 YAML을 작성하고 kubectl 명령어를 반복적으로 사용하는 방식으로 준비하였다.

### 1. Udemy Mumshad 강의 완주

Kubernetes 학습 자료로 가장 유명한 과정 중 하나인 Mumshad Mannambeth의 CKAD 강의를 완주하였다.

학습 내용은 다음과 같다.

- Pod
- Deployment
- Service
- ConfigMap
- Secret
- Volume
- Ingress
- NetworkPolicy
- RBAC
- CronJob
- PersistentVolume
- PersistentVolumeClaim

각 챕터별로 실습을 직접 수행하며 Kubernetes 오브젝트 간의 관계를 이해하는 데 집중하였다.

### 2. Killer.sh 실전 문제 풀이

CKAD 준비 과정에서 가장 도움이 되었던 것은 Killer.sh였다.

실제 시험과 유사한 환경에서 제한된 시간 안에 문제를 해결하는 연습을 반복하였다.

특히 다음 영역을 집중적으로 훈련하였다.

- Ingress 생성
- NetworkPolicy 구성
- RBAC 설정
- PersistentVolume/PVC 연결
- CronJob 작성
- Secret 및 ConfigMap 활용
- Multi-container Pod 구성

처음에는 시간 내에 문제를 해결하기 어려웠지만 반복 학습을 통해 kubectl 명령어와 YAML 작성 속도를 크게 향상시킬 수 있었다.

### 3. kubectl 중심 학습

CKAD는 Kubernetes를 얼마나 잘 아느냐보다 kubectl을 얼마나 빠르게 활용할 수 있느냐가 중요하다고 느꼈다.

반복적으로 사용한 명령어는 다음과 같다.

```bash
kubectl run
kubectl expose
kubectl create ingress
kubectl create configmap
kubectl create secret
kubectl set image
kubectl rollout undo
kubectl rollout status
kubectl explain
```

특히 kubectl explain은 시험 중 YAML 스펙을 확인하는 데 매우 유용했다.

## 실무와 연결된 부분

현재 업무에서는 Spring Boot, Kafka, Redis, AWS 기반 서비스를 개발 및 운영하고 있다.

CKAD를 준비하면서 다음과 같은 부분들을 Kubernetes 관점에서 다시 이해할 수 있었다.

### 애플리케이션 배포

- Rolling Update
- Rollback
- Replica 관리
- 무중단 배포

### 설정 관리

- ConfigMap
- Secret
- 환경별 설정 분리

### 네트워크

- Service
- Ingress
- NetworkPolicy

### 스토리지

- EmptyDir
- PersistentVolume
- PersistentVolumeClaim

## 느낀 점

CKAD는 단순히 자격증 취득이 목적이라기보다 Kubernetes 환경에서 애플리케이션을 운영하기 위한 실무 역량을 체계적으로 학습할 수 있는 과정이었다.

특히 Killer.sh 실습은 실제 운영 환경에서 발생할 수 있는 다양한 문제를 Kubernetes 관점에서 해결하는 연습이 되었고, kubectl 사용 능력과 YAML 작성 능력을 크게 향상시켜 주었다.

앞으로는 CKAD에서 학습한 내용을 바탕으로 실제 서비스 운영 환경에서도 Kubernetes 활용 범위를 넓혀 나갈 계획이다.

## 학습 결과

- CKAD(Certified Kubernetes Application Developer) 취득
- Udemy Mumshad CKAD 강의 완주
- Killer.sh 실전 문제 전부 풀이
- Kubernetes 핵심 오브젝트 실습 경험 확보
- Ingress, NetworkPolicy, RBAC, Storage 구성 역량 확보
- Kubernetes 기반 애플리케이션 배포 및 운영 역량 향상
