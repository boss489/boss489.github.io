---
title: "Chapter 07. EKS"
permalink: /aws-backend/part-09/07-eks/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. EKS
## AWS 관리형 Kubernetes

> **학습 목표**
>
> - EKS의 역할을 설명할 수 있다.
> - Control Plane과 Worker Node의 차이를 이해한다.
> - EKS를 선택할 때 고려할 운영 비용을 설명할 수 있다.
> - Spring Boot 워크로드에 IRSA와 운영 설정을 적용할 수 있다.

---

# 왜 EKS가 필요한가

Kubernetes를 직접 구축하면 API Server, etcd, 인증서, 백업과 고가용성을 운영팀이 모두 책임져야 한다.

컨테이너 수십 개를 여러 서버에 배치하고 장애 시 자동 복구하며 무중단 배포와 롤백을 수행하려면 안정적인 Control Plane이 필요하다.

EKS는 Kubernetes Control Plane 운영을 AWS에 맡기면서 표준 API와 생태계를 사용할 수 있게 한다.

---

# EKS란?

EKS(Elastic Kubernetes Service)는 AWS의 관리형 Kubernetes 서비스다.

AWS가 Control Plane 운영을 관리한다.

사용자는 Worker Node 또는 Fargate, 애플리케이션, Add-on, IAM, 네트워크와 업그레이드 계획을 여전히 관리한다.

---

# 구조와 동작 흐름

```text
Developer / CI
      │ kubectl
      ▼
┌────────────────────┐
│ EKS Control Plane  │  AWS managed
│ API / etcd / Ctrl  │
└─────────┬──────────┘
          │
    ┌─────┴───────────────┐
    ▼                     ▼
EC2 Managed Node       Fargate
kubelet + Pods         Pods
    │
VPC CNI ── Pod IP ── VPC
```

1. 사용자가 API에 Deployment를 제출한다.
2. Scheduler가 Pod를 실행할 컴퓨팅 용량을 선택한다.
3. kubelet이 ECR에서 이미지를 받아 Pod를 실행한다.
4. VPC CNI가 Pod 네트워크를 구성한다.
5. Add-on과 Controller가 Service, Ingress와 AWS 리소스를 조정한다.

---

# 구성 요소 비교

| 구성 요소 | 관리 주체 | 역할 |
|---|---|---|
| EKS Control Plane | AWS | Kubernetes API와 상태 저장 |
| Managed Node Group | 공동 | EC2 Worker 수명 주기 단순화 |
| Fargate | AWS 중심 | Pod 단위 실행 |
| VPC CNI | 사용자 설정 | Pod에 VPC 네트워크 연결 |
| CoreDNS | 사용자 설정 | 클러스터 DNS |
| ALB Controller | 사용자 설치 | ALB/NLB 프로비저닝 |

| 컴퓨팅 | 장점 | 고려사항 |
|---|---|---|
| Managed Node Group | 폭넓은 워크로드 | Node 패치·용량 관리 |
| Fargate | Node 직접 관리 감소 | 제약과 Pod별 비용 |
| Self-managed Node | 높은 제어력 | 가장 큰 운영 부담 |

---

# YAML 예시

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-api
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/OrderApiRole
---
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
      serviceAccountName: order-api
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-api
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/order-api:1.0.0
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
aws eks update-kubeconfig --region ap-northeast-2 --name backend-cluster
kubectl get node
kubectl get pod -A
```

---

# Spring Boot에서는 어떻게 쓰는가

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

ConfigMap과 Secret을 환경변수로 주입해 환경별 이미지를 다시 만들지 않는다.

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

IRSA는 ServiceAccount와 IAM Role을 연결해 S3, SQS 같은 AWS 권한을 Pod 단위로 부여한다.

Node IAM Role에 애플리케이션 권한을 몰아주면 같은 Node의 다른 Pod도 권한을 얻을 수 있으므로 피한다.

---

# 실무에서는 어떻게 사용할까

운영 클러스터는 여러 AZ의 Private Subnet에 Node를 분산하고 API Endpoint 접근 범위를 제한한다.

Karpenter 또는 Cluster Autoscaler로 Pending Pod에 필요한 용량을 조정하되 requests를 정확히 설정한다.

EKS, Add-on, Controller와 Node AMI 버전을 함께 관리하고 업그레이드 전 API 호환성을 검사한다.

---

# 장애 사례와 주의할 점

Pod가 Pending이면 자원 부족, Taint, Subnet IP 고갈과 스케줄링 이벤트를 확인한다.

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod-name> --previous
```

`ImagePullBackOff`는 ECR 권한, 이미지 태그와 Private Subnet의 ECR 접근 경로를 점검한다.

`CrashLoopBackOff`는 이전 로그, ConfigMap과 Secret을 검증한다.

resources limits 미설정은 Node 메모리 고갈을, 너무 낮은 제한은 `OOMKilled`를 만들 수 있다.

VPC CNI의 Pod IP가 부족하면 Node 자원이 남아도 Pod 네트워크를 만들지 못한다.

- AWS가 Control Plane을 관리해도 Kubernetes 운영 책임 전체가 사라지지 않는다.
- readinessProbe 없이 배포하면 준비되지 않은 Pod가 트래픽을 받을 수 있다.
- Pod IP는 바뀌므로 Service DNS를 사용한다.
- 롤링 업데이트의 연결 종료와 외부 세션 저장을 설계한다.

---

# 비용과 성능 고려사항

EKS Control Plane은 사용 시간에 따라 과금되며 Worker Node용 EC2 또는 Fargate 비용은 별도로 발생한다.

Ingress별 ALB, LoadBalancer Service의 NLB, NAT Gateway 처리량과 교차 AZ 데이터 전송도 비용 요소다.

환경마다 클러스터를 분리하면 격리는 강해지지만 Control Plane과 Add-on 비용이 반복된다.

requests와 Autoscaling을 실제 부하로 조정하면 안정성과 Node 이용률을 함께 개선한다.

---

# 기억해야 할 내용

- EKS는 AWS 관리형 Kubernetes Control Plane을 제공한다.
- Worker Node, Add-on, 애플리케이션과 업그레이드는 사용자의 책임이다.
- VPC CNI는 Pod 네트워크를 VPC와 연결한다.
- IRSA는 Pod별 최소 IAM 권한을 부여한다.
- Managed Node Group과 Fargate는 운영 부담과 제약이 다르다.
- Control Plane, 컴퓨팅, 로드밸런서와 네트워크 비용을 함께 본다.
- 단순한 AWS 컨테이너 운영에는 ECS가 더 적합할 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 [ECS와 EKS](/aws-backend/part-09/08-ecs-vs-eks/)를 학습 곡선, 운영 부담, 생태계와 비용 관점에서 비교한다.

기능 수가 아니라 팀과 서비스 조건에 맞는 플랫폼을 선택하는 기준을 세운다.


