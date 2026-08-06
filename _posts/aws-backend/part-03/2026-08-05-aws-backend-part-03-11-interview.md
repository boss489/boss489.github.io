---
title: "Chapter 11. Interview Questions"
permalink: /aws-backend/part-03/11-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 11. Interview Questions
## AWS Compute 면접 질문

---

# EC2

## EC2란 무엇인가요?

AWS에서 제공하는 가상 서버 서비스입니다.

AMI를 기반으로 생성되고, EBS 스토리지와 ENI 네트워크 인터페이스가 연결됩니다.

## EC2 인스턴스 타입은 무엇을 결정하나요?

CPU, 메모리, 네트워크 성능 같은 서버 사양을 결정합니다.

---

# AMI와 EBS

## AMI는 무엇인가요?

EC2 인스턴스를 시작하기 위한 이미지입니다.

OS와 초기 파일 시스템 구성이 포함됩니다.

## EBS는 무엇인가요?

EC2에 연결하는 블록 스토리지입니다.

루트 볼륨은 OS 디스크로 사용되고, Snapshot으로 백업할 수 있습니다.

---

# ENI와 네트워크

## ENI는 무엇인가요?

EC2에 붙는 가상 네트워크 인터페이스입니다.

Private IP, Public IP, Security Group, Subnet 정보와 연결됩니다.

## EC2가 Public Subnet에 있으면 항상 인터넷에서 접속 가능한가요?

아닙니다.

Public IP, Internet Gateway 경로, Security Group 허용이 함께 필요합니다.

---

# Linux와 SSH

## SSH 접속이 안 될 때 무엇을 확인하나요?

Public IP 또는 네트워크 경로, Security Group 22번 포트, 키 파일, 사용자 이름, NACL, 인스턴스 상태를 확인합니다.

## EC2에서 애플리케이션이 실행 중인지 어떻게 확인하나요?

`ps`, `ss`, `systemctl status`, `journalctl`, 애플리케이션 로그를 확인합니다.

---

# Docker와 Systemd

## Docker Image와 Container의 차이는 무엇인가요?

Image는 실행 템플릿이고 Container는 Image를 실행한 프로세스입니다.

## Systemd를 사용하는 이유는 무엇인가요?

애플리케이션을 Linux 서비스로 등록해 부팅 시 자동 시작, 상태 확인, 재시작, 로그 확인을 하기 위해 사용합니다.

---

# 심화 질문

## EC2에 직접 Docker를 올리는 방식과 ECS의 차이는 무엇인가요?

EC2 직접 운영은 서버와 Docker 실행 관리를 사용자가 담당합니다.

ECS는 컨테이너 배치, 재시작, 서비스 유지 같은 운영 기능을 더 많이 제공해 직접 관리 부담을 줄입니다.


