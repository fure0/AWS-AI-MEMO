# AWS VPC / Public Subnet / Private Subnet 정리

# 1. VPC란?

## VPC (Virtual Private Cloud)

AWS 안에서 내가 독립적으로 사용하는 가상 네트워크 공간.

비유:
- AWS 전체 = 거대한 도시
- VPC = 내 회사 부지

특징:
- IP 주소 대역 직접 설정 가능
- 네트워크 구조 직접 설계 가능
- 다른 사용자 네트워크와 분리됨

---

# 2. Subnet 이란?

VPC 내부를 여러 네트워크 구역으로 나눈 것.

비유:
- 회사 부지 안의 여러 구역

예시:
- 외부 손님용 구역
- 직원 전용 구역
- 창고 구역

---

# 3. Public Subnet

인터넷과 직접 연결 가능한 서브넷.

비유:
- 대로변 건물
- 외부 사람이 바로 출입 가능

특징:
- 인터넷 접근 가능
- Public IP 사용 가능
- Internet Gateway 연결

주 용도:
- 공개 웹 서버
- Load Balancer
- NAT Gateway
- Bastion Host

---

# 4. Private Subnet

인터넷에서 직접 접근할 수 없는 내부 전용 서브넷.

비유:
- 회사 내부 사무실
- 외부인은 직접 못 들어감

특징:
- 외부 인터넷 직접 접근 불가
- 보안성이 높음
- 내부 시스템 보호

주 용도:
- DB 서버
- 내부 API 서버
- ECS/EKS 컨테이너
- Redis 캐시 서버

---

# 5. Public Subnet 에 배치되는 대표 서비스

## 5-1. Load Balancer

예시:
- ALB (Application Load Balancer)
- NLB (Network Load Balancer)

역할:
- 인터넷 요청을 받아 내부 서버로 분산

비유:
- 건물 안내 데스크

---

## 5-2. Bastion Host

역할:
- 관리자가 SSH/RDP 접속하는 중계 서버

비유:
- 경비실

특징:
- 외부에서 관리 접속 가능

---

## 5-3. Public Web Server

예시:
- EC2 기반 웹 서버

역할:
- 사용자에게 직접 웹페이지 제공

최근 구조:
- ALB만 Public
- 실제 App 서버는 Private 에 두는 경우 많음

---

## 5-4. NAT Gateway

역할:
- Private Subnet 서버가 인터넷으로 나갈 수 있게 함

예시:
- OS 업데이트
- 패키지 다운로드

중요:
- NAT Gateway 는 반드시 Public Subnet 에 위치

---

# 6. Private Subnet 에 배치되는 대표 서비스

## 6-1. Database

예시:
- RDS
- Aurora

역할:
- 고객 데이터 저장

왜 Private?
- DB를 인터넷에 공개하면 위험

---

## 6-2. Application Server

예시:
- Spring Boot
- Node.js
- Django 서버

역할:
- 실제 비즈니스 로직 처리

흐름:
사용자 → ALB → App Server

---

## 6-3. ECS / EKS Worker Node

예시:
- ECS
- Kubernetes Node

역할:
- 컨테이너 실행

특징:
- 외부 직접 노출 안 함
- ALB 통해 접근

---

## 6-4. Cache Server

예시:
- Redis
- ElastiCache
- Memcached

역할:
- 빠른 응답용 메모리 캐시

왜 Private?
- 내부 서버만 사용하면 됨

---

## 6-5. Internal API Server

역할:
- 내부 시스템 간 통신

예시:
- 마이크로서비스 API
- 사내 전용 API

---

# 7. 실제 많이 사용하는 AWS 구조

```text
인터넷 사용자
       ↓
[Public Subnet]
ALB (공개 입구)
       ↓
[Private Subnet]
App Server / ECS
       ↓
[Private Subnet]
RDS / Redis
```

---

# 8. 왜 Public / Private 을 나누는가?

핵심 목적:
- 보안
- 네트워크 분리
- 최소 공개

잘못된 구조:

```text
인터넷 → DB 직접 접근 가능
```

안전한 구조:

```text
인터넷 → ALB
ALB → App Server
App Server → DB
```

즉:
- 외부 공개 최소화
- 내부 시스템 보호

---

# 9. 핵심 요약

| 구성 요소 | 비유 | 역할 |
|---|---|---|
| VPC | 회사 부지 | 독립 네트워크 |
| Subnet | 부지 내 구역 | 네트워크 분리 |
| Public Subnet | 대로변 건물 | 외부 공개 서버 |
| Private Subnet | 내부 사무실 | 내부 핵심 시스템 |

---

# 10. 한 줄 요약

> 사용자와 직접 만나는 시스템만 Public,
핵심 시스템(DB/App)은 Private 에 둔다.