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
> - Spring Boot Service의 포트와 상태 검사를 구성할 수 있다.

---

# 왜 Service가 필요한가

Pod는 장애 복구와 배포 과정에서 계속 교체되며 새 Pod는 이전과 다른 IP를 받을 수 있다.

호출자가 수십 개 Pod IP를 직접 추적하면 재시작할 때마다 설정을 바꿔야 하고 준비되지 않은 Pod에도 요청이 전달될 수 있다.

Service는 동적으로 변하는 Pod 집합 앞에 안정적인 이름을 제공하고 준비된 Pod로 트래픽을 분산한다.

---

# Service란?

Service는 여러 Pod 앞에 고정된 접근 지점을 제공하는 Kubernetes 객체다.

Pod는 죽고 다시 생성되면 IP가 바뀔 수 있다.

Service는 Label Selector로 Pod를 찾아 트래픽을 보낸다.

클러스터 DNS를 통해 `order-api.default.svc.cluster.local` 같은 이름으로 접근할 수 있으며 일반적으로 같은 Namespace에서는 `order-api`만 사용한다.

---

# 구조와 동작 흐름

```text
Client
  │ order-api:80
  ▼
Service (stable ClusterIP/DNS)
  │ label: app=order-api
  ├──────────┬──────────┐
  ▼          ▼          ▼
Pod :8080  Pod :8080  Pod :8080
```

1. Service의 Selector가 일치하는 Pod를 찾는다.
2. 준비 상태를 통과한 Pod IP가 EndpointSlice에 반영된다.
3. 클라이언트는 Service DNS와 포트로 요청한다.
4. 클러스터 네트워크가 선택된 Endpoint로 트래픽을 전달한다.

`port`는 Service가 노출하는 포트이고 `targetPort`는 컨테이너가 실제로 수신하는 포트다.

---

# Service 타입 비교

| 타입 | 접근 범위 | 외부 리소스 | 대표 용도 |
|---|---|
| ClusterIP | 클러스터 내부 | 없음 | 서비스 간 통신 |
| NodePort | 모든 Node의 고정 포트 | 없음 | 제한적 테스트·기반 연결 |
| LoadBalancer | 외부 또는 내부 | 클라우드 LB | TCP/UDP 공개 |
| ExternalName | DNS 별칭 | 없음 | 외부 DNS 연결 |

AWS Load Balancer Controller는 `type: LoadBalancer` Service를 관찰해 NLB를 프로비저닝할 수 있다.

HTTP Host와 Path 라우팅이 필요하면 Service마다 LoadBalancer를 만들기보다 Ingress와 ALB를 사용한다.

---

# YAML 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-api
spec:
  type: ClusterIP
  selector:
    app: order-api
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

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
          image: repository/order-api:1.0.0
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
kubectl apply -f service.yaml
kubectl get service,endpointslice
kubectl port-forward service/order-api 8080:80
```

---

# Spring Boot에서는 어떻게 쓰는가

Actuator readiness가 성공한 Pod만 Service Endpoint에 남도록 Probe를 연결한다.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
server:
  shutdown: graceful
```

ConfigMap과 Secret으로 환경을 분리하고 호출 대상은 Pod IP가 아니라 Service DNS로 설정한다.

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    valueFrom:
      configMapKeyRef:
        name: order-api-config
        key: profile
  - name: PAYMENT_BASE_URL
    value: http://payment-api
```

S3나 SQS 접근은 IRSA가 연결된 ServiceAccount를 Deployment에 지정해 Pod별 최소 권한을 부여한다.

종료 시 Endpoint 제거와 진행 중 요청 처리를 위해 `server.shutdown=graceful`과 `terminationGracePeriodSeconds`를 함께 설정한다.

---

# 실무에서는 어떻게 사용할까

내부 Spring Boot API는 기본적으로 ClusterIP를 사용하고 외부 HTTP 진입점은 Ingress에 모은다.

Service 이름은 환경에 독립적인 논리 주소가 되므로 애플리케이션 설정과 서비스 디스커버리를 단순화한다.

Selector와 Deployment Label은 공통 규칙으로 관리하고 `kubectl get endpointslice`로 실제 연결 대상을 확인한다.

---

# 장애 사례와 주의할 점

Service에 Endpoint가 없으면 Selector와 Pod Label 불일치, readinessProbe 실패, Namespace 차이를 확인한다.

```bash
kubectl describe service order-api
kubectl get pod -l app=order-api --show-labels
kubectl get endpointslice -l kubernetes.io/service-name=order-api
```

`targetPort`가 Spring Boot의 `server.port`와 다르면 연결 거부 또는 502가 발생한다.

readinessProbe가 없으면 초기화 중인 Pod가 Endpoint에 포함되어 요청이 실패할 수 있다.

Pod IP를 직접 저장하면 롤링 업데이트 후 연결이 끊기므로 반드시 Service DNS를 사용한다.

장시간 연결은 Pod 종료 시 끊길 수 있으므로 재연결, 종료 유예와 외부 세션 저장소를 적용한다.

---

# 비용과 성능 고려사항

ClusterIP 자체보다 이를 외부에 공개하는 NLB·ALB와 데이터 처리 비용이 주요 과금 요소다.

Service마다 `LoadBalancer`를 만들면 로드밸런서가 늘어나므로 HTTP 서비스는 Ingress 공유를 검토한다.

다른 AZ로 전달되는 트래픽과 NAT Gateway를 통과하는 외부 호출은 데이터 전송 및 처리 비용에 영향을 준다.

자원 requests와 limits가 없으면 과밀 Node에서 Endpoint 응답 시간이 불안정해질 수 있다.

---

# 기억해야 할 내용

- Service는 변하는 Pod 앞에 안정적인 가상 IP와 DNS를 제공한다.
- Label Selector가 Service의 대상 Pod를 결정한다.
- 준비된 Pod가 EndpointSlice에 반영된다.
- ClusterIP는 내부 통신의 기본 타입이다.
- LoadBalancer Service는 AWS에서 NLB와 연결할 수 있다.
- `port`와 `targetPort`의 의미를 구분해야 한다.
- 외부 HTTP 라우팅은 Ingress와 함께 구성한다.

---

# 다음 Chapter

다음 Chapter에서는 외부 HTTP 요청을 여러 Service로 분기하는 [Ingress](/aws-backend/part-09/05-ingress/)를 학습한다.

Host와 Path 규칙이 하나의 진입점을 여러 백엔드에 연결하는 방식을 살펴본다.


