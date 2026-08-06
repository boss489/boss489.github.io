---
title: "Chapter 04. Hypervisor"
permalink: /aws-backend/part-01/04-hypervisor/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Hypervisor
## VM을 만들고 관리하는 계층

> **학습 목표**
>
> - Hypervisor의 역할을 설명할 수 있다.
> - Type 1과 Type 2 Hypervisor의 차이를 이해한다.
> - AWS Nitro Hypervisor가 어떤 맥락에서 등장했는지 이해한다.

---

# Hypervisor란?

Hypervisor는 물리 서버 자원을 여러 VM에 나누어 주는 소프트웨어 계층이다.

Hypervisor는 다음을 관리한다.

- CPU 스케줄링
- 메모리 할당
- 디스크 접근
- 네트워크 장치
- VM 생성과 삭제

---

# Type 1 Hypervisor

Type 1은 물리 서버 위에서 직접 실행되는 방식이다.

```
Hardware
└── Hypervisor
    └── VM
```

서버 환경에서 많이 사용한다.

예시는 다음과 같다.

- VMware ESXi
- Xen
- KVM
- AWS Nitro Hypervisor

---

# Type 2 Hypervisor

Type 2는 기존 OS 위에서 실행되는 방식이다.

```
Hardware
└── Host OS
    └── Hypervisor
        └── VM
```

개발자 로컬 환경에서 흔히 사용한다.

예시는 다음과 같다.

- VirtualBox
- VMware Workstation
- Parallels

---

# AWS와 Nitro

AWS는 EC2를 제공하기 위해 가상화 기술을 사용한다.

Nitro System은 가상화, 네트워크, 스토리지 기능을 전용 하드웨어와 경량 Hypervisor로 분리해 성능과 보안을 높인 구조다.

백엔드 개발자는 Nitro 내부 구현을 깊게 알 필요는 없지만, EC2가 물리 서버가 아니라 가상화된 컴퓨팅 리소스라는 점은 이해해야 한다.

---

# 기억해야 할 내용

- Hypervisor는 물리 자원을 VM에 나누어 주는 계층이다.
- Type 1은 서버에 직접 올라가고, Type 2는 Host OS 위에서 동작한다.
- EC2는 AWS의 가상화 기술 위에서 실행된다.


