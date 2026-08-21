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
> - Spring Boot Pod의 상태 검사와 종료 방식을 구성할 수 있다.

---

# 왜 Pod가 필요한가

Kubernetes가 컨테이너 하나만 직접 관리하면 애플리케이션과 로그 수집기처럼 함께 배치되어야 하는 컨테이너의 생명주기를 하나로 묶기 어렵다.

수십 개 컨테이너를 서버에 직접 배치하면 포트 충돌, 재시작, 네트워크 주소 변경과 자원 할당을 운영자가 일일이 처리해야 한다.

Pod는 함께 실행되어야 하는 컨테이너를 하나의 스케줄링 단위로 만들고, 장애 시 새 인스턴스로 교체할 수 있게 한다.

---

# Pod란?

Pod는 Kubernetes에서 컨테이너를 실행하는 최소 단위다.

보통 하나의 Pod에 하나의 애플리케이션 컨테이너를 둔다.

같은 Pod의 컨테이너는 동일한 네트워크 Namespace를 사용하므로 `localhost`로 통신하고 같은 볼륨을 마운트할 수 있다.

Pod는 수정하며 오래 쓰는 서버가 아니라 선언으로 다시 생성할 수 있는 일회성 자원이다.

---

# 구조와 생명주기

```text
Node
└── Pod (IP: 10.0.1.23)
    ├── Spring Boot :8080
    ├── Sidecar :15000
    └── Shared Volume
```

1. API Server가 Pod 선언을 저장한다.
2. Scheduler가 자원 요청량과 제약 조건을 보고 Node를 선택한다.
3. Node의 kubelet이 이미지를 받아 컨테이너를 시작한다.
4. Probe와 컨테이너 종료 상태가 Pod 상태에 반영된다.
5. Pod가 삭제되면 Deployment가 새 Pod를 만들 수 있으며 IP도 달라진다.

| 상태 | 의미 |
|---|---|
| Pending | 스케줄링 또는 이미지 준비 중 |
| Running | 하나 이상의 컨테이너가 실행 중 |
| Succeeded | 모든 컨테이너가 정상 종료 |
| Failed | 하나 이상의 컨테이너가 실패 종료 |
| Unknown | Node와 상태 통신 불가 |

---

# Pod와 다른 실행 객체 비교

| 객체 | 목적 | 적합한 사용 |
|---|---|---|
| Pod | 컨테이너 실행 단위 | 일회성 진단 |
| Deployment | 무상태 Pod 배포 | Spring Boot API |
| StatefulSet | 순서·이름·볼륨 안정성 | 데이터 저장 시스템 |
| DaemonSet | Node마다 Pod 실행 | 로그·보안 에이전트 |
| Job | 완료될 때까지 실행 | 마이그레이션·배치 |

---

# YAML 예시

직접 만든 Pod는 복제와 롤백 기능이 없으므로 학습과 진단에 주로 사용한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-api
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
      env:
        - name: SPRING_PROFILES_ACTIVE
          valueFrom:
            configMapKeyRef:
              name: order-api-config
              key: profile
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
kubectl apply -f pod.yaml
kubectl get pod order-api -o wide
kubectl describe pod order-api
kubectl logs order-api
```

---

# Spring Boot에서는 어떻게 쓰는가

Actuator의 liveness는 프로세스가 복구 불가능한 상태인지, readiness는 현재 요청을 받을 준비가 되었는지 판단하는 데 사용한다.

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

`terminationGracePeriodSeconds`는 Spring Boot가 진행 중 요청을 마치도록 `server.shutdown=graceful`의 종료 시간보다 충분히 길게 잡는다.

ConfigMap과 Secret을 환경변수로 주입해 `SPRING_PROFILES_ACTIVE`와 접속 정보를 설정한다.

```java
@ConfigurationProperties(prefix = "storage")
public record StorageProperties(String bucketName) {
}
```

S3 같은 AWS API 호출은 IRSA로 ServiceAccount에 최소 IAM 권한을 연결하며 액세스 키를 Secret에 저장하지 않는다.

---

# 실무에서는 어떻게 사용할까

Spring Boot Pod는 상태를 내부 파일에 보존하지 않고 데이터베이스, Redis, S3 같은 외부 저장소를 사용해야 자유롭게 교체할 수 있다.

로그는 표준 출력으로 기록하고 Node의 수집 에이전트가 CloudWatch Logs 같은 중앙 저장소로 전달하게 한다.

초기화 작업은 `initContainer`, 보조 기능은 Sidecar를 사용할 수 있지만 컨테이너가 많아지면 자원과 장애 원인도 함께 복잡해진다.

---

# 장애 사례

`CrashLoopBackOff`는 애플리케이션 시작 실패, 잘못된 환경변수, Probe 실패가 반복될 때 나타난다.

```bash
kubectl logs order-api --previous
kubectl describe pod order-api
```

`ImagePullBackOff`는 잘못된 이미지 URI·태그, ECR 권한, DNS 또는 NAT 경로 문제를 확인한다.

`OOMKilled`는 컨테이너가 메모리 제한을 초과했다는 뜻이므로 실제 사용량, JVM 힙과 Native Memory, `limits`를 함께 분석한다.

readinessProbe가 없으면 시작 중인 Pod가 Service Endpoint에 포함되어 사용자 요청이 실패할 수 있다.

---

# 주의할 점

- Pod IP는 재생성 시 바뀌므로 IP에 직접 의존하지 않는다.
- 컨테이너 파일 시스템은 영구 저장소로 간주하지 않는다.
- livenessProbe에 외부 DB 상태를 무조건 포함하면 DB 장애가 Pod 재시작 폭풍으로 확대될 수 있다.
- CPU·메모리 요청량과 제한을 측정 없이 지나치게 낮게 설정하지 않는다.
- 운영 애플리케이션은 Pod 대신 Deployment로 관리한다.

---

# 비용과 성능 고려사항

작은 Pod를 과도하게 늘리면 kubelet, 네트워크, 로그 및 모니터링 오버헤드가 커진다.

자원 요청량이 실제보다 크면 Node 이용률이 낮아지고, 너무 작으면 과밀 배치와 성능 편차가 생긴다.

EKS에서는 Pod가 사용하는 EC2 Node 또는 Fargate 자원, 이미지 다운로드와 NAT Gateway 데이터 처리 비용을 함께 고려한다.

---

# 기억해야 할 내용

- Pod는 Kubernetes의 최소 스케줄링 및 실행 단위다.
- 같은 Pod의 컨테이너는 네트워크와 볼륨을 공유한다.
- Pod는 교체 가능한 일회성 자원이며 IP도 변할 수 있다.
- 운영 Pod는 Deployment 같은 상위 Controller로 관리한다.
- readiness와 liveness는 목적이 다르다.
- 자원 설정은 안정적인 스케줄링과 장애 격리의 기준이다.
- Spring Boot의 Graceful Shutdown과 종료 유예 시간을 함께 설정한다.

---

# 다음 Chapter

다음 Chapter에서는 [ReplicaSet과 Deployment](/aws-backend/part-09/03-replicaset-deployment/)를 학습한다.

Pod 개수를 자동으로 유지하고 새 버전을 롤링 업데이트하거나 이전 버전으로 롤백하는 흐름을 살펴본다.

- 고유 IP를 가진다.
- 죽으면 새 Pod로 교체될 수 있다.
- 상태를 오래 보존하는 단위가 아니다.
- 로그와 파일은 외부 저장소로 보내야 한다.

---

# 기억해야 할 내용

Pod는 일회성 실행 단위다.

직접 Pod를 오래 관리하기보다 Deployment로 관리한다.


