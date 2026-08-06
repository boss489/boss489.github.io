---
title: "Chapter 06. Cloud Service Models"
permalink: /aws-backend/part-01/06-cloud-service-model/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Cloud Service Models
## 어디까지 직접 관리할 것인가

> **학습 목표**
>
> - IaaS, PaaS, SaaS, FaaS의 차이를 설명할 수 있다.
> - Managed Service의 의미를 이해한다.
> - 서비스 모델에 따라 운영 책임이 달라지는 것을 설명할 수 있다.

---

# 서비스 모델이 필요한 이유

Cloud는 모든 것을 같은 수준으로 제공하지 않는다.

어떤 서비스는 서버에 가까운 형태이고, 어떤 서비스는 완성된 기능에 가깝다.

서비스 모델은 사용자가 어디까지 관리하고 AWS가 어디까지 관리하는지 구분하는 기준이다.

---

# IaaS

IaaS(Infrastructure as a Service)는 서버, 네트워크, 스토리지를 빌려 쓰는 모델이다.

대표 예시는 EC2다.

사용자는 OS, Runtime, 애플리케이션을 직접 관리한다.

---

# PaaS

PaaS(Platform as a Service)는 애플리케이션 실행 플랫폼을 제공하는 모델이다.

사용자는 코드와 설정에 집중하고, 서버 운영 부담은 줄어든다.

예시는 다음과 같다.

- Elastic Beanstalk
- App Runner

---

# SaaS

SaaS(Software as a Service)는 완성된 소프트웨어를 사용하는 모델이다.

사용자는 인프라나 런타임을 관리하지 않는다.

예시는 다음과 같다.

- Gmail
- Slack
- Salesforce

---

# FaaS

FaaS(Function as a Service)는 함수를 배포하고 이벤트가 발생할 때 실행하는 모델이다.

AWS에서는 Lambda가 대표적이다.

---

# Managed Service

Managed Service는 AWS가 운영의 일부를 대신 관리해주는 서비스다.

예를 들어 RDS는 데이터베이스 엔진을 직접 설치하지 않아도 사용할 수 있다.

하지만 데이터 모델, 권한, 백업 정책, 비용은 여전히 사용자가 설계해야 한다.

---

# 기억해야 할 내용

- 서비스 모델은 책임 범위를 나누는 기준이다.
- EC2는 IaaS에 가깝다.
- Lambda는 FaaS이다.
- Managed Service를 써도 설계 책임이 사라지지는 않는다.


