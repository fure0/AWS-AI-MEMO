# AWS 멀티 AZ 네트워크 구성 개념

일반적인 AWS 멀티 AZ 구성에서는:

- **VPC는 보통 1개**
- **그 안에 AZ별로 Public / Private Subnet을 여러 개 구성**

하는 패턴이 가장 흔하다.

## 기본 구조

```
                VPC (10.0.0.0/16)
                         |
        ---------------------------------
        |               |               |
       AZ-a            AZ-b            AZ-c
        |               |               |
   -------------   -------------   -------------
   |           |   |           |   |           |
Public       Private Public    Private Public  Private
Subnet       Subnet Subnet    Subnet Subnet   Subnet
   |           |       |          |       |       |
 ALB        ECS     ALB        ECS     NAT     DB
```

---

# 왜 VPC는 보통 1개인가?

VPC는 AWS 네트워크의 큰 경계이다.

```
VPC
 ├─ Routing Table
 ├─ Security Group
 ├─ NACL
 ├─ Internet Gateway
 ├─ VPC Endpoint
 └─ Subnet
```

애플리케이션 하나를 운영할 때 VPC를 여러 개로 나누면 관리 복잡도가 증가한다.

그래서 보통:

```
Production VPC

 ├─ Public Subnet
 ├─ Private Application Subnet
 ├─ Private Database Subnet
 └─ 기타 Subnet
```

형태로 구성한다.

---

# 왜 AZ마다 Public / Private Subnet을 만드는가?

멀티 AZ의 목적:

> 하나의 AZ 장애가 발생해도 서비스가 유지되도록 하기 위해서

예:

```
VPC

AZ-a
 ├ Public Subnet
 │    └ ALB
 │
 └ Private Subnet
      └ ECS


AZ-b
 ├ Public Subnet
 │    └ ALB
 │
 └ Private Subnet
      └ ECS
```

AZ-a 장애 발생:

```
AZ-b

ALB
 |
ECS
```

서비스 유지 가능.

---

# 일반적인 3-Tier 구조

```
VPC

Public Subnet
    |
    └ ALB


Private Application Subnet
    |
    └ ECS / EC2


Private Database Subnet
    |
    └ RDS
```

멀티 AZ:

```
AZ-a                    AZ-b

Public                  Public
 ALB                     ALB

Private                 Private
 ECS                     ECS

Database                Database
 RDS Primary             RDS Standby
```

---

# VPC를 여러 개 사용하는 경우

대규모 환경에서는 환경별로 VPC를 분리한다.

```
AWS Account

├─ Production VPC
│
├─ Staging VPC
│
├─ Development VPC
│
└─ Shared Services VPC
```

이유:

- 환경 격리
- 보안 경계 분리
- 비용 관리
- 팀별 관리

---

# 일반적인 구성 정리

| 구성요소 | 일반적인 패턴 |
|---|---|
| VPC | 보통 1개 |
| AZ | 2~3개 |
| Public Subnet | AZ마다 1개 |
| Private Subnet | AZ마다 1개 이상 |
| ALB | Public Subnet |
| ECS / EC2 | Private Subnet |
| RDS | Private Subnet |
| NAT Gateway | 보통 AZ마다 1개 |

---

결론:

**AWS 멀티 AZ 구성의 일반적인 형태는  
"하나의 VPC 안에 여러 AZ를 사용하고, 각 AZ마다 Public / Private Subnet을 배치하는 구조"이다.**