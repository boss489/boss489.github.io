---
title: "Chapter 03. Virtualization"
permalink: /aws-backend/part-01/03-virtualization/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Virtualization
## 하나의 서버를 여러 서버처럼 사용하는 기술

> **학습 목표**
>
> - Virtual Machine이 왜 등장했는지 설명할 수 있다.
> - Host OS, Guest OS, VM의 관계를 이해한다.
> - Resource Isolation의 의미를 설명할 수 있다.

---

# 왜 가상화가 필요했는가

물리 서버 하나에 애플리케이션 하나만 올리면 자원이 낭비된다.

CPU와 메모리가 남아 있어도 다른 서비스가 쉽게 사용할 수 없었다.

가상화는 하나의 물리 서버를 여러 개의 가상 서버처럼 나누어 사용하게 만든다.

![Virtualization stack](/assets/aws-backend/virtualization-stack.png)

---

# Virtual Machine

Virtual Machine은 소프트웨어로 만든 서버이다.

각 VM은 독립된 OS를 가진다.

```
Physical Server
└── Hypervisor
    ├── VM 1
    ├── VM 2
    └── VM 3
```

VM마다 CPU, Memory, Disk, Network가 할당된다.

---

# Resource Isolation

Resource Isolation은 한 VM의 문제가 다른 VM에 영향을 주지 않도록 자원을 분리하는 것이다.

예를 들어 VM 1의 프로세스가 CPU를 많이 사용해도 VM 2가 완전히 멈추지 않도록 제한할 수 있다.

이 격리 덕분에 여러 서비스를 같은 물리 서버 위에서 운영할 수 있다.

---

# 기억해야 할 내용

- 가상화는 물리 서버 자원을 더 효율적으로 사용하기 위해 등장했다.
- VM은 독립된 OS를 가진 가상 서버이다.
- Resource Isolation은 서비스 간 영향을 줄이는 핵심 개념이다.
- Cloud의 EC2도 가상화 기술 위에서 동작한다.


