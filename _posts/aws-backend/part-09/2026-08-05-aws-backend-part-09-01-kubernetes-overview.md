---
title: "Chapter 01. Kubernetes Overview"
permalink: /aws-backend/part-09/01-kubernetes-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Kubernetes Overview
## 컨테이너 오케스트레이션 플랫폼

> **학습 목표**
>
> - Kubernetes가 해결하는 문제를 설명할 수 있다.
> - Pod, Deployment, Service, Ingress의 큰 관계를 이해한다.
> - EKS가 Kubernetes 운영에서 맡는 범위를 설명할 수 있다.
> - Spring Boot 애플리케이션의 배포 흐름을 설명할 수 있다.

---

# 왜 Kubernetes가 필요한가

컨테이너가 수십 개로 늘어나면 운영자가 서버를 골라 배치하고, 죽은 컨테이너를 재시작하고, 트래픽 증가에 맞춰 복제본을 추가하는 방식은 한계에 도달한다.

새 버전을 무중단으로 배포하려면 새 컨테이너의 준비 상태를 확인하면서 기존 컨테이너를 순차적으로 교체하고, 문제가 생기면 이전 버전으로 되돌려야 한다.

서버 한 대가 죽더라도 다른 서버에서 컨테이너를 자동으로 복구해야 서비스 가용성을 유지할 수 있다.

Kubernetes는 운영자가 원하는 상태를 선언하면 현재 상태를 지속적으로 비교하고 차이를 자동으로 조정한다.

---

# Kubernetes란?

Kubernetes는 컨테이너를 여러 서버 위에서 배치, 실행, 복구, 확장하는 오케스트레이션 플랫폼이다.

핵심은 명령을 한 번 실행하는 것이 아니라 `replicas: 3`처럼 **원하는 상태**를 선언하는 데 있다.

![Kubernetes workload](/assets/aws-backend/kubernetes-workload.png)

---

# 전체 구조

```text
                  kubectl / API
                       │
               ┌───────▼────────┐
               │ Control Plane  │
               │ API/Scheduler  │
               │ Controllers    │
               └───────┬────────┘
                       │ desired state
           ┌───────────┴───────────┐
    ┌──────▼──────┐         ┌──────▼──────┐
    │ Worker Node │         │ Worker Node │
    │ kubelet     │◀───────▶│ kubelet     │
    │ Pod, Pod    │         │ Pod, Pod    │
    └─────────────┘         └─────────────┘
```

Control Plane은 API 요청과 클러스터 상태를 저장하고 Pod를 실행할 Node를 결정한다.

각 Node의 `kubelet`은 Control Plane의 지시를 받아 컨테이너 런타임을 통해 Pod를 실행하고 상태를 보고한다.

| 구성 요소 | 역할 |
|---|---|
| API Server | Kubernetes API의 진입점 |
| Scheduler | 새 Pod를 실행할 Node 선택 |
| Controller Manager | 원하는 상태와 현재 상태 조정 |
| etcd | 클러스터 상태 저장 |
| kubelet | Node에서 Pod 실행과 상태 보고 |
| Container Runtime | 컨테이너 이미지 실행 |

---

# 핵심 객체와 동작 흐름

```text
Ingress → Service → Deployment → ReplicaSet → Pod
   │          │          │             │
HTTP 규칙   안정적 주소   배포 전략      개수 유지
```

1. Deployment가 새 ReplicaSet을 만든다.
2. ReplicaSet이 지정한 수만큼 Pod를 유지한다.
3. Service가 Label Selector로 Pod를 찾아 안정적인 DNS와 가상 IP를 제공한다.
4. Ingress가 Host와 Path 규칙에 따라 외부 HTTP 요청을 Service로 전달한다.

| 객체 | 핵심 책임 | 직접 만드는 경우 |
|---|---|---|
| Pod | 하나 이상의 컨테이너 실행 | 진단 목적 외에는 드묾 |
| ReplicaSet | Pod 복제본 수 유지 | 보통 Deployment가 생성 |
| Deployment | 무상태 앱 배포와 롤백 | 가장 일반적인 배포 단위 |
| StatefulSet | 안정적인 이름과 저장소가 필요한 앱 | 데이터 저장 워크로드 |
| DaemonSet | 모든 Node에 Pod 배치 | 로그·모니터링 에이전트 |
| Service | Pod 집합의 안정적인 접근점 | 내부·외부 통신 |
| Ingress | HTTP/HTTPS 라우팅 규칙 | 외부 웹 트래픽 |

---

# YAML 예시

다음 Deployment는 Spring Boot 애플리케이션 세 개를 실행하고 자원 및 상태 검사 기준을 선언한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-api
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployment,replicaset,pod
kubectl rollout status deployment/order-api
```

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot 3.x에서는 Actuator의 Kubernetes Probe를 활성화하고 종료 중 새 요청을 받지 않도록 Graceful Shutdown을 사용한다.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

ConfigMap과 Secret은 환경변수로 주입하며, 비밀값을 이미지나 Git에 포함하지 않는다.

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    valueFrom:
      configMapKeyRef:
        name: order-api-config
        key: profile
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-api-secret
        key: db-password
```

AWS API 접근 권한은 Node 전체 권한이 아니라 IRSA로 Kubernetes ServiceAccount와 IAM Role을 연결해 Pod 단위로 부여한다.

---

# 실무에서는 어떻게 사용할까

애플리케이션 팀은 Deployment와 Service를 선언하고 플랫폼 팀은 EKS, Ingress Controller, 관측성, 정책과 업그레이드를 관리하는 방식으로 책임을 나눌 수 있다.

운영 배포에서는 이미지 태그를 고정하고 readinessProbe 통과 후에만 트래픽을 받게 하며 롤아웃 상태를 CI/CD에서 확인한다.

개발·스테이징·운영은 Namespace나 별도 클러스터로 분리하되 보안 및 장애 격리 요구에 따라 경계를 결정한다.

---

# 장애 사례와 주의할 점

`CrashLoopBackOff`는 컨테이너가 반복 종료될 때 나타나므로 `kubectl logs --previous`와 `kubectl describe pod`로 이전 로그와 이벤트를 확인한다.

`ImagePullBackOff`는 이미지 이름, 태그, ECR 인증, 네트워크 경로를 우선 점검한다.

readinessProbe가 없으면 초기화가 끝나지 않은 Pod로 요청이 전달되어 5xx가 발생할 수 있다.

자원 제한과 요청량이 없으면 스케줄링이 부정확해지고 메모리 경합으로 `OOMKilled`가 발생해 Node 전체가 불안정해질 수 있다.

Pod는 재생성될 때 IP가 바뀌므로 Pod IP를 설정 파일에 고정하지 않고 Service DNS를 사용한다.

롤링 업데이트 중 장시간 연결이나 메모리 세션이 끊길 수 있으므로 종료 유예, 외부 세션 저장소, 재시도 전략을 함께 설계한다.

---

# 비용과 성능 고려사항

EKS는 Control Plane 사용 시간에 대한 비용과 Worker Node용 EC2 또는 Fargate 비용이 각각 발생한다.

Ingress마다 ALB를 별도로 만들면 로드밸런서 수가 늘어나므로 호스트·경로 라우팅과 Ingress Group을 검토한다.

Private Subnet의 Node가 ECR이나 외부 저장소에 접근하면 NAT Gateway 처리량과 데이터 전송 비용이 발생할 수 있다.

`requests`는 스케줄링과 용량 계획의 기준이고 `limits`는 과도한 자원 점유를 막지만 지나치게 낮으면 성능 저하와 재시작을 유발한다.

---

# 기억해야 할 내용

- Kubernetes는 선언한 원하는 상태를 지속적으로 유지한다.
- Control Plane은 클러스터를 제어하고 kubelet은 Node에서 Pod를 실행한다.
- Deployment가 ReplicaSet을 만들고 ReplicaSet이 Pod 수를 유지한다.
- Service는 변하는 Pod 앞에 안정적인 DNS와 가상 IP를 제공한다.
- Ingress는 HTTP 라우팅 규칙이며 실제 구현에는 Controller가 필요하다.
- Probe, 자원 설정, Graceful Shutdown은 안정적인 Spring Boot 운영의 기본이다.
- Kubernetes의 유연성은 높은 학습 및 운영 부담과 함께 평가해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 Kubernetes의 최소 실행 단위인 [Pod](/aws-backend/part-09/02-pod/)를 학습한다.

컨테이너가 Pod 안에서 네트워크와 저장 공간을 어떻게 공유하고, Pod를 일회성 자원으로 다뤄야 하는 이유를 살펴본다.
