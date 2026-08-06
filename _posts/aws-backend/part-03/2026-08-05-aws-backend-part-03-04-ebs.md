---
title: "Chapter 04. EBS"
permalink: /aws-backend/part-03/04-ebs/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. EBS
## EC2에 붙는 블록 스토리지

> **학습 목표**
>
> - EBS의 역할을 설명할 수 있다.
> - 루트 볼륨과 데이터 볼륨을 구분할 수 있다.
> - Snapshot과 볼륨 타입의 의미를 이해한다.

---

# EBS란?

EBS(Elastic Block Store)는 EC2에 연결하는 블록 스토리지다.

EC2의 디스크처럼 사용한다.

루트 볼륨은 OS가 설치되는 디스크이고, 데이터 볼륨은 추가 저장 공간으로 사용할 수 있다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# EBS와 EC2 생명주기

EC2가 중지되어도 EBS 볼륨은 남을 수 있다.

반대로 인스턴스를 종료할 때 루트 볼륨을 함께 삭제하도록 설정할 수도 있다.

운영에서는 중요한 데이터가 EC2 로컬에만 남지 않도록 주의해야 한다.

---

# Snapshot

Snapshot은 EBS 볼륨의 백업이다.

Snapshot을 이용해 다음 작업을 할 수 있다.

- 볼륨 복구
- 다른 AZ 또는 Region으로 복사
- AMI 생성
- 장애 대응

---

# 볼륨 타입

대표적으로 `gp3`를 많이 사용한다.

성능이 더 필요하면 IOPS나 처리량을 조정하거나 다른 타입을 선택한다.

처음부터 고성능 타입을 쓰기보다 실제 지표를 보고 조정하는 것이 낫다.

---

# 기억해야 할 내용

- EBS는 EC2에 붙는 블록 스토리지다.
- 루트 볼륨은 OS 디스크다.
- Snapshot은 백업과 복구에 사용한다.
- 중요한 영속 데이터는 RDS, S3 같은 목적에 맞는 저장소를 우선 고려한다.


