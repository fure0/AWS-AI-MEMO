# AWS API Gateway를 굳이 사용하는 이유

웹서버 백엔드에서 직접 API Endpoint를 만들 수 있는데도  
왜 AWS API Gateway를 사용하는가?

## 결론

API Gateway는 단순히 API Endpoint를 만드는 서비스가 아니라,

> API를 인프라 레벨에서 관리하기 위한 서비스

이다.

단순한 웹 애플리케이션 Backend라면 ECS, EC2 등에서 직접 API를 만들어도 된다.  
하지만 인증, 보안, 트래픽 제어, API 관리가 필요하면 API Gateway가 유용하다.

---

# 1. API Gateway를 사용하는 이유

## 1) 인증 / 보안 처리를 분리

API Gateway에서 처리 가능한 것:

- JWT 검증
- OAuth 인증
- API Key 관리
- IAM 인증
- Rate Limit 적용


구조:

```text
Client
  |
  v
API Gateway
  |
  v
Backend Server
```

장점:

- 인증 로직을 Backend 코드에서 제거 가능
- 보안 정책 중앙 관리 가능
- 서비스별 중복 구현 방지


---

# 2. 트래픽 제어 및 보호

API Gateway는 요청량 제어 기능 제공:

- Throttling
- Rate Limit
- Burst Control


예:

```text
Client 요청 10000개
        |
        v
API Gateway
        |
        +-- 허용 요청 → Backend
        |
        +-- 초과 요청 → 차단
```


장점:

- Backend 장애 방지
- 갑작스러운 트래픽 증가 대응
- API 보호


---

# 3. Lambda와 조합 (Serverless)

API Gateway는 Lambda와 많이 사용된다.


구조:

```text
Client
  |
  v
API Gateway
  |
  v
Lambda
```


장점:

- EC2/ECS 서버 불필요
- 요청 발생 시만 실행
- 서버 운영 부담 감소


---

# 4. API 관리 기능

API Gateway는 API 운영 기능 제공


## API Version 관리

예:

```text
/api/v1/users
/api/v2/users
```


## Stage 관리

예:

```text
dev
staging
prod
```


기타 기능:

- Request / Response 변환
- CloudWatch 로그 연동
- Monitoring
- 배포 관리


즉 API를 하나의 제품처럼 관리 가능하다.


---

# 5. 여러 Backend 통합

API Gateway 뒤에 여러 Backend 연결 가능


구조:

```text
              API Gateway
                   |
        ----------------------
        |          |         |
      Lambda      ECS       EC2
```


외부에서는 하나의 API Endpoint처럼 보이고,
내부는 여러 서비스로 분리 가능하다.


---

# API Gateway를 굳이 사용하지 않아도 되는 경우

다음 상황에서는 필요성이 낮다.

- 단순 CRUD API
- 내부 서비스 API
- 트래픽이 크지 않음
- 인증이 단순함
- 이미 ALB + ECS 구조 사용


일반적인 구조:

```text
Client
  |
  v
ALB
  |
  v
ECS Backend
```


이 경우 ALB만으로 충분한 경우가 많다.


---

# API Gateway 단점

## 1) 비용 증가

추가 관리 계층이 생기므로 비용 발생


구조:

```text
Client
 |
API Gateway
 |
Backend
```


---

## 2) Latency 증가

요청 경로가 한 단계 증가한다.


```text
Client
 |
API Gateway
 |
Backend
```


직접 Backend 호출보다 약간 느려질 수 있다.


---

# ALB vs API Gateway

## ALB

목적:

> Backend 서버로 요청을 전달하는 역할


구조:

```text
Client
 |
ALB
 |
ECS / EC2
```


주요 기능:

- Load Balancing
- Health Check
- SSL 처리
- Target 관리


---

## API Gateway

목적:

> API 자체를 관리하는 역할


구조:

```text
Client
 |
API Gateway
 |
Lambda / ECS / EC2
```


주요 기능:

- 인증
- API Key
- Rate Limit
- API Version 관리
- Request / Response 제어


---

# 실무 선택 기준

## API Gateway 추천

- 외부 공개 API
- B2B API
- 파트너 연동 API
- 인증 정책이 복잡한 경우
- Serverless 구조
- API 정책 관리 필요


## ALB + ECS 추천

- 일반 웹 서비스 Backend
- 내부 API
- CRUD 중심 서비스
- 비용 중요
- 컨테이너 기반 서비스


---

# 한 줄 요약

API Gateway:

> API를 코드가 아니라 인프라 레벨에서 관리하기 위한 서비스


ALB:

> Backend 서버로 요청을 전달하기 위한 Load Balancer


일반적인 웹 서비스:

```text
Client
 |
ALB
 |
ECS Backend
```


구조가 충분한 경우가 많다.


외부 API 관리 또는 Serverless 구조에서는:

```text
Client
 |
API Gateway
 |
Lambda / Backend
```


구조를 많이 사용한다.