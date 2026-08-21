---
title: "Chapter 03. API Gateway"
permalink: /aws-backend/part-10/03-api-gateway/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. API Gateway
## 외부 HTTP 요청을 안전하게 백엔드로 연결하기

> **학습 목표**
>
> - API Gateway의 역할을 설명할 수 있다.
> - Lambda Proxy Integration 흐름을 이해한다.
> - 인증, 제한, 로깅 기능을 설명할 수 있다.
> - REST API와 HTTP API의 선택 기준을 설명할 수 있다.

---

# 왜 API Gateway가 필요한가

Lambda 함수는 코드를 실행하지만 브라우저와 모바일 앱이 호출할 안정적인 공개 URL, HTTP 라우팅, 인증, 요청 제한을 그 자체로 제공하는 것은 아니다.

클라이언트가 함수를 직접 호출하게 만들면 AWS 자격 증명과 호출 권한을 클라이언트에 배포해야 하고 API 경로와 오류 응답을 일관되게 관리하기 어렵다.

주문 API라면 `/orders` 경로, JWT 인증, CORS, 요청별 로그, 과도한 호출 제한을 백엔드 코드마다 중복 구현하지 않는 진입점이 필요하다.

API Gateway는 외부 요청과 Lambda 사이에서 **API 계약과 공통 정책을 관리하는 관리형 관문** 역할을 한다.

---

# API Gateway란?

API Gateway는 HTTP, REST, WebSocket API를 만들고 요청을 Lambda, HTTP 엔드포인트, 일부 AWS 서비스로 전달하는 관리형 서비스다.

서버리스 API의 앞단 역할을 한다.

![Serverless flow](/assets/aws-backend/serverless-flow.png)

---

# 동작 흐름

Lambda Proxy Integration을 사용하는 주문 조회 흐름은 다음과 같다.

```
Client
  │ GET /orders/100
  │ Authorization: Bearer ...
  ▼
API Gateway
  ├──▶ Route 선택
  ├──▶ 인증·인가
  ├──▶ 제한·Access Log
  ▼
Lambda
  │ orderId=100 조회
  ▼
DynamoDB
  │ 결과
  ▼
Lambda Response ──▶ API Gateway ──▶ Client
```

1. Custom Domain과 Stage가 요청을 대상 API로 연결한다.
2. API Gateway는 메서드와 경로에 맞는 Route를 선택한다.
3. 설정된 Authorizer나 IAM 정책이 호출 권한을 확인한다.
4. Proxy Integration은 경로, Query String, Header, Body를 Lambda 이벤트로 전달한다.
5. Lambda는 `statusCode`, `headers`, `body` 형식으로 응답하고 API Gateway가 HTTP 응답으로 변환한다.

---

# Lambda Proxy Integration

Proxy Integration은 API Gateway가 HTTP 요청 정보를 표준 이벤트 구조로 전달하고 Lambda 응답을 거의 그대로 HTTP 응답으로 사용하는 방식이다.

Lambda가 반환하는 `body`는 일반적으로 JSON 객체 자체가 아니라 직렬화된 문자열이어야 한다.

```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"orderId\":\"100\",\"status\":\"PAID\"}"
}
```

---

# REST API와 HTTP API 비교

| 기준 | REST API | HTTP API |
|---|---|---|
| 목적 | 다양한 API 관리 기능 | 낮은 지연의 단순 HTTP API |
| 기능 범위 | 변환, 캐시, API Key 등 폭넓다. | 핵심 라우팅과 JWT/OIDC 연동에 집중한다. |
| 비용 구조 | 기능이 많은 만큼 상대적으로 복잡하다. | 일반적으로 단순하고 비용 효율적이다. |
| 요청 형식 | REST API Payload 형식을 사용한다. | HTTP API Payload 형식을 선택한다. |
| 선택 기준 | REST API 전용 기능이 필요하다. | Lambda/HTTP Proxy 중심의 새 API이다. |

두 유형은 기능과 이벤트 Payload 형식이 다르므로 가격만 보고 바꾸지 말고 필요한 인증, 변환, 캐시, 사용량 계획 기능을 먼저 확인한다.

---

# 주요 기능

| 기능 | 역할 | 주의점 |
|---|---|---|
| Route | 메서드와 경로를 Integration에 연결한다. | 경로 우선순위를 확인한다. |
| Authorizer | JWT 또는 사용자 인증 결과를 검증한다. | 인증과 업무 인가를 구분한다. |
| Throttling | 과도한 요청을 제한한다. | 하류 Lambda 동시성과 함께 조정한다. |
| CORS | 브라우저 교차 출처 요청을 제어한다. | 허용 Origin을 무분별하게 열지 않는다. |
| Access Log | 요청, 응답, 지연 정보를 기록한다. | 민감한 Header와 Body를 제외한다. |
| Custom Domain | 서비스 도메인과 인증서를 연결한다. | Route53과 인증서 Region을 확인한다. |

---

# 설정 예시

다음 CLI는 HTTP API를 만들고 Lambda Proxy Integration과 Route를 연결하는 핵심 흐름을 보여준다.

```bash
API_ID=$(aws apigatewayv2 create-api \
  --name order-api \
  --protocol-type HTTP \
  --query ApiId \
  --output text)

INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id "$API_ID" \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:ap-northeast-2:123456789012:function:order-api \
  --payload-format-version 2.0 \
  --query IntegrationId \
  --output text)

aws apigatewayv2 create-route \
  --api-id "$API_ID" \
  --route-key "GET /orders/{orderId}" \
  --target "integrations/$INTEGRATION_ID"
```

---

# Spring Boot / Java에서는 어떻게 쓰는가

작은 API는 HTTP API Payload 2.0 이벤트를 직접 받는 `RequestHandler<I, O>`로 구현하면 초기화 비용을 줄일 수 있다.

```java
public final class GetOrderHandler implements RequestHandler<
        APIGatewayV2HTTPEvent, APIGatewayV2HTTPResponse> {
    private final ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public APIGatewayV2HTTPResponse handleRequest(
            APIGatewayV2HTTPEvent event, Context context
    ) {
        String orderId = event.getPathParameters().get("orderId");
        String body = "{\"orderId\":\"" + orderId + "\"}";
        return APIGatewayV2HTTPResponse.builder()
                .withStatusCode(200)
                .withHeaders(Map.of("Content-Type", "application/json"))
                .withBody(body)
                .build();
    }
}
```

Spring Cloud Function은 함수형 Bean과 API Gateway 이벤트 어댑터를 조합할 수 있어 도메인 로직을 AWS 이벤트 타입에서 분리하기 좋다.

기존 Spring MVC Controller와 Filter를 최대한 유지해야 한다면 Serverless Java Container의 `AWSLambdaStreamHandler` 방식을 검토할 수 있다.

Spring Boot 전체를 Lambda에 올리면 JVM 콜드 스타트가 커질 수 있으므로 의존성 축소, SnapStart, Provisioned Concurrency, GraalVM 네이티브 이미지의 효과와 비용을 실제 요청으로 측정한다.

**긴 처리, 스트리밍 응답, 지속 연결, 항상 낮은 지연이 필요한 대규모 Spring Boot API는 ALB와 ECS/EKS가 더 적합할 수 있다.**

---

# 실무에서는 어떻게 사용할까

| 시나리오 | 구성 | 핵심 설계 |
|---|---|---|
| 모바일 API | HTTP API → Lambda | JWT와 응답 계약을 적용한다. |
| 파트너 API | REST API → Lambda/ECS | 사용량 제한과 감사 로그를 둔다. |
| 사내 API | Private API 또는 VPC Link | 네트워크 접근 범위를 제한한다. |
| 점진적 이전 | API Gateway → Lambda와 기존 HTTP 서비스 | Route 단위로 백엔드를 분리한다. |

---

# 장애 사례

Lambda가 API Gateway의 통합 제한 시간 안에 응답하지 못하면 Lambda가 나중에 완료되더라도 클라이언트는 Timeout 오류를 받는다.

API Gateway REST API의 Lambda 통합에서 널리 사용하는 기본적인 최대 통합 Timeout은 29초이므로 긴 작업은 즉시 작업 ID를 반환하고 SQS나 Step Functions로 넘기는 비동기 구조를 고려한다.

잘못된 Proxy 응답 형식은 통합 오류를 만들 수 있으므로 `statusCode`, `headers`, 문자열 `body`를 계약 테스트로 검증한다.

Authorizer 장애나 캐시 설정 오류는 정상 사용자의 모든 요청을 막을 수 있으므로 인증 지연, 거부율, 캐시 TTL을 관찰한다.

스로틀링 설정이 Lambda 동시성보다 지나치게 높으면 API Gateway를 통과한 요청이 Lambda에서 다시 제한되어 재시도 폭증이 생길 수 있다.

---

# 주의할 점

Access Log에 Authorization Header, Cookie, 개인정보가 포함되지 않도록 로그 포맷을 검토한다.

클라이언트가 Timeout 후 같은 POST 요청을 재시도할 수 있으므로 결제와 주문 생성 API에는 멱등성 Key를 받는 방식을 고려한다.

인증은 사용자가 누구인지 확인하는 과정이고 인가는 해당 주문을 조회할 권한이 있는지 판단하는 도메인 로직이므로 둘을 분리한다.

---

# 비용과 성능 고려사항

API Gateway 비용은 요청 수, 데이터 전송량, 선택한 API 유형과 부가 기능의 영향을 받는다.

Lambda와 결합하면 API Gateway 요청 비용 외에 Lambda 요청·실행 시간, CloudWatch 로그, 외부 데이터 전송 비용도 포함된다.

REST API 캐시는 반복 조회의 백엔드 부하를 줄일 수 있지만 데이터 신선도, 무효화, 별도 캐시 비용을 함께 고려한다.

HTTP API는 필요한 기능이 충족되는 단순 Proxy API에서 비용과 지연을 줄이는 선택지가 될 수 있다.

응답 크기와 직렬화 시간도 비용과 지연에 영향을 주므로 필요한 필드만 반환하고 압축과 Pagination을 검토한다.

---

# 기억해야 할 내용

- API Gateway는 외부 HTTP 요청과 백엔드를 연결하는 관리형 진입점이다.
- Route, Integration, Stage가 API 요청 전달 구조의 핵심이다.
- Lambda Proxy Integration은 요청과 응답 형식을 정확히 지켜야 한다.
- HTTP API와 REST API는 기능 범위와 Payload 형식이 다르다.
- 인증, 인가, CORS, 요청 제한은 서로 다른 문제이다.
- 긴 작업은 동기 요청에서 끝내지 말고 비동기 워크플로우로 전환한다.
- 로그에는 요청 ID를 남기고 민감 정보는 제외한다.

---

# 다음 Chapter

다음 Chapter에서는 서비스 사이의 사건을 느슨하게 연결하는 **EventBridge**를 학습한다.

Event Bus, Rule, Target의 관계와 주문 완료 이벤트를 여러 시스템으로 안전하게 전달하는 방법을 알아본다.

