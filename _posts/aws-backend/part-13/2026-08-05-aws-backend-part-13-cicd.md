---
title: "Part 13. CI/CD"
permalink: /aws-backend/part-13/
date: 2026-08-05T09:04:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 13. CI/CD
## 배포 자동화

> 목표
>
> Spring Boot 서비스를 AWS에 자동으로 빌드하고 배포하는 흐름을 이해한다.

---

# 학습 내용

- [Chapter 01. CI/CD Overview](/aws-backend/part-13/01-cicd-overview/)
- [Chapter 02. GitHub Actions](/aws-backend/part-13/02-github-actions/)
- [Chapter 03. CodeBuild](/aws-backend/part-13/03-codebuild/)
- [Chapter 04. CodePipeline](/aws-backend/part-13/04-codepipeline/)
- [Chapter 05. CodeDeploy](/aws-backend/part-13/05-codedeploy/)
- [Chapter 06. Rolling Deployment](/aws-backend/part-13/06-rolling-deployment/)
- [Chapter 07. Blue/Green Deployment](/aws-backend/part-13/07-blue-green-deployment/)
- [Chapter 08. Deployment Safety](/aws-backend/part-13/08-deployment-safety/)
- [Chapter 09. Summary](/aws-backend/part-13/09-summary/)
- [Chapter 10. Interview Questions](/aws-backend/part-13/10-interview/)

---

# 학습 흐름

```text
Source Commit
     │
     ▼
CI Test & Build
     │
     ▼
Immutable Artifact
     │
     ▼
Staging Verification
     │
     ▼
Approval & Production Deployment
     │
     ▼
Monitoring & Rollback
```

각 Chapter는 Spring Boot 3.x와 Java 17 애플리케이션을 기준으로 Source 검증부터 ECS 배포 안전장치까지 연결한다.

---

# 완료 후 설명할 수 있어야 하는 것

- CI와 CD의 차이
- GitHub Actions와 AWS 배포 서비스의 역할
- Rolling Update와 Blue/Green 배포의 차이
- 배포 실패 시 롤백해야 하는 지점
- Build Once, Deploy Many와 Artifact Provenance
- OIDC 기반 AWS Role Assume과 최소 권한
- DB Migration, Feature Flag와 Alarm 기반 배포 안전장치

---

다음 Part

→ **Part 14. Cost Optimization**

