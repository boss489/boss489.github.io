---
title: "Chapter 03. AMI"
permalink: /aws-backend/part-03/03-ami/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. AMI
## EC2를 시작하기 위한 이미지

> **학습 목표**
>
> - AMI의 역할을 설명할 수 있다.
> - OS 이미지와 애플리케이션 이미지의 차이를 이해한다.
> - Golden Image 전략의 장단점을 설명할 수 있다.

---

# AMI란?

AMI(Amazon Machine Image)는 EC2 인스턴스를 만들 때 사용하는 시작 이미지다.

AMI에는 보통 다음 정보가 포함된다.

- OS
- 기본 패키지
- 파일 시스템 구성
- 시작 설정

EC2는 AMI를 기반으로 루트 볼륨을 만들고 부팅한다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# AMI 선택

가장 흔한 선택은 다음과 같다.

- Amazon Linux
- Ubuntu
- Red Hat Enterprise Linux
- Windows Server

백엔드 서버에서는 Amazon Linux 또는 Ubuntu를 자주 사용한다.

---

# Golden Image

Golden Image는 필요한 패키지와 보안 설정을 미리 반영한 표준 AMI다.

장점은 인스턴스 생성 후 설정 시간이 줄어든다는 것이다.

단점은 이미지 관리 프로세스가 필요하다는 것이다.

작은 팀에서는 처음부터 복잡한 Golden Image 체계를 만들기보다 User Data, Docker Image, IaC로 시작하는 편이 단순하다.

---

# 기억해야 할 내용

- AMI는 EC2를 시작하기 위한 이미지다.
- AMI는 OS와 초기 파일 시스템 구성을 제공한다.
- Golden Image는 반복 설정을 줄이지만 관리 비용이 생긴다.
- 애플리케이션 배포 단위는 AMI보다 Docker Image가 더 자주 사용된다.


