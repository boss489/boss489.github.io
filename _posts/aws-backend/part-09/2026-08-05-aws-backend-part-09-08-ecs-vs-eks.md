---
title: "Chapter 08. ECS vs EKS"
permalink: /aws-backend/part-09/08-ecs-vs-eks/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. ECS vs EKS
## 어떤 컨테이너 플랫폼을 선택할 것인가

> **학습 목표**
>
> - ECS와 EKS의 선택 기준을 설명할 수 있다.
> - 운영 복잡도와 유연성의 trade-off를 이해한다.
> - 팀 상황에 맞는 선택 기준을 세울 수 있다.
> - Spring Boot 서비스에 적합한 플랫폼을 근거와 함께 결정할 수 있다.

---

# 왜 플랫폼 선택 기준이 필요한가

ECS와 EKS는 모두 컨테이너를 배치하고 장애 시 복구하며 롤링 배포할 수 있으므로 기능 목록만으로 선택하면 운영 부담을 과소평가하기 쉽다.

서버 장애 자동 복구와 무중단 배포라는 목표는 같지만 객체, 권한, 네트워크와 업그레이드 책임은 다르다.

좋은 선택은 더 많은 기능을 가진 플랫폼이 아니라 현재 팀이 안정적으로 운영할 수 있는 플랫폼이다.

---

# ECS와 EKS란?

ECS는 AWS 전용 오케스트레이션 서비스이며 Task Definition, Task, Service와 Cluster를 중심으로 동작한다.

EKS는 AWS가 Kubernetes Control Plane을 관리하며 Deployment, ReplicaSet, Pod, Service와 Ingress를 사용한다.

두 서비스 모두 EC2 또는 Fargate를 사용할 수 있지만 운영 모델과 지원 범위는 동일하지 않다.

---

# 구조와 배포 흐름

```text
ECS
Image → Task Definition → ECS Service → Task → ALB

EKS
Image → Deployment → ReplicaSet → Pod ← Service ← Ingress/ALB
                         ▲
                  EKS Control Plane
```

ECS에서는 새 Task Definition Revision으로 Service를 갱신하고, EKS에서는 Deployment의 새 ReplicaSet으로 Pod를 교체한다.

---

# 핵심 비교표

| 기준 | ECS | EKS |
|---|---|---|
| 학습 곡선 | AWS 개념 중심으로 상대적으로 낮음 | Kubernetes 객체와 생태계 학습 필요 |
| 운영 부담 | 제어 구성 요소가 적음 | Add-on, 버전, 정책과 Node 운영 필요 |
| 생태계 | AWS 통합에 집중 | Helm, Operator 등 폭넓음 |
| 멀티 클라우드 | AWS 종속성이 큼 | API 이식성은 높지만 완전하지 않음 |
| 팀 규모 | 소규모 팀에도 적합 | 전담 플랫폼 역량이 있을수록 유리 |
| 배포 단위 | Task Definition Revision | Deployment와 Pod Template |
| 권한 | Task Role | IRSA와 ServiceAccount |
| 외부 HTTP | ALB와 Target Group | Ingress와 ALB Controller |
| Control Plane 비용 | 별도 EKS Control Plane 없음 | EKS Control Plane 시간 과금 |

ALB Annotation, IRSA와 VPC CNI 같은 AWS 통합은 다른 클라우드로 이동할 때 변경해야 하므로 Kubernetes가 멀티 클라우드를 자동 보장하지는 않는다.

---

# 언제 ECS를 고르는가

- AWS 중심으로 운영하고 Kubernetes 환경과의 일관성이 필요하지 않다.
- 작은 팀이 컨테이너 운영을 빠르고 단순하게 시작해야 한다.
- Task, Service와 ALB로 요구사항을 충분히 표현할 수 있다.
- Operator, CRD 또는 Helm 생태계가 필수 조건이 아니다.
- Spring Boot 비즈니스 기능 개발에 역량을 집중해야 한다.

ECS는 임시 선택이 아니며 AWS 전용 운영 모델이 요구에 맞으면 장기간 사용할 수 있다.

---

# 언제 EKS를 고르는가

- 조직에 Kubernetes 운영 경험과 전담 플랫폼 역량이 있다.
- 기존 Kubernetes 도구, 정책, Helm Chart 또는 Operator를 재사용해야 한다.
- 온프레미스나 여러 클라우드에서 공통 Kubernetes API를 운영 기준으로 삼는다.
- 복잡한 스케줄링과 확장 정책이 중요한 요구사항이다.
- 여러 팀에 표준화된 내부 플랫폼을 제공해야 한다.

단순히 더 강력해 보인다는 이유로 EKS를 고르면 업그레이드와 장애 대응 부담만 커질 수 있다.

---

# YAML 예시

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
      serviceAccountName: order-api
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-api
          image: repository/order-api:1.0.0
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

ECS의 대응 설정은 Task Definition의 Image·CPU·Memory·Task Role과 Service의 Desired Count 및 ALB Health Check에 나뉜다.

```bash
kubectl rollout status deployment/order-api
aws ecs wait services-stable --cluster production --services order-api
```

---

# Spring Boot에서는 어떻게 쓰는가

플랫폼과 관계없이 Image는 환경 중립적으로 만들고 Profile과 Secret을 실행 시점에 주입한다.

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

ECS는 `SPRING_PROFILES_ACTIVE`를 Task Definition 환경변수로, 비밀값을 Secrets Manager 참조로 제공한다.

EKS는 ConfigMap과 Secret을 사용하고 `terminationGracePeriodSeconds`와 Probe를 설정한다.

AWS 권한은 ECS Task Role 또는 EKS IRSA로 부여하고 장기 액세스 키를 넣지 않는다.

---

# 실무 의사결정 절차

1. 반드시 필요한 기능을 ECS에서 구현할 수 있는지 확인한다.
2. Kubernetes 전용 요구가 현재 요구인지 막연한 미래 가능성인지 구분한다.
3. 온콜 담당자가 배포, 네트워크와 권한 장애를 진단할 수 있는지 평가한다.
4. 대표 서비스로 배포·복구·확장과 업그레이드 절차를 검증한다.
5. 컴퓨팅, Control Plane, ALB, NAT와 관측성의 총비용을 비교한다.

이미 ECS를 안정적으로 운영한다면 명확한 이익 없이 EKS로 이전하지 않는다.

---

# 장애 사례와 주의할 점

ECS의 `CannotPullContainerError`와 EKS의 `ImagePullBackOff`는 이미지 URI, 권한과 네트워크 경로부터 확인한다.

ECS 종료 코드 `137`과 EKS `OOMKilled`는 메모리 설정과 JVM 사용량을 분석한다.

readiness 또는 ALB Health Check가 없으면 준비되지 않은 인스턴스로 요청이 전달될 수 있다.

자원 제한이 없으면 EC2 Capacity 또는 Kubernetes Node 전체가 불안정해질 수 있다.

롤링 업데이트 중 세션과 연결이 끊기므로 외부 세션 저장소, Graceful Shutdown과 재시도가 필요하다.

EKS에서는 `kubectl logs --previous`와 `kubectl describe pod`, ECS에서는 Service Event와 Stopped Reason을 확인한다.

---

# 비용과 성능 고려사항

EKS는 Control Plane 시간 과금과 EC2 Node 또는 Fargate 비용이 발생하고 ECS도 선택한 EC2 또는 Fargate 비용이 발생한다.

Ingress마다 ALB를 만들면 ALB 수가 늘어나며 Ingress Group으로 공유할 수 있지만 보안 경계를 함께 고려한다.

두 플랫폼 모두 NAT Gateway 트래픽, ECR, ALB/NLB와 로그·지표 수집이 비용에 포함된다.

비용 비교는 동일한 요청량, 가용성, 배포 여유 용량과 운영 인력 시간을 포함해야 한다.

---

# 기억해야 할 내용

- ECS와 EKS 모두 복구, 확장과 롤링 배포를 지원한다.
- ECS는 AWS 중심의 단순한 운영 모델이 강점이다.
- EKS는 Kubernetes 생태계가 강점이지만 운영 부담이 크다.
- 멀티 클라우드는 Kubernetes API만으로 자동 달성되지 않는다.
- 권한은 ECS Task Role 또는 EKS IRSA로 최소화한다.
- 선택에는 팀 역량과 온콜 책임을 포함해야 한다.
- 더 복잡한 플랫폼이 항상 더 좋은 플랫폼은 아니다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 09. Part 9 Summary**이다.

Kubernetes 객체와 EKS 운영 경계, ECS와 EKS 선택 기준을 정리한 뒤 Part 10 Serverless로 이어진다.


