# AWS Route 53 정리

## Route 53이란?

Route 53은 AWS의 **관리형 DNS(Domain Name System) 서비스**이다.

쉽게 말하면,

> **도메인 이름을 적절한 목적지(IP, 서버, AWS 리소스 등)로 연결해 주는 서비스**

라고 이해하면 된다.

---

# Route 53의 주요 역할

## 1. 도메인 등록 (Domain Registration)

Route 53에서는 도메인을 직접 구매(등록)할 수 있다.

예시:

- example.com
- mycompany.jp

하지만 많은 기업은 다른 등록기관(가비아, GoDaddy 등)에서 도메인을 구매한 후 DNS만 Route 53을 사용한다.

---

## 2. DNS 관리 (가장 중요한 역할)

사용자가 브라우저에 다음을 입력하면

```text
www.example.com
```

Route 53이 실제 목적지를 찾아준다.

예시:

```text
www.example.com
↓
54.123.45.67
```

또는

```text
api.example.com
↓
api-server.example.net
```

이를 위해 사용하는 레코드 종류:

- A 레코드
- AAAA 레코드
- CNAME 레코드
- MX 레코드
- TXT 레코드

---

## 3. 트래픽 라우팅

Route 53은 단순히 IP를 알려주는 것뿐 아니라 트래픽을 분산시킬 수 있다.

예시:

```text
한국 사용자 → 서울 서버
일본 사용자 → 도쿄 서버
미국 사용자 → 버지니아 서버
```

대표 기능:

- Latency Routing
- Geolocation Routing
- Weighted Routing
- Failover Routing

---

# Route 53이 꼭 필요한가?

## EC2를 직접 연결할 수도 있다

예를 들어 가비아에서 구매한 도메인이 있다면

```text
example.com
↓
54.123.45.67 (EC2)
```

처럼 직접 연결할 수 있다.

즉,

> "EC2에 연결하려면 Route 53이 반드시 필요하다"

는 잘못된 생각이다.

---

# 그렇다면 왜 Route 53을 사용할까?

## 1. AWS 서비스와 쉽게 연동된다

많은 AWS 서비스는 IP 대신 DNS 이름을 가진다.

예시:

```text
my-alb-123.ap-northeast-1.elb.amazonaws.com
```

이 경우 Route 53의 Alias Record를 사용하면

```text
www.example.com
↓
ALB
```

처럼 쉽게 연결할 수 있다.

연동 가능한 대표 서비스:

- ALB (Application Load Balancer)
- CloudFront
- API Gateway
- S3 Static Website Hosting

---

## 2. IP 변경에 강하다

EC2를 직접 연결하면

```text
example.com
↓
54.123.45.67
```

구조가 된다.

하지만 EC2가 교체되거나 IP가 변경되면 DNS 설정도 수정해야 한다.

실무에서는 보통 다음 구조를 사용한다.

```text
example.com
↓
Route 53
↓
ALB
↓
EC2 여러 대
```

이렇게 하면 EC2가 바뀌어도 도메인 설정은 그대로 유지된다.

---

## 3. 고급 트래픽 제어가 가능하다

예시:

```text
한국 사용자 → 서울 ALB
일본 사용자 → 도쿄 ALB
```

또는

```text
서버 A → 80%
서버 B → 20%
```

와 같은 트래픽 분산이 가능하다.

---

# AWS 아키텍처에서의 위치

일반적으로 다음과 같은 구조를 사용한다.

```text
도메인 등록기관
(가비아, GoDaddy 등)
        ↓
Route 53
        ↓
ALB
        ↓
ECS / EC2
```

여기서 Route 53은

> "도메인 이름과 AWS 리소스를 연결해 주는 DNS 관제탑"

역할을 한다.

---

# 핵심 정리

## Route 53의 본질

- AWS의 관리형 DNS 서비스
- 도메인 이름을 실제 목적지에 연결
- AWS 리소스와 강력하게 통합
- 트래픽 분산 및 장애 조치 기능 제공

## 한 줄 요약

> Route 53은 "도메인의 목적지를 관리하는 AWS의 DNS 및 트래픽 라우팅 서비스"이다.

## 시험(SAA) 관점 요약

암기 포인트:

```text
Route 53 = AWS Managed DNS Service
```

역할:

```text
도메인 이름
↓
Route 53
↓
IP / ALB / CloudFront / API Gateway / S3
```

즉,

"도메인을 발급하는 서비스"보다는

"도메인을 적절한 AWS 리소스나 IP로 연결하는 서비스"

라고 이해하는 것이 가장 정확하다.