# AWS ECS / EKS / EC2 / Fargate 구조 정리

## 핵심 개념

AWS 컨테이너 구조는 크게 두 가지로 나뉜다.

- 컨트롤 플레인(Control Plane)
- 데이터 플레인(Data Plane)

---

# 1. 컨트롤 플레인 vs 데이터 플레인

## 컨트롤 플레인 (Control Plane)

“무엇을 어떻게 실행할지 관리하는 두뇌”

역할:
- 컨테이너 배치
- 스케줄링
- 상태 관리
- 오토 스케일링
- 장애 복구

대표 서비스:
- ECS
- EKS

---

## 데이터 플레인 (Data Plane)

“실제로 컨테이너가 실행되는 환경”

역할:
- CPU 제공
- 메모리 제공
- 네트워크 제공
- 실제 컨테이너 실행

대표 서비스:
- EC2
- Fargate

---

# 2. 전체 구조

```text
[ECS / EKS]  ← 컨트롤 플레인 (관리/오케스트레이션)
      ↓
[EC2  or  Fargate] ← 데이터 플레인 (실행 환경)
      ↓
[Container 실행]
```

---

# 3. ECS와 EKS의 역할

## ECS

AWS 전용 컨테이너 오케스트레이터

역할:
- 컨테이너 배치
- Task 실행
- Service 유지
- Auto Scaling

특징:
- 단순함
- AWS 친화적
- Kubernetes보다 쉬움

---

## EKS

AWS 관리형 Kubernetes

역할:
- Kubernetes 제공
- Pod 스케줄링
- 클러스터 관리

특징:
- 표준 Kubernetes
- 복잡하지만 강력함
- 멀티 클라우드에 유리

---

# 4. EC2와 Fargate의 차이

## EC2

가상 서버(VM)를 직접 관리

특징:
- OS 직접 관리
- 인스턴스 타입 선택
- 높은 자유도
- 운영 부담 큼

핵심:
> "서버를 내가 운영한다"

---

## Fargate

서버 없는(Serverless) 컨테이너 실행 환경

특징:
- 서버 관리 불필요
- 컨테이너만 실행
- AWS가 인프라 관리

핵심:
> "서버 없이 컨테이너만 실행한다"

---

# 5. 중요한 이해 포인트

## EC2 vs Fargate는 단순히 “수동 vs 자동”이 아님

잘못된 이해:
- EC2 = 수동
- Fargate = 자동화된 EC2

정확한 이해:
- EC2 = 서버 기반 데이터 플레인
- Fargate = 서버리스 데이터 플레인

즉:

```text
EC2 → 서버가 보임
Fargate → 서버가 숨겨짐
```

---

# 6. ECS가 EC2를 제어한다고 볼 수 있을까?

정확히는:

❌ ECS가 EC2 자체를 직접 제어한다  
✅ ECS가 EC2 위에서 컨테이너를 오케스트레이션한다

---

## 실제 구조

```text
ECS Control Plane
        ↓
ECS Agent (EC2 내부)
        ↓
Docker / containerd
        ↓
Container 실행
```

즉:
- ECS는 컨테이너 실행을 지시
- EC2는 리소스를 제공

---

# 7. 정리

| 구분 | 역할 |
|---|---|
| ECS | AWS 컨테이너 오케스트레이터 |
| EKS | Kubernetes 오케스트레이터 |
| EC2 | 서버 기반 실행 환경 |
| Fargate | 서버리스 실행 환경 |

---

# 8. 핵심 한 줄 요약

```text
EC2 / Fargate = 어디서 실행할까?
ECS / EKS = 어떻게 관리할까?
```

---

# 9. 비유

- EC2 → 직접 운영하는 주방
- Fargate → 완성된 주방 대여
- ECS/EKS → 요리 작업을 지시하는 매니저

---

# 10. 최종 핵심

```text
ECS/EKS = 컨트롤 플레인
EC2/Fargate = 데이터 플레인
```