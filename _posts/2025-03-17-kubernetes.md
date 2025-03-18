---
title: "Kubernetes 핸즈온 가이드"
categories:
  - kubernetes
tags:
  - kubernetes
  - docker
  - minikube
last_modified_at: 2025-03-18T06:31:52-07:00
---

# Kubernetes 로컬 개발 환경 설정

## Minikube 설치 및 설정
- Minikube를 활용하여 로컬에서 쿠버네티스 환경을 구성할 수 있습니다.
- Docker Desktop이 설치되어 있다면 간단히 Kubernetes 환경을 활성화할 수 있습니다.


# Kubernetes 기본 개념

## Pod란?
Pod는 Kubernetes에서 가장 작은 배포 단위입니다. 하나 이상의 컨테이너를 포함하며, 공유 네트워크와 스토리지 공간을 가집니다.

- [Pod 공식 문서](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes API 참조](https://kubernetes.io/docs/reference/using-api/)

## Kubernetes 기본 명령어

### 클러스터 리소스 확인
```bash
# 클러스터의 모든 리소스 확인
kubectl get all
```

### Pod 관리
```bash
# Pod 생성
kubectl apply -f first-pod.yaml

# Pod 상태 확인
kubectl describe pod {podname}

# Pod 내부 명령어 실행
kubectl exec {podname} -- {command}

# Pod에 대화형 셸 접속
kubectl -it exec {podname} -- sh
```

## Pod 접근성
- Pod는 기본적으로 클러스터 외부에서 직접 접근할 수 없습니다.
- Service나 Ingress를 통해 외부 접근을 설정해야 합니다.

## Pod 문제 해결
### Pod 상태 확인
```bash
# Pod 상세 정보 확인
kubectl describe pod {podname}

# Pod 로그 확인
kubectl logs {podname}

# Pod 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'
```

### Pod 문제 해결
1. Pod 상태 확인
2. 이벤트 로그 확인
3. 컨테이너 로그 확인
4. 필요한 경우 Pod 재시작

## 참고 자료
- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Docker 공식 문서](https://docs.docker.com/)
