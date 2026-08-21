---
title: "Chapter 06. ALB Controller"
permalink: /aws-backend/part-09/06-alb-controller/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. ALB Controller
## Kubernetes와 AWS ALB 연결하기

> **학습 목표**
>
> - ALB Controller의 역할을 설명할 수 있다.
> - Ingress가 AWS ALB로 변환되는 흐름을 이해한다.
> - IAM 권한과 Annotation이 필요한 이유를 설명할 수 있다.
> - IRSA와 Target Type을 올바르게 선택할 수 있다.

---

# 왜 ALB Controller가 필요한가

Kubernetes API는 AWS의 ALB, Listener, Target Group과 Security Group을 직접 생성하는 방법을 알지 못한다.

운영자가 Ingress를 바꿀 때마다 AWS 콘솔에서 규칙과 Target을 수동 변경하면 Pod 교체와 배포 속도를 따라갈 수 없고 설정 불일치가 발생한다.

AWS Load Balancer Controller는 Kubernetes 선언을 감시해 AWS 리소스의 실제 상태를 지속적으로 맞춘다.

---

# ALB Controller란?

AWS Load Balancer Controller는 Kubernetes Ingress나 Service 설정을 보고 AWS ALB/NLB를 생성하고 관리한다.

Kubernetes 객체와 AWS 리소스를 연결하는 다리 역할을 한다.

Ingress를 보고 ALB를, `type: LoadBalancer` Service를 보고 NLB를 프로비저닝할 수 있다.

---

# 동작 흐름

```text
Ingress / Service
       │ watch
       ▼
AWS Load Balancer Controller Pod
       │ AWS API + IRSA
       ▼
ALB or NLB ──▶ Listener ──▶ Target Group
                                  │
                            Pod IP or Node
```

1. 사용자가 Ingress를 만든다.
2. Controller가 Ingress를 감지한다.
3. AWS API로 ALB, Listener, Target Group을 만든다.
4. Pod 또는 Node를 Target으로 등록한다.
5. 선언이 바뀌거나 Pod가 교체되면 AWS 리소스를 다시 조정한다.

---

# Target Type 비교

| Target Type | 등록 대상 | Service 요구 | 특징 |
|---|---|---|---|
| `ip` | Pod IP | ClusterIP 가능 | Pod로 직접 전달 |
| `instance` | Worker Node | NodePort 필요 | Node를 거쳐 Pod로 전달 |

EKS의 VPC CNI를 사용하는 일반적인 구성에서는 `ip` Target Type으로 Pod IP를 Target Group에 직접 등록할 수 있다.

Fargate Pod는 Node Instance를 Target으로 등록할 수 없으므로 `ip` 방식을 사용한다.

---

# YAML 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789012:certificate/example
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/readiness
    alb.ingress.kubernetes.io/group.name: backend-api
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-api
                port:
                  number: 80
```

Controller가 사용할 ServiceAccount에는 IRSA로 IAM Role을 연결한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/AwsLoadBalancerControllerRole
```

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get ingress order-api
kubectl describe ingress order-api
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

# Spring Boot에서는 어떻게 쓰는가

ALB Health Check와 Kubernetes readinessProbe를 동일한 Actuator readiness Endpoint에 연결하면 트래픽 투입 기준을 일관되게 유지할 수 있다.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
server:
  shutdown: graceful
  forward-headers-strategy: framework
```

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
```

애플리케이션의 AWS 권한도 별도 IRSA Role로 부여하며 Controller Role을 Spring Boot Pod와 공유하지 않는다.

ConfigMap과 Secret으로 `SPRING_PROFILES_ACTIVE`와 비밀값을 주입하고 `terminationGracePeriodSeconds`로 종료 시간을 보장한다.

---

# 실무에서는 어떻게 사용할까

Controller는 Helm 등으로 버전을 고정해 설치하고 EKS 버전과의 호환성을 확인한다.

Public ALB와 Internal ALB를 Scheme으로 구분하고 Subnet 태그, Security Group과 인증서 수명 주기를 함께 관리한다.

Ingress Group은 ALB 수를 줄이는 데 유용하지만 같은 그룹을 수정할 수 있는 사용자를 신뢰 경계 안으로 제한한다.

---

# 장애 사례

ALB가 생성되지 않으면 Controller 로그에서 IAM `AccessDenied`, 잘못된 Annotation, Subnet 자동 발견 실패를 확인한다.

Target이 Unhealthy이면 Service Selector, EndpointSlice, Security Group과 Health Check 경로 및 응답 코드를 순서대로 점검한다.

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
kubectl describe ingress order-api
kubectl get endpointslice
```

Controller가 `CrashLoopBackOff`이면 설치 인자, Webhook 인증서, 클러스터 이름과 IRSA 신뢰 정책을 확인한다.

Spring Boot Pod가 `OOMKilled`되거나 readiness에 실패하면 ALB Target도 빠지므로 자원 requests/limits와 JVM 메모리를 분석한다.

---

# 주의할 점

- Controller IAM Policy는 공식 버전과 맞추고 최소 권한으로 관리한다.
- Annotation 오타는 AWS 리소스 생성 실패나 의도하지 않은 공개 범위를 만들 수 있다.
- Ingress 삭제 시 함께 삭제될 AWS 리소스와 DNS 의존성을 확인한다.
- ALB Health Check가 인증 또는 Redirect 때문에 실패하지 않게 한다.
- 롤링 배포의 종료 유예와 Target Deregistration 시간을 조정한다.

---

# 비용과 성능 고려사항

Ingress 하나마다 ALB가 생성되는 구성은 ALB 개수와 처리량 기반 비용을 늘리므로 Grouping을 검토한다.

ALB, EKS Control Plane, EC2 Node 또는 Fargate, NAT Gateway와 교차 AZ 데이터 전송이 별도 비용 요소다.

`ip` Target은 네트워크 경로를 단순화하지만 Pod IP 수요가 늘어나므로 Subnet 주소 용량을 계획해야 한다.

Health Check 주기를 지나치게 짧게 하면 대상과 네트워크에 불필요한 부하가 생긴다.

---

# 기억해야 할 내용

- Controller는 Kubernetes 선언을 AWS 로드밸런서 리소스로 조정한다.
- Ingress는 ALB, LoadBalancer Service는 NLB와 연결할 수 있다.
- `ip` Target은 Pod IP, `instance` Target은 Worker Node를 등록한다.
- Controller의 AWS API 권한은 IRSA로 부여한다.
- Annotation이 Scheme, 인증서, Target Type과 Health Check를 결정한다.
- Controller 로그와 Ingress 이벤트가 생성 실패의 핵심 단서다.
- ALB Grouping은 비용과 신뢰 경계를 함께 고려해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 관리형 Kubernetes 서비스인 [EKS](/aws-backend/part-09/07-eks/)를 학습한다.

AWS가 관리하는 Control Plane과 사용자가 책임지는 Worker Node, 네트워크 및 Add-on의 경계를 살펴본다.


