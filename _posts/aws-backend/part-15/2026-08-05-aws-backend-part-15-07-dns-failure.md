---
title: "Chapter 07. DNS Failure"
permalink: /aws-backend/part-15/07-dns-failure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. DNS Failure
## 권한 서버와 캐시 계층 진단

> **학습 목표**
>
> - authoritative 응답과 recursive cache 응답을 구분할 수 있다.
> - NXDOMAIN과 SERVFAIL을 증거로 분석할 수 있다.
> - Route 53 Alias, CNAME, TTL과 JVM DNS cache를 진단할 수 있다.

---

# 실제 장애 징후

사용자는 도메인을 찾을 수 없거나 일부 네트워크에서만 이전 주소로 접속한다.

애플리케이션 로그에는 `UnknownHostException`, name resolution timeout 또는 downstream 연결 실패가 나타날 수 있다.

변경 직후 일부 resolver는 새 값을 반환하고 다른 resolver는 TTL이 남은 이전 값을 반환할 수 있다.

NXDOMAIN은 이름이 존재하지 않는다는 응답이고 SERVFAIL은 resolver가 유효한 응답을 완성하지 못했다는 뜻이다.

---

# 정의와 가능한 원인

authoritative DNS 서버는 zone의 원본 레코드에 대한 권한 있는 응답을 제공한다.

recursive resolver는 client를 대신해 조회하고 TTL 동안 결과 또는 negative response를 cache한다.

잘못된 record name, type, hosted zone 또는 위임은 NXDOMAIN과 잘못된 응답을 만든다.

DNSSEC 검증 실패, 위임 오류, 권한 서버 접근 문제는 SERVFAIL 원인이 될 수 있다.

긴 TTL은 안정적인 cache 효율을 주지만 변경 전 계획하지 않으면 전환 시간을 늘린다.

CNAME은 다른 이름을 가리키며 DNS 표준상 zone apex에 일반 CNAME을 둘 수 없다.

Route 53 Alias는 지원 AWS 리소스를 zone apex에도 연결할 수 있는 Route 53 기능이다.

JVM과 OS 및 로컬 resolver는 각각 DNS 결과를 cache할 수 있다.

---

# 계층 구조

```text
Spring Boot JVM cache
        |
OS stub resolver
        |
Recursive resolver cache
        |
Root and TLD delegation
        |
Authoritative Route 53 zone
        |
Alias / CNAME / A / AAAA
```

같은 이름도 질의한 resolver와 시각에 따라 cache 상태가 달라질 수 있다.

---

# 증거 기반 진단 순서

## 1. 영향을 확인한다

영향을 받는 도메인, record type, 지역, ISP, VPC와 애플리케이션 인스턴스를 구분한다.

사용자가 받은 정확한 응답 코드와 IP 및 관찰 시각을 기록한다.

## 2. 최근 변경을 확인한다

record, hosted zone, registrar의 name server, DNSSEC, Alias 대상과 TTL 변경을 확인한다.

변경 전 TTL을 낮추지 않았다면 기존 cache가 만료될 때까지 혼재할 수 있음을 고려한다.

## 3. 여러 경계에서 조회한다

```bash
dig api.example.com A
dig api.example.com A +noall +answer +authority
dig @1.1.1.1 api.example.com A
dig @8.8.8.8 api.example.com A
```

public resolver 결과와 VPC resolver 결과를 비교해 split-horizon 또는 private hosted zone 영향을 찾는다.

## 4. authoritative 서버에 직접 묻는다

```bash
dig example.com NS +short
dig @ns-123.awsdns-45.net api.example.com A +noall +answer +authority
```

실제 zone에 지정된 name server를 사용하고 registrar 위임과 Route 53 NS가 일치하는지 확인한다.

authoritative 응답은 정상인데 recursive 결과만 오래되었다면 TTL과 negative cache를 조사한다.

## 5. 위임 경로를 확인한다

```bash
dig +trace api.example.com A
```

`+trace` 결과에서 root, TLD, zone 위임과 최종 권한 응답이 어디에서 끊기는지 확인한다.

사내 방화벽이 iterative query 결과에 영향을 줄 수 있으므로 다른 네트워크 결과와 비교한다.

## 6. Route 53 설정을 확인한다

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID"
aws route53 get-hosted-zone \
  --id "$HOSTED_ZONE_ID"
```

record name의 끝점, type, Alias 대상, EvaluateTargetHealth와 public 또는 private zone을 확인한다.

동일 이름의 private hosted zone이 VPC에서 public 응답을 가릴 수 있다.

## 7. 응답 코드를 구분한다

NXDOMAIN이면 정확한 이름, zone 존재, 위임과 negative TTL을 확인한다.

SERVFAIL이면 DNSSEC validation, 위임 loop, 권한 서버 응답과 resolver 상태를 확인한다.

NOERROR인데 answer가 없으면 요청 type에 해당 레코드가 없는 NODATA 상황을 구분한다.

## 8. 애플리케이션 cache를 확인한다

동일 호스트의 `dig` 결과와 실행 중 JVM의 실제 연결 대상을 비교한다.

JVM DNS cache TTL과 HTTP client connection pool이 이전 주소 연결을 얼마나 유지하는지 확인한다.

## 9. 안전하게 완화한다

잘못된 record 변경은 검증된 값으로 되돌리고 TTL 전파 상태를 계속 관찰한다.

애플리케이션에 IP를 하드코딩하면 failover와 주소 변경을 깨뜨리므로 임시 대응으로도 신중해야 한다.

---

# Spring Boot 관찰 포인트

`UnknownHostException`에는 host, resolver 실패 시각과 호출 dependency를 기록하되 secret을 남기지 않는다.

JVM의 positive 및 negative DNS cache 정책은 JDK와 보안 설정을 기준으로 확인한다.

connection pool은 DNS를 다시 조회해도 기존 keep-alive 연결을 재사용할 수 있다.

DNS lookup 시간과 downstream connect 시간을 trace에서 분리하면 이름 해석과 network 장애를 구분하기 쉽다.

---

# 대응과 복구

권한 레코드를 수정한 뒤 authoritative 응답부터 검증하고 주요 recursive resolver의 cache 만료를 관찰한다.

잘못된 위임은 registrar와 hosted zone의 NS를 일치시키고 전체 trace를 다시 확인한다.

복구 후 JVM과 장기 실행 client가 새 endpoint로 연결하는지 확인한다.

---

# 재발 방지

DNS 변경에는 현재 TTL을 고려한 사전 계획, 검증 질의와 롤백 값을 포함한다.

public과 private hosted zone의 소유자와 목적을 문서화한다.

핵심 도메인은 여러 위치에서 resolution과 실제 HTTPS 성공을 합성 모니터링한다.

DNSSEC와 위임 변경은 별도 변경 검토와 단계적 검증을 거친다.

---

# 기억해야 할 내용

- authoritative 원본과 recursive cache를 구분한다.
- NXDOMAIN, SERVFAIL과 NODATA는 의미와 조사 경로가 다르다.
- `dig +trace`는 위임 경로를 확인하는 도구다.
- Alias와 CNAME은 같은 개념이 아니다.
- JVM 및 connection pool cache까지 확인해야 복구를 검증할 수 있다.

---

# 다음 Chapter

다음 장에서는 [Network 장애](/aws-backend/part-15/08-network-failure/)를 분석한다.
