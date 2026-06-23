# NAT Gateway는 Multi-AZ에서 2대가 필요하지만, ALB는 1대로 충분한 이유

## 결론

| 항목 | NAT Gateway | ALB |
|---|---|---|
| AZ마다 생성? | 기본적으로 Yes | AWS가 내부에서 자동으로 Multi-AZ화 |
| 대수 이미지 | 각 AZ에 1개 | 1개 생성으로 OK |
| 주요 목적 | Outbound 통신 | Inbound 부하 분산 |
| AZ 장애 시 | 같은 AZ의 NAT 필요 | AWS 측에서 이미 이중화됨 |
| 권장 구성 | 각 AZ에 배치 | 1 ALB + 여러 AZ subnet |

---

## NAT Gateway가 AZ마다 필요한 이유

### 1. NAT Gateway는 AZ에 속한다

NAT Gateway는:

- 특정 Public Subnet
- 특정 AZ

에 배치된다.

예:

```text
NAT-1a → ap-northeast-1a
NAT-1c → ap-northeast-1c
```

---

### 2. Private Subnet은 같은 AZ의 NAT를 사용한다

일반적인 구성:

| Private Subnet | 사용하는 NAT |
|---|---|
| private-1a | NAT-1a |
| private-1c | NAT-1c |

이유:

- AZ 장애 내성
- Cross-AZ 통신 비용 회피
- 지연 시간 감소

---

### 3. NAT를 1대만 두면 문제가 있다

예:

```text
private-1a --> NAT-1a
private-1c --> NAT-1a
```

이 상태에서:

- AZ 1a 장애
- NAT-1a 장애

가 발생하면,

```text
private-1c도 Internet으로 나갈 수 없게 된다
```

영향 예:

- ECS가 ECR pull을 할 수 없음
- 외부 API 호출 실패
- patch download 실패
- CloudWatch Logs 전송 실패

---

## ALB는 왜 1대로 충분한가

### ALB는 내부적으로 Multi-AZ 구성이다

ALB 생성 시:

```text
subnet-1a
subnet-1c
```

를 지정한다.

그러면 AWS 내부에서:

```text
ALB Node (1a)
ALB Node (1c)
```

가 자동으로 배치된다.

즉:

```text
논리적으로는 ALB 1대
실제로는 각 AZ에 분산 배치
```

되어 있는 구조다.

---

## NAT와 ALB의 본질적인 차이

### NAT Gateway

역할:

```text
Private → Internet
```

특징:

- Outbound 전용
- 경로 장치
- AZ 로컬 성격이 강함

---

### ALB

역할:

```text
Internet → Backend
```

특징:

- Inbound 부하 분산
- AWS Managed Service
- 이중화 포함

---

## ECS/Fargate의 일반적인 구성

```text
Internet
   ↓
ALB (Multi-AZ)
   ↓
ECS Tasks (Private Subnet)
   ↓
NAT Gateway (AZ마다)
   ↓
Internet outbound
```

---

## ALB → ECS는 NAT를 경유하지 않는다

ALB에서 ECS로 가는 통신은:

```text
VPC 내부 통신
```

이므로:

- NAT 불필요
- Internet 불필요

NAT가 필요한 것은:

```text
ECS → 외부 통신
```

일 때뿐이다.

예:

- ECR pull
- OpenAI API
- Stripe API
- CloudWatch Logs
- SSM
- npm install

---

## 실무에서의 구분

### 개발 환경

비용 절감을 위해:

```text
NAT 1대
```

로 구성하는 경우도 많다.

다만:

- AZ 장애 내성 저하
- Cross-AZ 통신 비용 발생

을 허용한다는 전제가 필요하다.

---

### 운영 환경

일반적으로는:

```text
각 AZ에 NAT Gateway
```

를 배치한다.

AWS 공식 권장 구성도 이 방식이다.

---

## 한 줄 요약

```text
ALB:
  "하나의 서비스 안에 Multi-AZ 이중화가 포함되어 있다"

NAT Gateway:
  "AZ 단위의 구성 요소이므로 직접 이중화해야 한다"
```