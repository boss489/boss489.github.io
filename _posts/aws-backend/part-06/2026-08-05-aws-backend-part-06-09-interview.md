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

## Cache Stampede는 무엇이고 어떻게 줄이나요?

인기 키의 TTL이 동시에 만료되어 많은 요청이 한꺼번에 DB를 조회하는 현상입니다. TTL을 분산하거나, 짧은 Lock·단일 갱신으로 재생성을 한 요청으로 제한하는 방법을 사용합니다.

## 캐시 삭제와 갱신 중 무엇을 선택하나요?

쓰기 뒤 값 전체를 확실히 만들 수 있으면 갱신할 수 있고, 그렇지 않으면 삭제 후 다음 읽기에서 채우는 방식이 단순합니다. 어느 쪽이든 DB 쓰기 성공 뒤 처리해야 합니다.

## Redis가 장애 나면 어떻게 해야 하나요?

캐시 조회 실패가 서비스 전체 장애로 번지지 않도록 timeout을 짧게 두고, DB 우회가 가능한 데이터인지 정합니다. 세션·Lock처럼 Redis가 필수인 기능은 별도의 장애 정책이 필요합니다.

