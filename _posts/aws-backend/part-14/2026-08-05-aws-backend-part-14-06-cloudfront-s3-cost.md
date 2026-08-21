---
title: "Chapter 06. CloudFront와 S3 비용"
permalink: /aws-backend/part-14/06-cloudfront-s3-cost/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. CloudFront와 S3 비용
## 캐시 적중률로 원본 요청과 전송 줄이기

> **학습 목표**
>
> - CloudFront와 S3의 요청 및 데이터 전송 비용 흐름을 설명할 수 있다.
> - Cache Hit Ratio에 영향을 주는 캐시 키와 TTL을 조정할 수 있다.
> - 압축, 버전 URL과 Invalidation의 트레이드오프를 판단할 수 있다.

---

# 비용 폭증 시나리오

이미지 요청 수는 비슷한데 CloudFront와 S3 비용이 동시에 증가한 상황을 가정한다.

배포 후 모든 URL에 무작위 Query String이 붙으면 캐시 키가 파편화되어 Cache Miss가 늘 수 있다.

짧은 TTL과 반복적인 전체 Invalidation도 Edge 재사용을 낮추고 원본 요청을 증가시킨다.

---

# 핵심 정의

## Cache Hit Ratio

전체 캐시 요청 중 Edge가 원본 접근 없이 응답한 비율이다.

## TTL

응답을 캐시에서 유효하게 사용할 수 있는 기간이다.

## Compression

전송 바이트를 줄이기 위해 응답 표현을 압축하는 방식이다.

## Invalidation

TTL 만료 전 CloudFront 캐시 객체를 무효화하는 작업이다.

## Versioned URL

콘텐츠 변경 때 경로나 파일명에 버전을 반영해 새 캐시 키를 만드는 방식이다.

---

# 비용 흐름

```
Browser
  -> CloudFront request + edge transfer
       ├ Hit  -> cached response
       └ Miss -> S3 GET request
                    -> origin transfer
                    -> edge cache
비용 = 요청 + 전송 + 저장 + 선택적 invalidation
```

이 흐름에서 사용량이 생성되는 지점과 청구 데이터에서 보이는 지점을 연결해야 한다.

---

# 비교표

| 전략 | 효과 | 주의점 |
|---|---|---|
| 긴 TTL | 적중률 상승 | 변경 반영 지연 |
| 버전 URL | 안전한 장기 캐시 | 참조 갱신 |
| Invalidation | 즉시 제거 | 남용 시 요청과 비용 |
| 압축 | 전송량 감소 | CPU와 포맷 적합성 |

비교표는 절대적인 우열이 아니라 워크로드에 맞는 선택 기준을 제공한다.

---

# Cost Explorer와 AWS CLI

다음 명령은 예시 기간의 데이터를 조회하며 실행 계정의 권한과 실제 날짜를 확인해야 한다.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=EDFDVBD6EXAMPLE Name=Region,Value=Global \
  --statistics Average --period 3600 \
  --start-time 2026-08-01T00:00:00Z --end-time 2026-08-02T00:00:00Z
aws cloudfront create-invalidation --distribution-id EDFDVBD6EXAMPLE --paths "/assets/app.js"
```

CLI 결과는 청구 확정 시점과 데이터 지연을 고려해 해석한다.

특정 가격 수치는 문서에 고정하지 않고 Region, 시점과 구매 모델에 맞는 공식 가격 정보를 확인한다.

---

# Spring Boot 운영자가 통제할 수 있는 요소

정적 자산 파일명에 콘텐츠 해시를 넣고 긴 `Cache-Control`을 설정한다.

사용자별 응답은 공유 캐시 대상과 분리해 데이터 노출을 막는다.

이미지는 업로드 시 적절한 크기와 포맷으로 변환해 저장과 전송 바이트를 줄인다.

---

# 실무 적용

## 1단계: 기준선 만들기

변경 전 최소 한 주기의 비용, 사용량, 처리량과 SLO를 같은 시간 범위로 저장한다.

## 2단계: 원인 분리하기

Cache Hit Ratio와 S3 GET 요청량을 같은 기간으로 비교해 Miss가 원본 비용에 미치는 영향을 확인한다.

## 3단계: 작은 변경 적용하기

Query String, Header와 Cookie 중 응답을 실제로 바꾸는 값만 캐시 키에 포함한다.

## 4단계: 가격 조건 확인하기

압축 전후 전송량, Edge 지연과 원본 CPU를 함께 측정한다.

## 5단계: 효과 검증하기

변경 후 총비용뿐 아니라 단위 비용, 지연, 오류율과 운영 부담이 어떻게 달라졌는지 확인한다.

---

# 운영 함정

Cache Hit Ratio만 높이려고 사용자별 데이터를 공유하면 심각한 보안 사고가 발생할 수 있다.

이미 압축된 이미지에 무리한 재압축을 적용하면 CPU만 늘고 품질이 떨어질 수 있다.

매 배포마다 `/*`를 무효화하면 버전 URL이 제공하는 캐시 재사용 이점을 잃는다.

---

# 비용과 성능의 Trade-off

긴 TTL은 비용과 지연을 낮추지만 오래된 콘텐츠를 보여줄 가능성을 늘린다.

CloudFront 계층은 자체 요청과 전송 비용이 있지만 원본 요청 감소와 사용자 성능 개선을 함께 평가해야 한다.

최적 정책은 콘텐츠 변경 빈도, 크기, 지역 분포와 일관성 요구에 따라 달라진다.

비용 최적화는 단순한 최저가 선택이 아니라 비즈니스 가치, 가용성, 성능과 운영 가능성 사이의 균형을 찾는 일이다.

---

# 점검 체크리스트

- 비용 변화의 시작 시각과 배포 시각을 비교했는가.

- 서비스, 계정, 태그와 사용 유형으로 비용을 분해했는가.

- 변경 전후의 단위 비용과 SLO를 함께 비교했는가.

- 되돌리기 절차와 담당 owner가 정해져 있는가.

- 최신 Region과 구매 모델의 가격 조건을 확인했는가.

---

# 핵심 요약

캐시 적중률은 원본 S3 요청과 데이터 흐름을 결정하는 핵심 지표다.

TTL과 캐시 키를 안전하게 단순화하고 버전 URL을 우선 사용한다.

요청 수와 전송량을 함께 관찰하며 압축과 Invalidation을 조정한다.

---

# 다음 장

→ **[Chapter 07. Savings Plans와 Reserved Instances](/aws-backend/part-14/07-savings-plans-reserved-instances/)**
