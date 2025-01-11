# AWS re:Invent 2024 메모

## 1차: compute

### 발표 내용
- **Trn2**: 2025년 하반기 출시 예정.
- **P6**: 2025년 상반기 출시 예정.
- **Amazon Nova**: 공식 발표.

### Dr. Werner Vogels 키노트 주요 내용
1. **진화 가능성을 요구사항으로 설정**: 시스템이 진화할 수 있는 능력을 갖추도록 설계.
2. **복잡성을 작은 단위로 분해**:
  - 예: Amazon CloudWatch.
3. **조직을 시스템 복잡성에 맞게 조정**:
  - 팀에게 권한과 재량을 부여하여 오너십을 강화.
4. **셀 기반 아키텍처**: 모듈화되고 확장 가능한 아키텍처 디자인.
5. **예측 가능한 시스템 설계**: 신뢰성과 예측 가능성을 중점적으로 설계.
6. **복잡성의 자동화**:
  - 판단이 필요하지 않은 작업을 자동화.
  - 예: Amazon Bedrock 서버리스 에이전트 워크플로우.

---

## 2차: 생성형 AI

### 주요 테마
1. **데이터와의 통합**: 데이터 통합이 생성형 AI 성공의 핵심.
2. **SageMaker Unified Studio (프리뷰)**:
  - 데이터 분석부터 배포까지의 엔드 투 엔드 서비스.
  - 서드파티 및 AWS 데이터 소스를 통합하는 레이크하우스 아키텍처 지원.
3. **Bedrock IDE**: 생성형 AI 개발 지원 도구.
4. **SageMaker LakeHouse**: 데이터 세트 탐색 및 선택을 간소화.
5. **AI 개발**:
  - 에이전트, RAG(Retrieval-Augmented Generation), 프롬프트 엔지니어링 워크플로우.
  - 예제 워크플로우:
    1. 문서를 S3에 업로드.
    2. 임베딩 모델 선택.
    3. 멀티 에이전트 협업을 통해 고객 불만 해결 및 환불 추천.
6. **구조화된 데이터 검색**:
  - 사용자의 질문을 SQL과 같은 쿼리로 변환.
  - 연결된 데이터 웨어하우스와 통합.

### GitHub 기능
- **Amazon Q in GitLab Duo**:
  - 코드 생성, 리뷰, 수정 및 머지 요청 자동화.

### AWS 모델 파인튜닝 및 개발 환경
- **Amazon Bedrock**
- **SageMaker Training**
- **SageMaker HyperPod**

### 주목할 협력사
- **Anthropic**
- **Poolside**

---

## 3차: 생성형 AI 실제 활용 사례

### 주요 사례
1. **추천 시스템**:
  - 제품 이미지 기반 속성 분석으로 검색 및 설명 개선.
  - 사람의 작업을 자동화하여 생성형 AI의 변혁적 잠재력을 보여줌.
2. **GS SHOP 사례 연구**:
  - 검색 기능에 생성형 AI 통합.
  - 속성 추출을 간소화하여 검색 용이성 향상.
  - 과잉 검색 결과 감소(예: 4,100만 건의 원피스 검색 결과 필터링).
  - 5주 만에 프로젝트 완료.
3. **LG전자 데이터 플랫폼**:
  - 데이터 기반 대시보드 구축.
  - AI 챗봇이 차트를 생성하고 인사이트 제공(예: 텍스트-SQL, 텍스트-차트 기능).
  - 제안된 개선 사항:
    - AWS Bedrock Knowledge Bases를 활용한 구조화된 데이터 검색(프리뷰).
    - Amazon Bedrock Agent 활용.

---

## 롯데 이커머스를 위한 신규 기능 추천

### 데이터베이스 향상
1. **Amazon Aurora Serverless**:
  - Zero ACU 지원.
2. **Amazon Aurora PostgreSQL Limitless**:
  - 샤딩을 통한 성능 향상.
3. **Amazon Aurora DSQL** (NewSQL):
  - 버퍼 풀 및 관련 오버헤드를 제거하여 성능 향상.
  - 같은 행 경합이 많은 경우 비추천.
4. **Amazon CloudWatch Database Insights**:
  - 심층적인 데이터베이스 성능 분석.
5. **AWS DMS와 생성형 AI**:
  - 데이터 마이그레이션을 위한 스키마 변환 향상.
6. **AWS 가격 인하**: 비용 효율적인 엔터프라이즈 데이터베이스 솔루션.

### 컨테이너 및 DevOps
1. **EKS Auto Mode**:
  - Kubernetes 클러스터 인프라 자동화.
2. **OpenSearch on CloudWatch**:
  - 간소화된 검색 및 로그 분석.
3. **AWS Glue 5.0**: 업데이트된 데이터 통합 기능.
4. **S3 Storage Browser for AWS**: 스토리지 관리 도구 강화.

---
