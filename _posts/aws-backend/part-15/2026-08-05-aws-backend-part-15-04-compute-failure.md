---
title: "Chapter 04. Compute Failure"
permalink: /aws-backend/part-15/04-compute-failure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Compute Failure
## EC2, ECS, EKS 실행 환경 진단

> **학습 목표**
>
> - 공통 자원 포화와 플랫폼별 실패 신호를 구분할 수 있다.
> - Linux, ECS와 Kubernetes 증거를 수집할 수 있다.
> - 재시작 전에 원인 보존과 안전한 완화를 수행할 수 있다.

---

# 실제 장애 징후

응답 지연, timeout, health check 실패, 컨테이너 재시작과 프로세스 종료가 대표 징후다.

CPU, memory, disk, file descriptor 또는 연결 자원이 포화되면 서로 다른 오류가 연쇄적으로 발생한다.

배포 직후 새 인스턴스나 Pod에만 장애가 집중되면 이미지, 설정과 리소스 제한을 의심한다.

---

# 정의와 가능한 원인

Compute 장애는 EC2 호스트, ECS Task, EKS Pod 또는 애플리케이션 프로세스가 기대한 요청을 처리하지 못하는 상태다.

공통 원인은 CPU throttling, memory pressure, OOM kill, disk full, inode 부족, file descriptor 고갈과 포트 포화다.

잘못된 health check, 종료 처리, 환경 변수, IAM 권한과 배포 artifact도 실행 실패를 만든다.

EC2 status check 실패는 인스턴스 내부 문제와 AWS 기반 시스템 문제를 구분해 확인한다.

ECS는 stopped reason과 container reason이 실패 경계를 설명한다.

EKS는 Pod phase, event, restart count와 이전 컨테이너 로그가 핵심 증거다.

---

# 계층 구조

```text
AWS infrastructure
       |
EC2 node or container host
       |
ECS Task / EKS Pod
       |
JVM process
       |
Thread / heap / file / socket
       |
Spring Boot request
```

상위 오케스트레이터가 보이는 재시작은 원인이 아니라 하위 자원 문제의 결과일 수 있다.

---

# 증거 기반 진단 순서

## 1. 영향을 확인한다

실패한 AZ, 인스턴스, Task, Pod, 버전과 endpoint를 정상 대상과 비교한다.

## 2. 최근 변경을 확인한다

AMI, 이미지, Task Definition, Deployment, resource request와 limit, JVM 옵션과 환경 변수 변경을 확인한다.

## 3. 공통 자원을 확인한다

```bash
top
free -m
df -h
df -i
ss -s
ss -lntp
```

CPU 사용률뿐 아니라 load average, iowait와 throttling을 함께 해석한다.

가용 memory와 swap, kernel OOM 기록을 확인해 단순 heap 부족과 시스템 memory pressure를 구분한다.

disk 사용량과 inode를 함께 확인하고 로그 폭증이나 임시 파일 누적 여부를 찾는다.

```bash
ulimit -n
journalctl --since "30 min ago" --priority warning
journalctl -k --since "30 min ago"
```

file descriptor 사용량은 프로세스 한도와 실제 열린 파일 및 socket 수를 비교한다.

## 4. EC2 증거를 확인한다

```bash
aws ec2 describe-instance-status \
  --instance-ids "$INSTANCE_ID" \
  --include-all-instances
systemctl status shop-api
journalctl -u shop-api --since "30 min ago"
```

System status와 Instance status를 구분하고 애플리케이션 서비스 종료 시각을 대조한다.

## 5. ECS 증거를 확인한다

```bash
aws ecs describe-tasks \
  --cluster "$CLUSTER" \
  --tasks "$TASK_ARN" \
  --query 'tasks[].{stopCode:stopCode,stoppedReason:stoppedReason,containers:containers[].{name:name,reason:reason,exitCode:exitCode}}'
```

`stoppedReason`, 컨테이너 `reason`, `exitCode`와 배포 이벤트를 함께 확인한다.

Task가 시작되지 않으면 image pull, IAM, secret, port와 capacity 오류를 구분한다.

## 6. EKS 증거를 확인한다

```bash
kubectl describe pod "$POD" -n "$NAMESPACE"
kubectl logs "$POD" -n "$NAMESPACE" --all-containers
kubectl logs "$POD" -n "$NAMESPACE" --previous
kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp
```

`OOMKilled`, `CrashLoopBackOff`, probe 실패, scheduling 실패와 node pressure event를 확인한다.

## 7. 가설과 완화를 검증한다

OOM이면 limit, heap 상한, off-heap과 동시 요청 증가를 함께 분석한다.

disk full이면 생성 주체와 보존 정책을 확인하고 승인된 안전 절차로 용량을 확보한다.

문제 버전이면 검증된 artifact로 단계적으로 롤백하고 정상 용량을 유지한다.

---

# Spring Boot 관찰 포인트

JVM heap, non-heap, GC pause, live thread, process file descriptor와 HTTP server thread를 수집한다.

Actuator readiness와 liveness의 실패 원인을 로그와 event에 연결한다.

요청 큐와 executor rejection은 CPU가 낮아도 내부 병목이 있음을 보여줄 수 있다.

```sql
fields @timestamp, level, message, serviceVersion, instanceId
| filter level = "ERROR"
| stats count() by serviceVersion, instanceId, message
| sort count() desc
```

OutOfMemoryError 로그와 heap dump는 민감 정보와 저장 공간 영향을 고려한 승인된 절차로 보존한다.

---

# 대응과 복구

사용자 영향을 줄이기 위해 문제 대상을 트래픽에서 제외하되 남은 용량을 먼저 확인한다.

자원 증설은 안전한 완화가 될 수 있지만 leak, 무제한 큐와 잘못된 limit의 근본 원인을 별도로 수정한다.

복구 후 재시작 횟수, error rate, saturation과 배포 안정성을 충분한 관찰 구간 동안 확인한다.

---

# 재발 방지

CPU, memory, disk, inode, file descriptor와 restart에 포화도 경보를 구성한다.

컨테이너 request와 limit은 부하 시험과 실제 사용량을 기반으로 정한다.

배포에는 readiness, graceful shutdown, 단계적 전환과 자동 롤백 기준을 포함한다.

구조화 로그에 실행 단위, 버전, AZ와 종료 이유를 연결한다.

---

# 기억해야 할 내용

- EC2, ECS와 EKS에도 공통 자원 포화 원리가 적용된다.
- OOM과 재시작은 로그가 사라지기 전에 증거를 확보한다.
- ECS stopped reason과 EKS event 및 이전 로그를 확인한다.
- CPU만 보지 않고 memory, disk, file descriptor와 socket을 함께 본다.
- 롤백과 격리는 남은 처리 용량을 확인한 뒤 수행한다.

---

# 다음 Chapter

다음 장에서는 [RDS 장애](/aws-backend/part-15/05-rds-failure/)를 분석한다.
