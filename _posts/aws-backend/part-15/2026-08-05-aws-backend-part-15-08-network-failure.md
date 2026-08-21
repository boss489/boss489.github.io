---
title: "Chapter 08. Network Failure"
permalink: /aws-backend/part-15/08-network-failure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Network Failure
## 경로와 정책의 양방향 진단

> **학습 목표**
>
> - Security Group과 NACL의 상태 처리 차이를 설명할 수 있다.
> - route, IGW, NAT와 ephemeral port 문제를 진단할 수 있다.
> - Reachability Analyzer와 VPC Flow Logs를 증거로 사용할 수 있다.

---

# 실제 장애 징후

연결 timeout, connection refused, 간헐적 reset과 특정 AZ 또는 subnet의 통신 실패가 나타난다.

애플리케이션은 DNS를 정상 해석하지만 TCP 연결을 만들지 못할 수 있다.

외부 API 호출만 실패하거나 private subnet의 특정 목적지로만 나가지 못할 수 있다.

NAT 경로의 포트 또는 connection 자원이 포화되면 신규 연결 실패가 증가할 수 있다.

---

# 정의와 가능한 원인

Network 장애는 이름이 주소로 변환된 이후 packet이 목적지에 도달하고 응답이 돌아오는 경로의 실패다.

Security Group은 stateful이므로 허용된 연결의 응답 traffic을 별도 규칙 없이 추적한다.

NACL은 stateless이므로 요청과 응답 방향을 각각 허용해야 한다.

잘못된 route table, IGW 미연결, NAT 경로 누락과 subnet 연결 오류가 packet 전달을 막는다.

public IPv4 통신에는 public 주소, 올바른 route와 IGW 조건이 함께 필요하다.

private subnet의 outbound internet 통신은 일반적으로 NAT와 그 NAT가 사용하는 public 경로가 필요하다.

client의 ephemeral source port와 반환 traffic 범위를 NACL이 막으면 한 방향 규칙이 정상이어도 연결이 실패한다.

connection refused는 경로가 도달했지만 해당 port에 listener가 없거나 능동 거부되었을 가능성을 시사한다.

---

# 계층 구조

```text
Application socket
      |
ENI and Security Group
      |
Subnet NACL
      |
Route Table
      |
IGW / NAT / Transit path
      |
Destination policy
      |
Application listener
```

정방향 허용뿐 아니라 응답 packet이 돌아오는 역방향 경로도 검증해야 한다.

---

# 증거 기반 진단 순서

## 1. 흐름을 정의한다

source ENI와 IP, destination IP와 port, protocol, 방향과 실패 시각을 기록한다.

모든 통신인지 특정 AZ, subnet, endpoint 또는 IPv4와 IPv6 중 하나인지 구분한다.

## 2. 최근 변경을 확인한다

Security Group, NACL, route table, subnet association, NAT, endpoint와 배포 변경을 확인한다.

CloudTrail의 변경 API와 장애 시작 시각을 비교한다.

## 3. 애플리케이션 경계를 확인한다

```bash
ss -lntp
curl --verbose --connect-timeout 3 --max-time 5 https://api.example.com/health
```

listener 유무와 local firewall 및 실제 bind address를 확인한다.

TCP 연결 실패와 HTTP 상태 응답을 구분해 network와 application 계층을 섞지 않는다.

## 4. Security Group을 확인한다

source의 egress와 destination의 ingress가 protocol과 port 및 source 조건을 허용하는지 확인한다.

```bash
aws ec2 describe-security-groups --group-ids "$SOURCE_SG" "$DESTINATION_SG"
```

가능하면 변동하는 IP보다 Security Group 참조로 서비스 간 허용 관계를 표현한다.

stateful 특성은 명시적으로 허용된 신규 연결에 적용되며 잘못된 inbound 규칙을 보완하지 않는다.

## 5. NACL을 확인한다

subnet에 실제 연결된 NACL의 rule 번호, allow 또는 deny, protocol과 port 범위를 양방향으로 확인한다.

```bash
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values="$SUBNET_ID"
```

stateless NACL에서는 server port로 가는 요청과 client ephemeral port로 돌아오는 응답을 각각 허용해야 한다.

ephemeral port 범위는 client 운영체제와 중간 구성에 따라 확인하며 임의의 고정 범위를 단정하지 않는다.

## 6. route와 gateway를 확인한다

```bash
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values="$SUBNET_ID"
aws ec2 describe-nat-gateways \
  --filter Name=subnet-id,Values="$NAT_SUBNET_ID"
aws ec2 describe-internet-gateways \
  --filters Name=attachment.vpc-id,Values="$VPC_ID"
```

가장 구체적인 route가 선택되는지와 route target 상태가 active인지 확인한다.

NAT가 public subnet에 있고 그 subnet이 IGW로 가는 route를 가지는지 확인한다.

## 7. 경로 분석 도구를 사용한다

VPC Reachability Analyzer는 구성 모델을 기준으로 source와 destination 사이 차단 요소를 찾는다.

분석 결과가 reachable이어도 애플리케이션 listener, DNS, runtime 부하까지 정상이라는 뜻은 아니다.

VPC Flow Logs에서는 interface, source, destination, port, action과 시간 범위를 확인한다.

```sql
fields @timestamp, interfaceId, srcAddr, dstAddr,
       srcPort, dstPort, action
| filter srcAddr = "10.0.1.10" and dstAddr = "10.0.2.20"
| stats count() by action, dstPort, bin(1m)
| sort @timestamp desc
```

Flow Logs의 REJECT는 정책 단서를 제공하지만 모든 packet의 애플리케이션 성공을 보장하지 않는다.

## 8. NAT와 port 자원을 확인한다

NAT Gateway의 connection과 port allocation 관련 지표 및 오류를 확인한다.

짧은 연결 폭증, destination 집중과 client connection pool 미사용 여부를 함께 조사한다.

## 9. 안전하게 완화한다

최근의 잘못된 network 변경을 검증된 설정으로 되돌린다.

긴급 허용 규칙을 전체 인터넷에 개방하기보다 필요한 source, protocol과 port로 최소화한다.

---

# Spring Boot 관찰 포인트

DNS lookup, connect timeout, TLS handshake와 read timeout을 서로 다른 지표와 예외로 기록한다.

HTTP client connection pool의 active, pending, idle과 connection reuse를 관찰한다.

짧은 connection을 반복 생성하면 ephemeral port와 NAT connection 압력을 높일 수 있다.

dependency별 timeout과 제한된 retry를 설정하고 전체 요청 지연 예산을 넘지 않게 한다.

---

# 대응과 복구

정확한 packet 흐름을 기준으로 최소 규칙과 route를 수정한다.

NAT 포화는 connection 재사용과 요청 패턴을 개선하고 검증된 확장 설계를 적용한다.

복구 후 양방향 연결, 실제 HTTP 요청과 모든 관련 AZ에서의 성공을 확인한다.

---

# 재발 방지

network 구성을 코드로 관리하고 변경 리뷰와 경로 검증을 자동화한다.

Flow Logs 보존과 검색 필드를 미리 준비해 사고 시점의 증거를 확보한다.

핵심 경로는 Reachability Analyzer와 합성 요청으로 정기 검증한다.

NAT와 connection 지표에 서비스 기준의 경보를 구성한다.

---

# 기억해야 할 내용

- Security Group은 stateful이고 NACL은 stateless이다.
- route, gateway와 정책을 양방향으로 확인한다.
- ephemeral port는 client와 반환 경로에서 중요하다.
- Reachability Analyzer는 구성 경로를, Flow Logs는 관찰된 흐름을 설명한다.
- 전체 개방은 안전한 장애 완화가 아니다.

---

# 다음 Chapter

다음 장에서는 [Incident Response](/aws-backend/part-15/09-incident-response/)를 학습한다.
