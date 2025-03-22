---
layout: post
title: "쿠버네티스 핵심 개념 정리"
date: 2025-03-18
categories: [Kubernetes]
---

# 쿠버네티스 핵심 개념 정리

## 1. 서비스(Service)
### 목적
- 클러스터 내부의 파드를 외부에서 접근할 수 있도록 연결

### 주요 구성요소
- **selector**: 파드의 라벨과 매핑되어 연결
- **ports**: 서비스 포트 설정
- **type**: 
  - `ClusterIP`: 클러스터 내부 접근만 가능
  - `NodePort`: 외부 접근 가능

### 예시
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort
```

## 2. 레플리카셋(ReplicaSet)
### 목적
- 파드의 안정적인 운영을 위한 복제 관리

### 특징
- 운영 안정성을 위해 보통 2개 이상의 레플리카 지정
- `kubectl get all`로 상태 확인 가능
  - `current`: 대기 중인 수
  - `ready`: 실제 운용 중인 수

## 3. 디플로이먼트(Deployment)
### 목적
- 무중단 롤링 업데이트 지원

### 주요 기능
- 레플리카셋 자동 생성 및 관리
- 롤백 기능 지원

### 주요 명령어
```bash
kubectl rollout status deployment [이름]
kubectl rollout history deploy [이름]
kubectl rollout undo deploy [이름] --to-revision=[버전]
```

## 4. 유용한 쿠버네티스 명령어
```bash
kubectl get all                    # 모든 리소스 조회
kubectl get pods                   # 파드 조회
kubectl get po --show-labels       # 파드 라벨 조회
kubectl apply -f [경로]            # 지정 경로의 모든 파일 적용
kubectl describe rs [이름]         # 레플리카셋 상세 정보 조회
```

## 5. 문제 해결
### 일반적인 문제
- `ImagePullBackOff`: 이미지를 가져오지 못할 때 발생하는 상태

### 무중단 전환 방법
1. 여러 라벨 생성 후 셀렉터에서 라벨 스위칭
2. 디플로이먼트를 통한 롤링 업데이트

## 참고 문서
- [쿠버네티스 서비스 문서](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
- [쿠버네티스 레플리카셋 문서](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [쿠버네티스 디플로이먼트 문서](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)


