---
title: "Chapter 03. ReplicaSet and Deployment"
permalink: /aws-backend/part-09/03-replicaset-deployment/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. ReplicaSet and Deployment
## 원하는 Pod 수를 유지하고 배포하기

> **학습 목표**
>
> - ReplicaSet과 Deployment의 역할을 설명할 수 있다.
> - Deployment가 배포 단위로 사용되는 이유를 이해한다.
> - Rolling Update 흐름을 설명할 수 있다.
> - Spring Boot를 안전하게 배포하고 롤백할 수 있다.

---

# 왜 Deployment가 필요한가

Spring Boot 컨테이너가 수십 개로 늘어나면 서버 장애 때 복제본을 다시 만들고 새 버전으로 교체하는 작업을 사람이 반복할 수 없다.

무중단 배포에는 준비된 새 Pod만 트래픽에 넣고 기존 Pod를 점진적으로 줄이며, 문제 발생 시 이전 버전으로 되돌리는 조정자가 필요하다.

Deployment는 복제본 유지, 롤링 업데이트, 배포 이력과 롤백을 선언적으로 관리한다.

---

# ReplicaSet이란?

ReplicaSet은 원하는 Pod 개수를 유지한다.

현재 Pod 수가 `replicas`보다 적으면 만들고 많으면 제거하며 Label Selector로 관리 대상을 식별한다.

---

# Deployment란?

Deployment는 ReplicaSet을 관리하고 배포 전략을 제공한다.

일반적으로 사용자는 Pod나 ReplicaSet을 직접 만들기보다 Deployment를 만든다.

```text
Deployment
    │ creates
    ▼
ReplicaSet v2 ── maintains ──▶ Pod v2 × 3
    ▲
    │ rollout replaces
ReplicaSet v1 ───────────────▶ Pod v1 × 0
```

1. Deployment Controller가 Pod Template에 해당하는 ReplicaSet을 만든다.
2. ReplicaSet이 필요한 Pod를 생성한다.
3. 이미지가 바뀌면 새 ReplicaSet을 늘리고 기존 ReplicaSet을 줄인다.
4. 롤백하면 이전 ReplicaSet의 복제본 수를 다시 늘린다.

| 객체 | 복제 유지 | 업데이트 | 대표 용도 |
|---|---:|---:|---|
| ReplicaSet | O | 직접 관리 | Deployment 내부 |
| Deployment | O | Rolling Update | 무상태 API |
| StatefulSet | O | 순서 보장 | 상태 저장 앱 |
| DaemonSet | Node마다 | Rolling Update | 로그 에이전트 |

---

# Rolling Update

Deployment는 새 버전 Pod를 조금씩 늘리고 기존 버전 Pod를 줄일 수 있다.

Readiness Probe가 실패하면 트래픽을 받지 않게 할 수 있다.

`maxSurge`는 추가할 수 있는 Pod 수이고 `maxUnavailable`은 동시에 사용할 수 없어도 되는 Pod 수다.

---

# YAML 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
    spec:
      serviceAccountName: order-api
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
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/order-api
kubectl rollout history deployment/order-api
kubectl rollout undo deployment/order-api
```

---

# Spring Boot에서는 어떻게 쓰는가

Actuator Probe를 활성화해야 Deployment가 실제 준비 완료 시점과 프로세스 교착 상태를 구분할 수 있다.

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

ConfigMap으로 `SPRING_PROFILES_ACTIVE`를 주입하고 Secret으로 비밀값을 분리한다.

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    valueFrom:
      configMapKeyRef:
        name: order-api-config
        key: profile
envFrom:
  - secretRef:
      name: order-api-secret
```

AWS SDK를 사용하는 Pod는 IRSA가 연결된 `serviceAccountName`을 지정하여 필요한 AWS 권한만 얻는다.

JVM 종료 유예와 `terminationGracePeriodSeconds`를 맞추고 메모리 제한 안에 Heap과 Native Memory가 들어오도록 설정한다.

---

# 실무에서는 어떻게 사용할까

이미지에는 변경되지 않는 버전 태그나 Digest를 사용해 동일한 선언이 항상 동일한 바이너리를 가리키게 한다.

CI/CD는 `kubectl rollout status`가 성공할 때만 배포 완료로 판단하고 실패하면 로그와 이벤트를 보존한다.

DB 스키마는 구버전과 신버전이 함께 실행되는 롤링 배포 기간에도 호환되어야 한다.

---

# 장애 사례와 주의할 점

새 Pod가 readinessProbe를 통과하지 못하면 롤아웃이 멈추므로 로그와 `kubectl describe pod`의 이벤트를 확인한다.

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

잘못된 이미지와 ECR 권한은 `ImagePullBackOff`, 시작 예외는 `CrashLoopBackOff`를 만든다.

메모리 제한이 없으면 한 Pod가 Node를 불안정하게 만들고, 너무 낮으면 `OOMKilled`로 새 ReplicaSet이 준비되지 않는다.

메모리 세션과 장시간 연결을 Pod에 두면 업데이트 중 연결이 끊기므로 외부 세션 저장소와 재연결을 사용한다.

- Selector는 Pod Template Label과 정확히 일치해야 한다.
- readinessProbe 없이 `maxUnavailable: 0`만 설정해도 준비되지 않은 Pod 유입을 막을 수 없다.
- livenessProbe를 너무 공격적으로 설정하면 재시작 루프가 발생한다.
- `latest` 태그는 배포 재현성을 흐리므로 피한다.

---

# 비용과 성능 고려사항

`maxSurge`는 배포 중 추가 EC2 또는 Fargate 용량을 필요로 할 수 있다.

requests가 과대하면 Node가 늘고, 과소하면 CPU 경합과 메모리 압박으로 지연 시간이 불안정해진다.

EKS Control Plane, Worker Node, 이미지 전송, NAT Gateway와 관측성 데이터 비용을 함께 본다.

---

# 기억해야 할 내용

- ReplicaSet은 Pod의 원하는 개수를 유지한다.
- Deployment가 ReplicaSet을 만들고 ReplicaSet이 Pod를 관리한다.
- 새 Pod Template은 새 ReplicaSet과 Revision을 만든다.
- Rolling Update는 새 복제본을 늘리면서 기존 복제본을 줄인다.
- readinessProbe는 새 Pod의 트래픽 투입 시점을 결정한다.
- 고정 이미지 태그와 배포 이력이 안전한 롤백의 기반이다.

---

# 다음 Chapter

다음 Chapter에서는 안정적인 접근점을 제공하는 [Service](/aws-backend/part-09/04-service/)를 학습한다.

Label Selector, 가상 IP와 DNS가 Pod 교체를 호출자로부터 어떻게 숨기는지 살펴본다.


