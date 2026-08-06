---
title: "Chapter 03. Versioning"
permalink: /aws-backend/part-07/03-versioning/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Versioning
## 객체 변경 이력을 남기는 기능

> **학습 목표**
>
> - Versioning의 목적을 설명할 수 있다.
> - 삭제와 덮어쓰기 복구 가능성을 이해한다.
> - Versioning 사용 시 비용 증가를 설명할 수 있다.

---

# Versioning이란?

Versioning은 같은 Object Key에 여러 버전을 보관하는 기능이다.

실수로 파일을 덮어쓰거나 삭제했을 때 이전 버전을 복구할 수 있다.

---

# Delete Marker

Versioning이 켜진 Bucket에서 삭제하면 실제 객체가 바로 사라지는 것이 아니라 Delete Marker가 추가된다.

이전 버전은 남아 있을 수 있다.

---

# 비용

버전이 계속 쌓이면 저장 비용이 증가한다.

Lifecycle 정책으로 오래된 버전을 정리해야 한다.

---

# 기억해야 할 내용

Versioning은 복구 가능성을 높이지만 비용도 늘린다.

중요 파일 Bucket에 적용하고 Lifecycle과 함께 설계한다.


