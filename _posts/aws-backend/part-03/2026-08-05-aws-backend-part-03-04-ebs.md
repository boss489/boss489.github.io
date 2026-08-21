---
title: "Chapter 04. EBS"
permalink: /aws-backend/part-03/04-ebs/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. EBS
## EC2에 붙는 블록 스토리지

> **학습 목표**
>
> - EBS의 역할을 설명할 수 있다.
> - 루트 볼륨과 데이터 볼륨을 구분할 수 있다.
> - Snapshot과 볼륨 타입의 의미를 이해한다.
> - 파일 시스템 생성과 마운트 흐름을 설명할 수 있다.

---

# 왜 EBS가 필요한가

Spring Boot 서버가 재부팅되거나 EC2가 중지될 때 운영체제와 설정 파일까지 사라진다면 매번 서버를 다시 구성해야 한다.

로그나 배치 임시 데이터처럼 파일 시스템에 기록할 공간도 필요하다.

EBS는 EC2의 가상 하드디스크로 동작하며 인스턴스와 분리된 생명주기를 가질 수 있다.

다만 서버 디스크가 모든 데이터 문제의 해답은 아니므로 공유 파일은 S3나 EFS, 업무 데이터는 RDS 같은 목적별 저장소를 우선 검토해야 한다.

---

# EBS란?

EBS(Elastic Block Store)는 EC2에 연결하는 블록 스토리지다.

EC2의 디스크처럼 사용한다.

루트 볼륨은 OS가 설치되는 디스크이고, 데이터 볼륨은 추가 저장 공간으로 사용할 수 있다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# EBS의 구조

EBS 볼륨은 특정 Availability Zone에 생성되고 같은 AZ의 EC2에 연결한다.

```
Availability Zone A
├── EC2
│   ├── Root EBS
│   └── Data EBS
└── EBS Snapshot ── AWS 관리 스토리지
                         │
                         └── 새 AZ에 Volume 복원
```

EC2에 연결된 볼륨은 Linux에서 블록 장치로 보이며, 파일 시스템을 만든 뒤 디렉터리에 마운트해야 파일을 저장할 수 있다.

볼륨을 분리해 다른 인스턴스에 연결할 수 있지만 애플리케이션 쓰기를 중단하고 파일 시스템 일관성을 확보해야 한다.

---

# EBS와 EC2 생명주기

EC2가 중지되어도 EBS 볼륨은 남을 수 있다.

반대로 인스턴스를 종료할 때 루트 볼륨을 함께 삭제하도록 설정할 수도 있다.

운영에서는 중요한 데이터가 EC2 로컬에만 남지 않도록 주의해야 한다.

| 구분 | 루트 볼륨 | 데이터 볼륨 |
|---|---|---|
| 주 용도 | OS와 기본 파일 | 애플리케이션 데이터·로그 |
| 생성 | 인스턴스 시작 시 생성 | 별도로 생성 가능 |
| 종료 시 삭제 | 기본 설정을 반드시 확인 | 보통 별도 생명주기 |
| 교체 | AMI로 재생성 가능 | Snapshot 복구 고려 |

---

# Snapshot

Snapshot은 EBS 볼륨의 백업이다.

Snapshot을 이용해 다음 작업을 할 수 있다.

- 볼륨 복구
- 다른 AZ 또는 Region으로 복사
- AMI 생성
- 장애 대응

Snapshot은 변경된 블록을 기반으로 증분 저장되지만 각 Snapshot은 복구 시 완전한 볼륨을 만들 수 있도록 관리된다.

Snapshot 생성 완료와 애플리케이션 데이터 정합성은 같은 의미가 아니므로 데이터베이스나 쓰기 작업은 별도 정합성 절차가 필요하다.

---

# 볼륨 타입

대표적으로 `gp3`를 많이 사용한다.

성능이 더 필요하면 IOPS나 처리량을 조정하거나 다른 타입을 선택한다.

처음부터 고성능 타입을 쓰기보다 실제 지표를 보고 조정하는 것이 낫다.

---

# 볼륨 타입 비교

| 계열 | 특성 | 대표 용도 |
|---|---|---|
| 범용 SSD | 비용과 성능의 균형 | 부트 볼륨, 일반 애플리케이션 |
| Provisioned IOPS SSD | 일관된 높은 IOPS | 지연에 민감한 데이터 워크로드 |
| 처리량 최적화 HDD | 큰 순차 처리에 유리 | 로그·빅데이터 처리 |
| Cold HDD | 낮은 비용, 낮은 접근 빈도 | 자주 읽지 않는 순차 데이터 |

IOPS는 초당 입출력 작업 수이고 처리량은 초당 전송 데이터 양이므로 작은 랜덤 I/O와 큰 순차 I/O의 병목이 다르다.

---

# Linux에서 연결하기

새 데이터 볼륨은 장치 확인, 파일 시스템 생성, 마운트 순서로 사용한다.

```bash
lsblk
sudo mkfs.xfs /dev/nvme1n1
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
df -h
```

`mkfs`는 기존 데이터를 지우므로 새 볼륨인지 반드시 확인해야 한다.

재부팅 후에도 마운트하려면 장치 이름 대신 UUID를 확인해 `/etc/fstab`에 안전하게 등록한다.

```bash
sudo blkid /dev/nvme1n1
sudo mount -a
```

잘못된 `fstab` 설정은 부팅 실패를 일으킬 수 있으므로 `mount -a`로 검증한다.

---

# AWS 콘솔과 CLI에서는

볼륨의 AZ와 연결 상태를 조회할 수 있다.

```bash
aws ec2 describe-volumes \
  --volume-ids vol-0123456789abcdef0 \
  --query "Volumes[0].{State:State,Az:AvailabilityZone,Type:VolumeType,Size:Size}"
```

Snapshot에는 용도와 원본 볼륨을 식별할 태그를 남긴다.

```bash
aws ec2 create-snapshot \
  --volume-id vol-0123456789abcdef0 \
  --description "backend-data-before-migration"
```

---

# 실무에서는 어떻게 사용할까

애플리케이션 로그는 EBS에만 오래 보관하지 않고 CloudWatch Logs 같은 중앙 저장소로 전송한다.

인스턴스 교체가 잦은 환경에서는 로컬 파일에 상태를 남기지 않아야 새 인스턴스가 즉시 요청을 처리할 수 있다.

Snapshot은 백업 정책에 따라 자동 생성하고 복구 테스트를 수행해야 실제 장애 때 사용할 수 있다.

볼륨 암호화는 생성 시 적용하고 KMS Key 권한이 복구 계정과 리전에서도 유효한지 확인한다.

---

# 장애 사례와 주의할 점

| 증상 | 원인 후보 | 대응 |
|---|---|---|
| `No space left on device` | 용량 또는 inode 고갈 | `df -h`, `df -i` 확인 |
| 볼륨 연결 불가 | EC2와 AZ 불일치 | 같은 AZ에 생성 또는 Snapshot 복원 |
| 재부팅 후 `/data` 없음 | `fstab` 누락·오류 | UUID와 마운트 설정 확인 |
| 디스크 지연 증가 | IOPS·처리량 한계 | EBS와 OS 지표 분석 |
| Snapshot은 있지만 복구 실패 | 복구 절차 미검증 | 정기 복원 테스트 |

볼륨 크기를 늘린 뒤에는 파티션과 파일 시스템도 확장해야 운영체제에서 증가분을 사용할 수 있다.

---

# 비용과 성능 고려사항

EBS 비용은 볼륨 타입과 할당 용량, 설정한 성능, Snapshot 저장량에서 발생한다.

인스턴스를 종료해도 남아 있는 미연결 볼륨은 계속 비용이 발생하므로 태그와 사용 관계를 확인해 정리한다.

최고 성능 옵션부터 선택하지 말고 지연 시간, Queue Length, IOPS와 처리량을 관찰해 조정한다.

---

# 기억해야 할 내용

- EBS는 EC2에 붙는 블록 스토리지다.
- 루트 볼륨은 OS 디스크다.
- Snapshot은 백업과 복구에 사용한다.
- 중요한 영속 데이터는 RDS, S3 같은 목적에 맞는 저장소를 우선 고려한다.
- EBS 볼륨은 AZ 리소스이며 같은 AZ의 EC2에 연결한다.
- 새 볼륨은 파일 시스템 생성과 마운트가 필요하다.
- Snapshot 보유만으로는 충분하지 않으며 복구를 검증해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 [ENI](/aws-backend/part-03/05-eni/)를 학습한다.

EC2가 VPC의 Private IP와 Security Group을 통해 네트워크에 연결되는 구조를 알아본다.


