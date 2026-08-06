---
title: "Chapter 07. Pub/Sub and ElastiCache"
permalink: /aws-backend/part-06/07-pubsub-elasticache/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Pub/Sub and ElastiCache
## Redis 메시징과 AWS 관리형 Redis

> **학습 목표**
>
> - Redis Pub/Sub의 기본 구조를 설명할 수 있다.
> - ElastiCache의 역할을 이해한다.
> - 운영 Redis에서 고려할 장애 지점을 설명할 수 있다.

---

# Pub/Sub

Pub/Sub은 발행자가 메시지를 채널에 보내고 구독자가 메시지를 받는 구조다.

간단한 실시간 알림이나 캐시 무효화 이벤트에 사용할 수 있다.

하지만 메시지 영속성을 보장하는 큐가 아니므로 중요한 이벤트에는 SQS, Kafka 같은 도구를 검토한다.

---

# ElastiCache

ElastiCache는 AWS의 관리형 캐시 서비스다.

Redis 또는 Memcached를 사용할 수 있다.

Spring Boot 백엔드에서는 Redis 기반 캐시, 세션, Lock에 자주 사용한다.

---

# 운영 고려사항

- Multi-AZ
- 자동 Failover
- 메모리 사용률
- Eviction 정책
- 연결 수
- Security Group

---

# 기억해야 할 내용

ElastiCache는 Redis 운영 부담을 줄인다.

하지만 데이터 모델, TTL, 장애 영향은 애플리케이션이 고려해야 한다.


