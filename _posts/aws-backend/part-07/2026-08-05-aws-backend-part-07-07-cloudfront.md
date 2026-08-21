---
title: "Chapter 07. CloudFront"
permalink: /aws-backend/part-07/07-cloudfront/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. CloudFront
## S3 원본을 보호하며 전 세계에 빠르게 배포하기

> **학습 목표**
>
> - CloudFront와 Edge Location의 역할을 설명할 수 있다.
> - Cache Hit과 Cache Miss의 흐름을 이해한다.
> - S3 오리진을 OAC로 보호할 수 있다.
> - 캐시 키, TTL과 무효화 전략을 설계할 수 있다.

---

# 왜 CloudFront가 필요한가

서울 리전의 S3에 저장한 상품 이미지를 한국, 유럽과 미국 사용자가 반복해서 내려받는다고 가정한다.

모든 요청이 S3 원본까지 이동하면 먼 사용자의 지연 시간이 커지고 같은 객체를 반복해서 전송한다.

문제를 피하려고 Bucket을 공개하면 사용자가 CloudFront를 우회해 S3 URL로 직접 접근할 수 있고 원본 보호가 어려워진다.

콘텐츠를 사용자 가까운 Edge에 캐시하고 S3는 CloudFront에서 온 요청만 허용하면 성능과 원본 보호를 함께 달성할 수 있다.

이 역할을 수행하는 AWS의 글로벌 CDN이 CloudFront이다.

---

# CloudFront란?

CloudFront는 전 세계 Edge Location을 이용해 정적·동적 콘텐츠를 전달하는 글로벌 콘텐츠 전송 네트워크이다.

배포(Distribution)는 사용자 요청을 받을 도메인, 오리진, 캐시 동작과 보안 정책을 정의한다.

S3, ALB, API Gateway 또는 사용자 지정 HTTP 서버를 오리진으로 사용할 수 있다.

S3는 VPC 밖의 리전 서비스이고 CloudFront는 글로벌 Edge 서비스이므로 두 서비스의 범위를 구분해야 한다.

CloudFront는 캐시뿐 아니라 HTTPS 종료, 압축, 지리 제한, Signed URL과 AWS WAF 연동을 제공한다.

---

# 동작 흐름

```
Browser
   │ GET /products/42/image.webp
   ▼
CloudFront Edge
   ├── Cache Hit ─────────────▶ Browser
   │
   └── Cache Miss
          │ OAC 서명 요청
          ▼
      Private S3 Origin
          │ Object
          ▼
      Edge Cache ─────────────▶ Browser
```

1. DNS가 사용자 요청을 적절한 CloudFront Edge로 전달한다.
2. Edge는 캐시 키로 저장된 응답이 있고 유효한지 확인한다.
3. Cache Hit이면 S3에 접근하지 않고 캐시된 객체를 반환한다.
4. Cache Miss이면 CloudFront가 OAC를 사용해 S3 오리진에 서명된 요청을 보낸다.
5. S3 응답은 정책에 따라 Edge에 캐시되고 사용자에게 전달된다.
6. TTL이 끝나면 다음 요청에서 오리진 검증 또는 새 응답 가져오기가 수행된다.

캐시 키에 포함되는 경로, Query String, Header와 Cookie가 달라지면 별도 캐시 항목이 만들어진다.

---

# S3 직접 제공과 CloudFront 비교

| 구분 | S3 직접 제공 | CloudFront 경유 |
|------|--------------|-----------------|
| 사용자 접점 | 리전 엔드포인트 | 글로벌 Edge |
| 반복 요청 | 매번 S3 접근 | Cache Hit은 Edge 응답 |
| 원본 보호 | S3 정책으로 직접 제어 | 비공개 S3와 OAC 조합 |
| 사용자 도메인 | S3 도메인 중심 | 사용자 도메인과 HTTPS |
| 접근 제한 | S3 Presigned URL | Signed URL·Cookie 가능 |
| 비용 요소 | S3 요청과 전송 | CloudFront 요청·전송·무효화 |

OAC는 과거의 OAI보다 최신 방식이며 S3 오리진 접근 제어에는 OAC를 우선 사용한다.

---

# 캐시 정책과 무효화

| 항목 | 역할 | 잘못 설계했을 때 |
|------|------|------------------|
| Cache Key | 같은 응답을 공유할 요청 결정 | 캐시 파편화 또는 잘못된 공유 |
| TTL | 캐시 유효 기간 결정 | 오래된 콘텐츠 또는 낮은 적중률 |
| Cache-Control | 원본이 캐시 지침 전달 | 배포 설정과 충돌 가능 |
| Invalidation | 기존 캐시 강제 제거 | 잦은 호출 시 비용과 운영 복잡도 증가 |
| Versioned URL | 새 Key로 새 콘텐츠 배포 | 참조 URL 갱신 필요 |

정적 자산은 `app.a1b2c3.js`처럼 콘텐츠 해시를 Key에 넣고 긴 TTL을 사용하면 무효화 없이 안전하게 교체할 수 있다.

---

# AWS CLI에서는

배포 목록과 특정 배포 상태를 확인한다.

```bash
aws cloudfront list-distributions

aws cloudfront get-distribution \
  --id DISTRIBUTION_ID
```

긴급하게 변경된 경로의 캐시를 무효화한다.

```bash
aws cloudfront create-invalidation \
  --distribution-id DISTRIBUTION_ID \
  --paths "/products/42/image.webp"
```

배포 전체를 의미하는 `/*` 무효화는 편리하지만 버전 Key 전략을 대신하는 일상 배포 방식으로 남용하지 않는다.

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot는 업로드 후 S3 Key를 저장하고 조회 응답에서 CloudFront 도메인과 결합할 수 있다.

```yaml
app:
  asset:
    cloud-front-domain: https://cdn.example.com
```

도메인을 생성자 주입으로 받아 URL 생성 책임을 한곳에 둔다.

```java
@Service
public class AssetUrlService {

    private final URI cloudFrontDomain;

    public AssetUrlService(
            @Value("${app.asset.cloud-front-domain}") URI cloudFrontDomain) {
        this.cloudFrontDomain = cloudFrontDomain;
    }

    public URI createPublicUrl(String objectKey) {
        String encodedPath = Arrays.stream(objectKey.split("/"))
                .map(segment -> UriUtils.encodePathSegment(
                        segment, StandardCharsets.UTF_8))
                .collect(Collectors.joining("/"));

        return cloudFrontDomain.resolve("/" + encodedPath);
    }
}
```

CloudFront 운영 API가 필요하면 AWS SDK for Java v2의 `CloudFrontClient`를 별도 Bean으로 등록한다.

```java
@Configuration
public class CloudFrontConfiguration {

    @Bean
    CloudFrontClient cloudFrontClient() {
        return CloudFrontClient.builder()
                .region(Region.AWS_GLOBAL)
                .build();
    }
}
```

애플리케이션 요청마다 Invalidation을 호출하지 말고 배포 파이프라인이나 제한된 운영 기능에서만 사용한다.

---

# 실무에서는 어떻게 사용할까

S3 Block Public Access를 유지하고 Bucket 정책에서 특정 CloudFront Distribution의 OAC 요청만 `GetObject`로 허용한다.

사용자 도메인을 CloudFront에 연결하고 ACM 인증서로 HTTPS를 제공하며 인증서 리전 요구사항을 확인한다.

공개 상품 이미지는 긴 TTL과 버전 Key를 사용하고 사용자별 비공개 파일은 Signed URL 또는 Signed Cookie를 검토한다.

응답의 `Age`, `X-Cache`, Cache Hit 비율과 오리진 요청량을 관찰하여 캐시 정책을 조정한다.

---

# 장애 사례와 주의할 점

S3 Bucket을 공개한 채 CloudFront만 추가하면 사용자가 원본 URL로 우회할 수 있어 OAC의 보호 효과가 없다.

인증 헤더나 사용자별 Query String을 캐시 키에서 제외하면 한 사용자의 비공개 응답이 다른 사용자에게 노출될 수 있다.

반대로 모든 Header와 Cookie를 캐시 키에 포함하면 요청마다 키가 달라져 Cache Hit 비율이 급격히 떨어진다.

객체를 같은 Key로 덮어쓴 뒤 캐시가 즉시 바뀔 것으로 기대하면 TTL 동안 오래된 이미지가 보일 수 있다.

---

# 비용과 성능 고려사항

CloudFront는 요청 수, Edge에서 사용자로 전송되는 데이터와 무효화 사용량 등이 비용에 영향을 준다.

Cache Hit 비율이 높아지면 S3 원본 요청과 S3에서 나가는 데이터 전송을 줄이고 사용자 지연 시간도 낮출 수 있다.

캐시 키 차원을 최소화하되 응답이 실제로 달라지는 값은 반드시 포함해야 한다.

압축 가능한 텍스트 자산은 자동 압축을 활용하고 이미 압축된 이미지와 동영상에는 적절한 포맷과 크기를 사용한다.

모든 경로 무효화를 반복하기보다 콘텐츠 버전 Key와 TTL을 조합하는 것이 예측 가능하다.

---

# 기억해야 할 내용

- CloudFront는 사용자 가까운 Edge에서 콘텐츠를 제공하는 글로벌 CDN이다.
- Cache Hit은 S3 원본 요청과 응답 지연을 줄인다.
- S3 Bucket은 비공개로 유지하고 OAC로 CloudFront만 접근시킨다.
- 캐시 키에는 응답을 실제로 바꾸는 값만 포함한다.
- 정적 자산은 버전 Key와 긴 TTL을 사용한다.
- Invalidation은 긴급 변경 수단으로 제한한다.
- 비공개 콘텐츠는 Signed URL 또는 Signed Cookie를 검토한다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 08. Part 7 Summary**이다.

Bucket과 Object부터 Versioning, Lifecycle, Presigned URL, Multipart Upload와 CloudFront까지 연결해 S3 기반 파일 서비스의 전체 흐름을 정리한다.


