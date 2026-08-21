---
title: "Chapter 05. ENI"
permalink: /aws-backend/part-03/05-eni/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. ENI
## EC2의 네트워크 인터페이스

> **학습 목표**
>
> - ENI의 역할을 설명할 수 있다.
> - Private IP, Public IP, Security Group과의 관계를 이해한다.
> - EC2 네트워크 문제를 어떤 관점으로 봐야 하는지 이해한다.
> - 패킷이 EC2까지 도달하는 경로를 계층별로 점검할 수 있다.

---

# 왜 ENI가 필요한가

Spring Boot 프로세스가 `8080` 포트에서 정상 실행되어도 네트워크 인터페이스가 없다면 다른 서버와 통신할 수 없다.

쇼핑몰 API가 RDS에 접속하고 ALB의 요청을 받으려면 VPC 안에서 식별되는 IP와 통신 규칙이 필요하다.

물리 서버의 네트워크 카드 역할을 AWS에서 가상화한 자원이 ENI이다.

ENI를 이해하면 EC2 문제와 VPC 경로 문제, Security Group 문제를 분리해서 진단할 수 있다.

---

# ENI란?

ENI(Elastic Network Interface)는 EC2에 연결되는 가상 네트워크 카드다.

EC2가 VPC 안에서 통신하려면 ENI가 필요하다.

ENI에는 다음이 연결된다.

- Private IP
- Public IP 또는 Elastic IP
- Security Group
- Subnet
- MAC Address

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# ENI의 구조와 패킷 흐름

Primary ENI는 인스턴스를 시작한 Subnet에 생성되며 기본 Private IPv4 주소를 가진다.

```
Client
  │
Route Table
  │
Subnet
  │
ENI ── Private IP
 ├── Security Group
 ├── Public IP / Elastic IP 매핑
 └── EC2 ── Spring Boot :8080
```

인터넷 요청은 Public IP에 도착한 뒤 ENI의 Private IP로 변환되어 전달된다.

응답은 ENI에서 Subnet의 Route Table이 선택한 경로를 따라 나간다.

Security Group은 상태 저장 방식이므로 허용된 요청의 반환 트래픽은 연결 상태를 바탕으로 처리된다.

---

# Private IP와 Public IP

Private IP는 VPC 내부 통신에 사용한다.

Public IP는 인터넷에서 접근할 때 사용한다.

EC2가 Public Subnet에 있어도 Public IP가 없으면 인터넷에서 직접 접근할 수 없다.

| 주소 | 연결 대상 | 생명주기 | 대표 용도 |
|---|---|---|---|
| Primary Private IP | ENI | ENI와 함께 유지 | VPC 내부 통신 |
| Secondary Private IP | ENI | 할당·이동 가능 | 여러 서비스 주소 |
| 자동 Public IP | Private IP에 매핑 | Stop/Start 시 변경 가능 | 임시 인터넷 접속 |
| Elastic IP | Private IP에 매핑 | 명시적으로 해제할 때까지 유지 | 고정 공개 주소 |

운영 API의 안정적인 진입점은 EC2 Public IP보다 ALB와 DNS를 사용하는 것이 일반적이다.

---

# Security Group

Security Group은 ENI에 적용된다.

즉 EC2에 방화벽이 붙는다고 표현하지만, 실제로는 EC2의 네트워크 인터페이스에 규칙이 적용된다고 이해하면 좋다.

---

# 여러 ENI는 언제 사용하는가

추가 ENI를 사용하면 관리 트래픽과 서비스 트래픽을 분리하거나 네트워크 어플라이언스를 구성할 수 있다.

추가 ENI를 연결할 수 있는 수와 IP 수는 인스턴스 타입에 따라 다르다.

일반적인 Spring Boot API는 Primary ENI 하나로 충분하므로 목적 없이 인터페이스를 늘릴 필요가 없다.

ENI는 하나의 AZ에 속하므로 다른 AZ의 인스턴스에 직접 연결할 수 없다.

---

# AWS 콘솔과 CLI에서는

인스턴스의 ENI와 주소, Security Group을 한 번에 조회할 수 있다.

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=i-0123456789abcdef0" \
  --query "NetworkInterfaces[].{Id:NetworkInterfaceId,Ip:PrivateIpAddress,Subnet:SubnetId,Groups:Groups[].GroupId}"
```

연결되지 않은 ENI가 남아 있으면 어떤 서비스가 만들었는지 Description과 태그를 먼저 확인한다.

ALB, Lambda VPC 연결, VPC Endpoint 같은 관리형 서비스도 ENI를 생성할 수 있으므로 임의 삭제하면 안 된다.

---

# Spring Boot에서는 어떻게 보이는가

Spring Boot가 `127.0.0.1`에만 바인딩되면 EC2 내부에서만 접근할 수 있다.

외부 인터페이스의 요청을 받으려면 일반적으로 모든 인터페이스를 의미하는 주소로 바인딩한다.

```yaml
server:
  address: 0.0.0.0
  port: 8080
```

다만 `0.0.0.0` 바인딩이 인터넷 공개를 의미하지는 않으며 실제 접근 범위는 Route Table과 Security Group이 결정한다.

애플리케이션 로그에는 변경 가능한 Public IP보다 인스턴스 ID나 환경 태그를 함께 남기는 편이 추적에 유리하다.

---

# 실무에서는 어떻게 사용할까

ALB에서 EC2로 요청을 전달할 때 EC2 Security Group의 `8080` 인바운드 소스를 ALB Security Group으로 제한한다.

```
Internet
   │ :443
ALB ENI [ALB-SG]
   │ :8080
EC2 ENI [APP-SG]
   │ :5432
RDS ENI [DB-SG]
```

IP 대역 전체를 허용하는 대신 Security Group 참조를 사용하면 인스턴스가 교체되어 IP가 바뀌어도 규칙을 수정하지 않아도 된다.

---

# 장애 사례와 점검 순서

| 단계 | 확인할 내용 | 대표 도구 |
|---|---|---|
| 애플리케이션 | 프로세스가 올바른 주소와 포트에 바인딩됨 | `ss -lntp` |
| ENI | Private IP와 SG가 예상과 일치함 | EC2 콘솔·CLI |
| 보안 | 인바운드와 아웃바운드 규칙 | Security Group |
| Subnet | 목적지 경로가 존재함 | Route Table |
| 경계 | NACL, IGW, NAT 구성 | VPC 콘솔 |

Public Subnet이라는 이름만으로 인터넷 접속이 보장되지 않으며 Public IP, IGW 경로, Security Group이 모두 필요하다.

소스·목적지 확인을 비활성화하는 설정은 NAT 인스턴스 같은 특수 용도에만 사용해야 한다.

---

# 비용과 성능 고려사항

ENI 자체보다 연결된 Public IPv4 주소와 NAT, AZ 간 데이터 전송이 비용에 영향을 줄 수 있다.

인스턴스 타입별 네트워크 대역폭과 패킷 처리 성능이 다르므로 CPU가 낮아도 네트워크가 병목일 수 있다.

CloudWatch와 VPC Flow Logs를 함께 사용하면 허용·거부 흐름과 트래픽 변화를 분석할 수 있다.

---

# 기억해야 할 내용

- ENI는 EC2의 네트워크 카드다.
- EC2 통신 문제는 ENI, Subnet, Route Table, Security Group을 함께 봐야 한다.
- Public IP가 없으면 인터넷에서 직접 접근할 수 없다.
- Security Group은 리소스의 네트워크 인터페이스에 적용된다.
- ENI와 Subnet은 하나의 AZ에 속한다.
- 애플리케이션 바인딩과 네트워크 접근 허용은 서로 다른 문제이다.
- 관리형 서비스가 만든 ENI는 소유 관계를 확인한 뒤 다뤄야 한다.

---

# 다음 Chapter

다음 Chapter에서는 [Linux](/aws-backend/part-03/06-linux/)를 학습한다.

네트워크를 통해 EC2에 도달한 뒤 프로세스, 포트, 파일 권한과 로그를 어떻게 확인하는지 알아본다.


