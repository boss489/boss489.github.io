---
title: "Chapter 09. Interview Questions"
permalink: /aws-backend/part-06/09-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Interview Questions
## Cache 면접 질문

---

## Cache Aside란 무엇인가요?

캐시를 먼저 조회하고 없으면 DB에서 읽은 뒤 캐시에 저장하는 패턴입니다.

## Write Through와 Cache Aside의 차이는 무엇인가요?

Write Through는 쓰기 시 캐시도 갱신하고, Cache Aside는 읽기 시 필요한 데이터를 캐시에 채웁니다.

## TTL은 왜 필요한가요?

캐시 데이터가 무한히 남아 오래된 값을 반환하지 않도록 수명을 제한하기 위해 필요합니다.

## Redis Session을 사용하는 이유는 무엇인가요?

여러 서버가 같은 세션 저장소를 공유해 Scale Out 환경에서도 로그인 상태를 유지하기 위해 사용합니다.

## Distributed Lock을 사용할 때 주의할 점은 무엇인가요?

만료 시간을 반드시 두고, Lock보다 DB 트랜잭션이나 제약 조건으로 해결할 수 있는지 먼저 확인해야 합니다.


