---
title: "Chapter 10. Interview Questions"
permalink: /aws-backend/part-12/10-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Interview Questions
## AWS Security 면접 질문

---

# Shared Responsibility Model은 무엇인가?

모범 답변은 AWS가 클라우드 자체의 물리 시설과 기반 서비스를 보호하고 고객은 클라우드 안의 데이터, Identity, 운영체제와 서비스 설정을 보호한다는 책임 경계다.

서비스가 더 관리형일수록 AWS가 담당하는 기반 범위는 넓어지지만 데이터 분류와 접근 제어는 여전히 고객 책임이다.

## 꼬리질문

RDS를 사용하면 고객이 책임져야 하는 보안 설정에는 무엇이 있는가?

---

# Defense in Depth가 필요한 이유는 무엇인가?

모범 답변은 하나의 통제가 실패해도 네트워크, Identity, 암호화, 탐지와 복구의 다른 계층이 공격을 제한하도록 하기 위해서다.

암호화는 잘못된 애플리케이션 권한을 대신하지 않고 WAF는 내부 권한 오용을 막지 못하므로 통제를 겹쳐야 한다.

## 꼬리질문

Spring Boot API의 방어 계층을 요청 입구부터 데이터까지 설명해 보라.

---

# IAM User와 Role의 차이는 무엇인가?

모범 답변은 User가 주로 사람을 식별하고 장기 자격 증명을 가질 수 있는 반면 Role은 신뢰받은 주체가 맡아 STS 임시 자격 증명을 사용하는 Identity라는 것이다.

Workload에는 User의 액세스 키가 아니라 ECS Task Role, Instance Profile 또는 IRSA를 사용한다.

## 꼬리질문

사람에게도 IAM User보다 Identity Center를 우선할 수 있는 이유는 무엇인가?

---

# Trust Policy와 Permission Policy의 차이는 무엇인가?

모범 답변은 Trust Policy가 `Principal`과 `sts:AssumeRole`로 누가 Role을 맡는지 정하고 Permission Policy가 맡은 세션이 어떤 `Action`을 어떤 `Resource`에 수행하는지 정한다는 것이다.

두 정책 중 하나만 충족해서는 원하는 API 호출이 허용되지 않는다.

## 꼬리질문

교차 계정 SaaS 위임에서 External ID는 어떤 위험을 줄이는가?

---

# STS 임시 자격 증명의 장점은 무엇인가?

모범 답변은 자격 증명이 만료되고 세션별 권한과 식별 정보를 적용할 수 있어 장기 키보다 유출 범위와 교체 부담을 줄인다는 것이다.

Access Key ID, Secret Access Key와 Session Token을 함께 사용하며 SDK가 만료 전 갱신하도록 한다.

## 꼬리질문

임시 자격 증명이 유출되면 만료될 때까지 기다려도 되는가?

---

# IAM Policy 평가 순서를 설명해 보라.

모범 답변은 요청이 Implicit Deny에서 시작하고 적용 정책에 Explicit Allow가 있으며 어떤 Explicit Deny도 없어야 허용된다는 것이다.

Explicit Deny는 Allow보다 우선하며 SCP와 Permissions Boundary는 권한을 직접 부여하지 않고 상한을 제한한다.

## 꼬리질문

Identity Policy가 Allow하는데도 S3 요청이 거부되는 원인을 세 가지 말해 보라.

---

# Permissions Boundary와 SCP의 차이는 무엇인가?

모범 답변은 Permissions Boundary가 특정 User나 Role의 Identity 권한 상한이고 SCP가 Organization의 계정 또는 OU에 적용되는 권한 상한이라는 것이다.

둘 다 Allow를 추가하지 않으며 실제 권한에는 Identity 또는 Resource Policy의 허용이 필요하다.

## 꼬리질문

SCP에 `s3:GetObject`가 있어도 애플리케이션이 S3를 읽지 못하는 이유는 무엇인가?

---

# KMS Envelope Encryption을 설명해 보라.

모범 답변은 KMS가 생성한 평문 Data Key로 데이터를 로컬 암호화하고 암호화된 Data Key를 암호문과 함께 저장하는 방식이다.

복호화 시 KMS 권한으로 Data Key를 복호화하며 평문 Data Key는 사용 직후 메모리에서 제거한다.

## 꼬리질문

암호화된 Data Key를 데이터 옆에 저장해도 되는 이유는 무엇인가?

---

# Secrets Manager와 Parameter Store를 어떻게 선택하는가?

모범 답변은 자동 Rotation과 비밀 버전 수명 주기가 중요하면 Secrets Manager를 선택하고 일반 설정이나 계층형 Parameter가 중심이면 Parameter Store를 검토한다는 것이다.

정확한 기능과 비용은 현재 Region의 공식 문서와 실제 호출량으로 판단한다.

## 꼬리질문

모든 설정을 Secrets Manager에 넣는 설계의 단점은 무엇인가?

---

# Secret Rotation 때 애플리케이션 장애를 막는 방법은 무엇인가?

모범 답변은 Rotation 단계를 멱등하게 만들고 새 자격 증명을 검증한 뒤 현재 버전을 전환하며 애플리케이션 캐시 TTL과 연결 갱신을 함께 테스트하는 것이다.

이전 값과 새 값의 전환 구간을 대상 시스템이 어떻게 처리하는지도 설계한다.

## 꼬리질문

Rotation 성공 후에도 이전 비밀번호로 연결을 시도하는 원인은 무엇인가?

---

# Spring Boot에서 AWS 자격 증명을 어떻게 제공하는가?

모범 답변은 AWS SDK v2의 `DefaultCredentialsProvider`를 사용하고 운영 환경에서는 ECS Task Role, Instance Profile 또는 IRSA가 제공한 임시 자격 증명을 사용한다는 것이다.

장기 Access Key를 환경 변수, 설정 파일이나 컨테이너 이미지에 넣지 않는다.

## 꼬리질문

환경 변수의 장기 키가 Task Role보다 먼저 선택될 때 어떤 문제가 생기는가?

---

# 주요 Security Operations 서비스의 역할은 무엇인가?

모범 답변은 CloudTrail이 API 활동, Config가 구성 이력, Access Analyzer가 외부 접근과 미사용 권한, GuardDuty가 위협 신호, Security Hub가 Finding 집계를 담당한다는 것이다.

각 서비스는 다른 질문에 답하므로 하나로 나머지를 대체할 수 없다.

## 꼬리질문

Config 비준수와 GuardDuty Finding의 의미 차이를 설명해 보라.

---

# Access Key 유출 사고에 어떻게 대응하는가?

모범 답변은 키를 즉시 비활성화하고 관련 세션과 권한을 제한한 뒤 CloudTrail로 최초 사용, 호출 Action, 생성 Resource와 데이터 접근 범위를 조사하는 것이다.

증거를 보존한 상태에서 원인을 제거하고 깨끗한 자격 증명과 배포물로 복구한 뒤 탐지 규칙과 최소 권한을 개선한다.

## 꼬리질문

키가 포함된 Git 파일에서 문자열만 삭제하면 해결되지 않는 이유는 무엇인가?

---

# Part 12를 마치며

AWS 보안은 제품 하나가 아니라 책임 경계, Identity, 정책, 암호화, 비밀 관리, 탐지와 대응을 연결하는 운영 체계다.
