# ALB는 NAT Gateway를 경유하는가?

일반적으로 ALB는 NAT Gateway를 경유하지 않는다.

## 구성 이미지

```text
Internet
   ↓
ALB (Public Subnet)
   ↓
ECS Fargate / EC2 (Private Subnet)

ECS에서 외부로 통신할 때만
   ↓
NAT Gateway
   ↓
Internet
```

---

# 통신 방향별 역할

## 1. ALB → ECS

```text
Internet
   ↓
ALB
   ↓
ECS
```

- inbound 통신(외부 → 내부)
- NAT Gateway 불필요
- ALB가 ECS의 private IP로 직접 전달

---

## 2. ECS → Internet

```text
ECS
  ↓
NAT Gateway
  ↓
Internet
```

- outbound 통신(내부 → 외부)
- NAT Gateway 필요
- 외부 API나 AWS 서비스 연결에 사용

### 예

- ECR image pull
- CloudWatch Logs
- Secrets Manager
- 외부 SaaS API
- npm install
- apt/yum update

---

# ALB가 NAT를 통과하지 않는 이유

ALB는 Public Subnet에 존재하며,
Internet Gateway(IGW)에 직접 연결되어 있다.

```text
Internet
  ↓
IGW
  ↓
ALB
```

그 후 ALB는 VPC 내부 통신으로 ECS에 전달한다.

```text
ALB
  ↓
ECS (private IP)
```

이 통신은 VPC 내부 통신이므로 NAT가 필요 없다.

---

# NAT Gateway의 역할

NAT Gateway는:

```text
Private Subnet → Internet
```

을 위한 용도로만 존재한다.

반대 방향:

```text
Internet → Private Subnet
```

은 불가능하다.

즉 NAT Gateway는
"외부로 나가는 전용 경로"다.

---

# 자주 하는 오해

## 잘못된 이미지

```text
Internet
 ↓
ALB
 ↓
NAT Gateway
 ↓
ECS
```

## 실제 통신

### inbound

```text
Client
 ↓
ALB
 ↓
ECS
```

### outbound

```text
ECS
 ↓
NAT Gateway
 ↓
Internet
```

ALB와 NAT Gateway는 용도가 다르다.

---

# 이 구성의 장점

- ECS에 Public IP가 필요 없음
- ECS를 Private Subnet에 배치 가능
- Internet에서 ECS로 직접 접근 불가
- 보안 향상
- ALB만 공개

AWS의 대표적인 보안 구성이다.