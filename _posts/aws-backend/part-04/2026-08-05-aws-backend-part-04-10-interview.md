---
title: "Chapter 10. Interview Questions"
permalink: /aws-backend/part-04/10-interview/
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
## Request Flow 면접 질문

---

# DNS와 Route 53

## DNS는 어떤 역할을 하나요?

도메인 이름을 실제 접속 가능한 주소나 AWS 리소스로 연결합니다.

## Route 53 Alias Record는 언제 사용하나요?

ALB, CloudFront 같은 AWS 리소스를 도메인에 연결할 때 사용합니다.

---

# ALB

## ALB와 Target Group의 관계는 무엇인가요?

ALB는 요청을 받고, Target Group은 ALB가 요청을 전달할 대상 묶음입니다.

## Health Check가 실패하면 어떻게 되나요?

ALB는 실패한 Target에 사용자 요청을 보내지 않습니다.

---

# 운영

## API 요청이 502를 반환하면 무엇을 확인하나요?

Target Group Health Check, 애플리케이션 포트, Security Group, 애플리케이션 로그를 확인합니다.

## Sticky Session의 단점은 무엇인가요?

특정 서버에 부하가 몰릴 수 있고, 서버 장애 시 세션이 사라질 수 있습니다.


