---
title: "쿠버네티스 Persistent Volume 정리"
categories:
  - kubernetes
tags:
  - kubernetes
  - storage
  - pv
last_modified_at: 2026-06-17T09:00:00+09:00
---

# 쿠버네티스 Persistent Volume 정리

쿠버네티스에서 애플리케이션은 쉽게 재시작되고 재배포된다. 문제는 컨테이너가 재생성될 때 로컬 파일 시스템에 저장한 데이터는 함께 사라질 수 있다는 점이다. 데이터베이스나 파일 업로드 서비스처럼 상태를 가지는 워크로드에서는 영속 스토리지를 반드시 고려해야 한다.

## 왜 Persistent Volume이 필요한가

테스트 중 MongoDB Pod가 내려갔다가 다시 올라오면서 데이터가 모두 사라지는 경우를 자주 겪는다. 이는 Pod 내부 파일 시스템이 기본적으로 일시적이기 때문이다.

Pod가 재시작되면 다음과 같은 일이 발생할 수 있다.

- 컨테이너 파일 시스템 초기화
- 새 Node로 스케줄링
- 기존 데이터 디렉터리 유실

이 문제를 해결하기 위해 쿠버네티스는 Persistent Volume 관련 리소스를 제공한다.

## Persistent Volume과 Persistent Volume Claim

### Persistent Volume(PV)
- 클러스터 차원에서 준비된 실제 스토리지 자원
- NFS, EBS, Ceph, 로컬 디스크 등 다양한 백엔드와 연결 가능
- 인프라 관점의 리소스

### Persistent Volume Claim(PVC)
- 애플리케이션이 필요한 스토리지를 요청하는 선언
- 용량, 접근 모드 등을 기준으로 PV를 바인딩
- 애플리케이션 관점의 리소스

간단히 말하면 PV는 "실제 디스크", PVC는 "그 디스크를 사용하겠다는 요청"이다.

## 동작 흐름

1. 관리자가 PV를 만들거나 StorageClass를 통해 동적 프로비저닝을 구성한다.
2. 애플리케이션은 PVC를 생성한다.
3. 쿠버네티스가 조건에 맞는 PV를 바인딩한다.
4. Pod가 PVC를 마운트하여 데이터를 사용한다.

이 구조 덕분에 애플리케이션은 스토리지 구현 세부사항을 몰라도 된다.

## AWS 환경에서는 어떻게 연결되는가

AWS에서는 대표적으로 EBS가 Pod의 영속 스토리지로 사용된다.

- Amazon EBS: 블록 스토리지
- EC2 인스턴스에 연결되는 가상 디스크 개념
- Kubernetes에서는 CSI(Container Storage Interface) 드라이버를 통해 연동

즉, 쿠버네티스가 직접 데이터를 보존하는 것이 아니라, 외부 스토리지 시스템과 연결해서 데이터 생명주기를 Pod보다 길게 가져가는 구조다.

## 기본적인 활용 예시

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Pod에서는 이 PVC를 볼륨으로 마운트한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
    - name: mongodb
      image: mongo
      volumeMounts:
        - name: mongo-data
          mountPath: /data/db
  volumes:
    - name: mongo-data
      persistentVolumeClaim:
        claimName: mongodb-pvc
```

이렇게 구성하면 Pod가 재시작되어도 데이터는 스토리지에 남아 있게 된다.

## 운영 중 자주 사용하는 명령어

```bash
# 전체 리소스 삭제
kubectl delete -f pods.yaml

# 로그 확인
kubectl logs <pod-name>
kubectl logs -f <pod-name>

# PV/PVC 상태 확인
kubectl get pv
kubectl get pvc
kubectl describe pvc mongodb-pvc
```

## 실무에서 중요한 고려 사항

### 1. 접근 모드
- `ReadWriteOnce`: 하나의 노드에서 읽기/쓰기
- `ReadOnlyMany`: 여러 노드에서 읽기 전용
- `ReadWriteMany`: 여러 노드에서 읽기/쓰기

워크로드 특성에 따라 적절한 모드를 선택해야 한다.

### 2. StorageClass와 동적 프로비저닝
운영 환경에서는 수동 PV 생성보다 StorageClass를 통해 필요한 시점에 볼륨을 자동 생성하는 방식이 더 일반적이다. 특히 클라우드 환경에서는 이 방식이 관리 효율이 높다.

### 3. Pod 재시작과 데이터 보존은 별개
Deployment가 잘 동작한다고 해서 데이터가 안전한 것은 아니다. 애플리케이션 가용성과 데이터 내구성은 별개의 문제이며, 스토리지 설계가 별도로 필요하다.

### 4. 상태 저장 애플리케이션은 StatefulSet도 검토
DB, 메시지 브로커, 캐시처럼 인스턴스 정체성이 중요한 경우에는 Deployment보다 StatefulSet이 더 적합할 수 있다.

## 결론

쿠버네티스에서 상태 저장 워크로드를 운영하려면 Persistent Volume 개념을 정확히 이해해야 한다.

- Pod는 언제든 교체될 수 있다.
- 컨테이너 파일 시스템은 영속 저장소가 아니다.
- 데이터 보존은 PV, PVC, StorageClass 같은 스토리지 리소스로 해결해야 한다.

특히 AWS 환경에서는 EBS 기반 영속 볼륨 구성을 이해해두면 대부분의 기본적인 데이터 보존 요구사항을 안정적으로 처리할 수 있다.
