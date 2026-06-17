---
title: "쿠버네티스 네트워킹과 서비스 디스커버리"
categories:
  - kubernetes
tags:
  - kubernetes
  - network
  - dns
last_modified_at: 2026-06-17T09:00:00+09:00
---

# 쿠버네티스 네트워킹과 서비스 디스커버리

쿠버네티스를 처음 접하면 Pod, Service, DNS가 각각 따로 동작하는 것처럼 보이지만, 실제 운영에서는 이 세 가지를 함께 이해해야 애플리케이션 통신 구조가 선명해진다. 이 글에서는 Pod 간 통신, 네임스페이스, DNS 기반 서비스 검색을 중심으로 정리한다.

## 쿠버네티스 네트워크의 기본 전제

쿠버네티스는 다음 두 가지를 기본 가정으로 둔다.

1. 모든 Pod는 서로 직접 통신 가능한 IP를 가진다.
2. Node 간 Pod 통신에서도 NAT 없이 라우팅 가능해야 한다.

즉, 컨테이너 단위가 아니라 Pod 단위가 네트워크의 기본 엔드포인트다. 하나의 Pod 안에 여러 컨테이너가 있더라도 네트워크 네임스페이스를 공유하므로 같은 IP와 포트를 사용한다.

## Pod와 Service의 역할 분리

Pod는 실행 단위이고, Service는 접근 지점이다.

- Pod: 생성과 삭제가 빈번하며 IP가 바뀔 수 있음
- Service: 고정된 이름과 가상 IP를 제공

애플리케이션은 특정 Pod IP를 직접 바라보지 않고, 일반적으로 Service 이름을 통해 통신한다. 이 추상화 덕분에 Pod가 재생성되어도 클라이언트 설정을 바꿀 필요가 없다.

## DNS 서비스와 서비스 검색

쿠버네티스 클러스터 내부에는 DNS 서비스가 있으며, 예전에는 `kube-dns`, 최근에는 보통 `CoreDNS`가 사용된다. 역할은 동일하다. Service 이름을 DNS 레코드로 해석해서 Pod 간 통신을 가능하게 만든다.

예를 들어 `database`라는 Service가 있다면 같은 네임스페이스의 애플리케이션은 아래처럼 접근할 수 있다.

```bash
mysql -h database -uroot -ppassword fleetman
```

여기서 `database`는 고정 IP가 아니라 DNS 이름이다. 실제로는 DNS가 Service ClusterIP로 해석하고, Service가 적절한 Pod로 트래픽을 분산한다.

## 네임스페이스의 의미

네임스페이스는 쿠버네티스 리소스를 논리적으로 구분하는 단위다. Java의 패키지와 비슷하게 생각할 수 있지만, 실제 운영에서는 다음과 같은 의미가 더 크다.

- 팀 또는 서비스별 리소스 격리
- 동일한 이름의 리소스 공존
- 접근 제어와 쿼터 관리의 기준

별도로 지정하지 않으면 대부분 `default` 네임스페이스를 사용한다. 하지만 운영 환경에서는 `frontend`, `backend`, `monitoring`처럼 목적에 따라 분리하는 것이 일반적이다.

## 자주 사용하는 조회 명령어

```bash
# 네임스페이스 목록 조회
kubectl get namespaces
kubectl get ns

# 특정 네임스페이스의 Pod 조회
kubectl get po -n kube-system

# DNS 서비스 상세 조회
kubectl describe svc kube-dns -n kube-system
```

환경에 따라 DNS 서비스 이름이 `coredns`로 보일 수도 있다.

## Pod 내부에서 네트워크 확인하기

문제가 생기면 애플리케이션 코드보다 먼저 클러스터 내부에서 이름 해석과 TCP 연결이 되는지 확인해야 한다.

```bash
# Pod에 접속
kubectl exec -it webapp-7bf4db7f75-n6xc2 -- sh

# Alpine 계열 이미지에서 클라이언트 설치
apk update
apk add mysql-client

# Service 이름으로 DB 접속
mysql -h database -uroot -ppassword fleetman
```

이 과정에서 확인할 수 있는 것은 다음과 같다.

- DNS 이름이 정상 해석되는지
- 대상 Service가 살아 있는지
- Service 뒤의 Pod가 요청을 처리하는지
- 네트워크 정책이나 방화벽에 막히지 않는지

## Service 이름 해석 규칙

같은 네임스페이스라면 보통 짧은 이름만으로 충분하다.

- `database`

다른 네임스페이스에 있는 Service라면 FQDN 형태를 사용할 수 있다.

- `database.backend.svc.cluster.local`

이 규칙을 이해하면 멀티 네임스페이스 환경에서 통신 문제를 훨씬 빠르게 진단할 수 있다.

## 실무에서 자주 헷갈리는 부분

### 1. Pod IP에 직접 연결하는 습관
Pod IP는 영속적이지 않다. Deployment 롤링 업데이트나 장애 복구 후 바뀔 수 있으므로 애플리케이션 설정에는 Service 이름을 사용하는 것이 맞다.

### 2. Pod = 컨테이너라는 단순화
학습 단계에서는 이해를 위해 그렇게 설명하기도 하지만, 실제로는 하나의 Pod에 여러 컨테이너가 들어갈 수 있다. 사이드카 패턴을 쓰는 순간 이 차이가 매우 중요해진다.

### 3. DNS 문제와 애플리케이션 문제의 혼동
서비스 연결 실패가 발생했을 때 애플리케이션 설정만 확인하면 늦다. `nslookup`, `dig`, `curl`, DB 클라이언트 등으로 클러스터 내부 네트워크를 먼저 검증해야 한다.

## 정리

쿠버네티스 네트워킹의 핵심은 "Pod는 변하고, Service 이름은 유지된다"는 점이다.

- Pod는 실행 단위
- Service는 안정적인 접근 지점
- DNS는 Service 이름을 실제 네트워크 경로로 연결하는 수단

이 구조를 이해하면 마이크로서비스 간 통신, 장애 진단, 운영 자동화까지 훨씬 수월해진다.
