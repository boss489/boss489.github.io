---
title: "Chapter 05. Ingress"
permalink: /aws-backend/part-09/05-ingress/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Ingress
## HTTP 요청을 Service로 라우팅하기

> **학습 목표**
>
> - Ingress의 역할을 설명할 수 있다.
> - Host와 Path 기반 라우팅을 이해한다.
> - Ingress Controller가 필요한 이유를 설명할 수 있다.
> - ALB용 Ingress 매니페스트를 작성할 수 있다.

---

# 왜 Ingress가 필요한가

외부에 공개할 Spring Boot Service가 늘어날 때마다 별도 LoadBalancer를 만들면 진입점, 인증서, DNS와 비용 관리가 복잡해진다.

`api.example.com/orders`와 `api.example.com/payments`를 서로 다른 Service로 보내려면 HTTP 내용을 이해하는 공통 라우팅 계층이 필요하다.

Ingress는 여러 서비스의 Host와 Path 규칙을 하나의 선언으로 관리해 외부 진입점을 통합한다.

---

# Ingress란?

Ingress는 외부 HTTP/HTTPS 요청을 클러스터 내부 Service로 라우팅하는 규칙이다.

Ingress 객체 자체는 규칙일 뿐이며 실제 트래픽을 처리하지 않는다.

실제 구현은 `ingressClassName`으로 선택한 Ingress Controller가 담당한다.

---

# 구조와 동작 흐름

```text
Internet
   │ HTTPS
   ▼
Ingress Controller / ALB
   │ host=api.example.com
   ├── /orders/*   → order-api Service   → Pods
   └── /payments/* → payment-api Service → Pods
```

1. 사용자가 Ingress 규칙을 API Server에 생성한다.
2. Controller가 자신이 담당하는 Ingress를 감지한다.
3. Controller가 로드밸런서, Listener, Rule과 Target Group을 조정한다.
4. 요청의 Host와 Path에 일치하는 Service로 전달한다.
5. Service가 준비된 Pod Endpoint로 요청을 보낸다.

---

# 라우팅 방식 비교

| 방식 | 조건 예 | 활용 |
|---|---|---|
| Host 기반 | `orders.example.com` | 도메인별 서비스 분리 |
| Path 기반 | `/orders/*` | 한 도메인의 API 분리 |
| 기본 백엔드 | 규칙 불일치 | 404 처리 서비스 |
| Service LoadBalancer | TCP/UDP 포트 | L4 외부 공개 |

Ingress는 주로 HTTP/HTTPS를 다루며 임의 TCP/UDP 노출은 Controller별 별도 기능이나 LoadBalancer Service가 필요하다.

---

# YAML 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-api
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789012:certificate/example
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/readiness
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
          - path: /payments
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80
```

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
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress backend-api
```

---

# Spring Boot에서는 어떻게 쓰는가

ALB Health Check와 Kubernetes readinessProbe를 Actuator readiness Endpoint에 연결하되 보안 필터가 Health Check를 차단하지 않게 한다.

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

ALB가 전달하는 `X-Forwarded-Proto`와 `X-Forwarded-For`를 처리해야 HTTPS Redirect와 클라이언트 주소가 올바르게 동작한다.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    @GetMapping("/{orderId}")
    public OrderResponse findOrder(@PathVariable Long orderId) {
        return new OrderResponse(orderId);
    }
}
```

ConfigMap으로 `SPRING_PROFILES_ACTIVE`를 주입하고 Secret으로 비밀값을 제공하며 AWS 접근은 IRSA를 사용한다.

Deployment에는 readiness/liveness Probe, 자원 requests/limits와 `terminationGracePeriodSeconds`를 설정한다.

---

# 실무에서는 어떻게 사용할까

공개 API는 HTTPS Listener와 ACM 인증서를 사용하고 HTTP는 HTTPS로 Redirect하도록 구성한다.

Ingress Group을 이용하면 여러 Namespace 또는 팀의 Ingress가 ALB 하나를 공유할 수 있지만 규칙 변경 권한과 신뢰 경계를 엄격히 관리해야 한다.

DNS는 Route 53 Alias로 ALB를 가리키고, 외부 공개가 필요 없는 API는 `internal` Scheme을 사용한다.

---

# 장애 사례

Ingress 주소가 생성되지 않으면 Controller 로그, `ingressClassName`, Annotation, IAM 권한과 Subnet 태그를 확인한다.

Service 이름이나 포트가 틀리면 ALB Target이 Healthy가 되지 않고 503이 발생할 수 있다.

readinessProbe가 없거나 ALB Health Check 경로가 인증을 요구하면 준비되지 않은 Pod로 트래픽이 가거나 Target이 계속 Unhealthy가 된다.

```bash
kubectl describe ingress backend-api
kubectl logs -n kube-system deployment/aws-load-balancer-controller
kubectl get service,endpointslice
```

롤링 업데이트 중 세션과 연결이 끊기면 외부 세션 저장소, 종료 유예와 Deregistration Delay의 관계를 점검한다.

---

# 주의할 점

- `pathType`과 애플리케이션 Context Path를 일치시킨다.
- 인터넷 공개 여부와 Security Group 허용 범위를 검토한다.
- Annotation은 Controller 구현에 종속되므로 버전별 문서를 확인한다.
- Ingress Group 공유 시 다른 팀이 규칙 우선순위에 영향을 줄 수 있다.
- Pod IP에 의존하지 않고 Service를 백엔드로 지정한다.

---

# 비용과 성능 고려사항

Ingress마다 ALB를 만들면 ALB 개수와 시간·처리량 기반 과금 요소가 늘어나므로 적절한 그룹핑을 검토한다.

규칙과 서비스가 지나치게 많으면 운영 복잡도와 변경 영향 범위가 커지므로 보안 경계와 비용 절감 사이를 조정한다.

Private Subnet의 이미지 다운로드와 외부 호출은 NAT Gateway 데이터 처리 비용을 만들 수 있다.

긴 연결과 큰 응답은 ALB Timeout, Spring Boot Timeout과 연결 풀 설정을 함께 측정해야 한다.

---

# 기억해야 할 내용

- Ingress는 외부 HTTP/HTTPS 요청의 Host와 Path 라우팅 규칙이다.
- Ingress 자체는 트래픽을 처리하지 않으며 Controller가 필요하다.
- AWS에서는 `ingressClassName: alb`로 AWS Load Balancer Controller를 선택한다.
- 요청은 Ingress, Service, Pod 순서로 전달된다.
- TLS 인증서, Scheme, Target Type은 Annotation으로 구성할 수 있다.
- Health Check와 readinessProbe 경로를 함께 검증해야 한다.
- ALB 공유는 비용을 줄이지만 권한과 장애 범위를 넓힐 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 [AWS Load Balancer Controller](/aws-backend/part-09/06-alb-controller/)가 Ingress 선언을 실제 ALB로 변환하는 과정을 학습한다.

IAM 권한, Target Type과 AWS 리소스 조정 흐름을 자세히 살펴본다.


