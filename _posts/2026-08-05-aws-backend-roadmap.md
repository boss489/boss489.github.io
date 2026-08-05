---
title: "AWS for Backend Developer - Roadmap"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# AWS for Backend Developer
## Spring Boot 개발자를 위한 AWS 아키텍처 완전 정복

> **목표**
>
> AWS 서비스를 외우는 것이 아니라,
> **실무에서 아키텍처를 설계하고 운영할 수 있는 백엔드 개발자**가 되는 것을 목표로 한다.
>
> 이 과정은 AWS 자격증 과정이 아니다.
> "왜 이런 구조가 만들어졌는가?"를 이해하는 과정이다.

---

# 최종 목표

과정을 마치면 다음을 할 수 있다.

- 인터넷 요청이 어떻게 내 API까지 오는지 설명할 수 있다.
- AWS 전체 아키텍처를 스스로 설계할 수 있다.
- 운영 중인 시스템의 장애를 분석할 수 있다.
- 비용과 성능을 고려하여 서비스를 설계할 수 있다.
- Spring Boot 서비스를 AWS에서 안정적으로 운영할 수 있다.

---

# 전체 로드맵

```
                Internet
                    │
                Route53
                    │
              CloudFront
                    │
                   ALB
                    │
          ECS / EC2 (Spring Boot)
                    │
          Redis / Aurora / S3
                    │
             CloudWatch & IAM
```

---

# Level 1. Foundation
## 클라우드와 네트워크의 이해

> "AWS가 왜 필요한가?"

Cloud의 기본 개념과 AWS의 기반이 되는 네트워크를 학습한다.

---

## [Part 1. Cloud Computing](/aws/aws-backend-part-01-cloud-computing/)

### 목표

Cloud가 등장한 이유를 이해한다.

### 학습 내용

- On-Premise
- Virtual Machine
- Hypervisor
- Cloud Computing
- IaaS / PaaS / SaaS
- AWS Global Infrastructure
- Region
- Availability Zone
- Edge Location
- Shared Responsibility

---

## [Part 2. AWS Networking](/aws/aws-backend-part-02-networking/)

### 목표

AWS 서비스가 동작하는 네트워크 구조를 이해한다.

### 학습 내용

- [VPC](/aws/aws-backend-vpc/)
- [CIDR](/aws/aws-backend-cidr/)
- [Subnet](/aws/aws-backend-subnet/)
- [Route Table](/aws/aws-backend-route-table/)
- [Internet Gateway](/aws/aws-backend-internet-gateway/)
- [NAT Gateway](/aws/aws-backend-nat-gateway/)
- [Security Group](/aws/aws-backend-security-group/)
- [NACL](/aws/aws-backend-nacl/)
- [Elastic IP](/aws/aws-backend-elastic-ip/)

---

## [Part 3. Compute](/aws/aws-backend-part-03-compute/)

### 목표

애플리케이션이 실행되는 환경을 이해한다.

### 학습 내용

- EC2
- AMI
- EBS
- ENI
- Linux
- SSH
- Docker
- Systemd

---

## [Part 4. Request Flow](/aws/aws-backend-part-04-request-flow/)

### 목표

사용자의 요청이 어떻게 API까지 도달하는지 이해한다.

### 학습 내용

- DNS
- Route53
- ALB
- Target Group
- Health Check
- Auto Scaling
- Session
- Keep Alive

---

# Level 2. Application Platform
## 애플리케이션이 실행되는 플랫폼

> "Spring Boot 서비스는 AWS에서 어떻게 운영되는가?"

---

## [Part 5. Database](/aws/aws-backend-part-05-database/)

### 목표

AWS에서 데이터베이스를 안정적으로 운영한다.

### 학습 내용

- RDS
- Aurora
- Multi AZ
- Read Replica
- Backup
- Restore
- Failover

---

## [Part 6. Cache](/aws/aws-backend-part-06-cache/)

### 목표

Redis를 활용하여 성능을 향상시킨다.

### 학습 내용

- Cache Aside
- Write Through
- TTL
- Session
- Distributed Lock
- Pub/Sub
- ElastiCache

---

## [Part 7. Object Storage](/aws/aws-backend-part-07-object-storage/)

### 목표

S3 기반 파일 서비스를 이해한다.

### 학습 내용

- Bucket
- Object
- Versioning
- Lifecycle
- Presigned URL
- Multipart Upload
- CloudFront

---

## [Part 8. Container Platform](/aws/aws-backend-part-08-container-platform/)

### 목표

Docker 기반 서비스를 운영한다.

### 학습 내용

- Docker
- Image
- Container
- ECR
- ECS
- Task
- Service
- Cluster

---

## [Part 9. Kubernetes](/aws/aws-backend-part-09-kubernetes/)

### 목표

EKS 기반 컨테이너 오케스트레이션을 이해한다.

### 학습 내용

- Kubernetes
- Pod
- ReplicaSet
- Deployment
- Service
- Ingress
- ALB Controller
- EKS

---

## [Part 10. Serverless](/aws/aws-backend-part-10-serverless/)

### 목표

Serverless 아키텍처를 이해한다.

### 학습 내용

- Lambda
- API Gateway
- EventBridge
- DynamoDB
- Step Functions

---

# Level 3. Operations
## 운영, 보안, 자동화

> "서비스를 안전하게 운영하는 방법"

---

## [Part 11. Monitoring](/aws/aws-backend-part-11-monitoring/)

### 목표

운영 중인 서비스를 관찰한다.

### 학습 내용

- CloudWatch
- Logs
- Metrics
- Alarm
- Dashboard
- CloudTrail
- X-Ray

---

## [Part 12. Security](/aws/aws-backend-part-12-security/)

### 목표

AWS 보안 모델을 이해한다.

### 학습 내용

- IAM
- User
- Group
- Role
- Policy
- STS
- KMS
- Secrets Manager

---

## [Part 13. CI/CD](/aws/aws-backend-part-13-cicd/)

### 목표

배포를 자동화한다.

### 학습 내용

- GitHub Actions
- CodeBuild
- CodeDeploy
- CodePipeline
- Blue/Green
- Rolling Update

---

## [Part 14. Cost Optimization](/aws/aws-backend-part-14-cost-optimization/)

### 목표

AWS 비용을 최적화한다.

### 학습 내용

- EC2
- NAT Gateway
- ALB
- CloudFront
- S3
- Savings Plan
- Reserved Instance

---

## [Part 15. Troubleshooting](/aws/aws-backend-part-15-troubleshooting/)

### 목표

장애를 분석하고 대응한다.

### 학습 내용

- ALB 502
- ALB 503
- EC2 장애
- RDS 장애
- Redis 장애
- DNS 장애
- Network 장애
- 장애 분석 순서

---

# Level 4. Architecture

> "실제 서비스를 설계한다."

---

## [Part 16. Architecture Design](/aws/aws-backend-part-16-architecture-design/)

### 목표

실제 서비스를 AWS에서 설계한다.

### 학습 내용

- 쇼핑몰 아키텍처
- REST API
- Batch
- Kafka
- Redis
- Aurora
- ECS
- CloudFront
- S3
- Auto Scaling
- Multi AZ
- DR(Disaster Recovery)

---

## [Part 17. Final Project](/aws/aws-backend-part-17-final-project/)

### 목표

Spring Boot 서비스를 처음부터 AWS에 구축한다.

### 구현 항목

- VPC 설계
- ECS 구축
- Aurora 연결
- Redis 연결
- S3 업로드
- CloudFront 적용
- Route53 연결
- HTTPS 적용
- Auto Scaling
- CloudWatch 모니터링
- CI/CD 구축

---

# 각 Part 진행 방식

모든 Part는 동일한 형식으로 진행한다.

1. 왜 필요한가?
2. 핵심 개념
3. 아키텍처 다이어그램
4. 내부 동작 원리
5. AWS 콘솔 실습
6. Spring Boot 연동
7. 운영 사례
8. 장애 사례
9. 비용 및 성능 고려사항
10. 면접 질문
11. 핵심 요약

---

# 학습 시간

| Level | 내용 | 예상 시간 |
|--------|------|----------:|
| Level 1 | Foundation | 12h |
| Level 2 | Application Platform | 18h |
| Level 3 | Operations | 12h |
| Level 4 | Architecture | 10h |

**총 약 52시간**

---

# 이 과정을 마치면

다음과 같은 아키텍처를 스스로 설명하고 구축할 수 있다.

```
                Internet
                    │
              Route53 (DNS)
                    │
             CloudFront (CDN)
                    │
             WAF (선택사항)
                    │
                  ALB
                    │
        ┌───────────┴───────────┐
        │                       │
      ECS Task              ECS Task
   (Spring Boot)         (Spring Boot)
        │                       │
        └───────────┬───────────┘
                    │
            ElastiCache (Redis)
                    │
          Aurora (Writer / Reader)
                    │
                   S3
                    │
      CloudWatch / IAM / Secrets Manager
```
