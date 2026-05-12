# AWS API Gateway 역할 정리

## API Gateway란?

AWS의 API Gateway는  
클라이언트와 백엔드 서비스 사이의 **출입구(Gateway)** 역할을 하는 서비스이다.

예를 들어:

- 모바일 앱
- 웹 프론트엔드
- 외부 시스템

등에서 요청을 보내면 API Gateway가 이를 받아:

- Lambda 실행
- ECS/EKS 서비스 호출
- EC2 서버 전달
- 외부 HTTP API 연결

등으로 중계한다.

---

# 기본 구조

```text
클라이언트
    ↓
API Gateway
    ↓
백엔드 서비스들
```

---

# 핵심 역할

## 1. API의 진입점 제공

사용자는 하나의 URL만 사용하면 된다.

예시:

```text
https://api.example.com/users
```

하지만 내부에서는 여러 서비스로 연결 가능:

- Lambda
- ECS
- EC2
- 마이크로서비스

등.

즉:

```text
클라이언트 → API Gateway → 내부 서비스들
```

형태로 동작한다.

---

## 2. 요청 라우팅

경로나 HTTP 메서드에 따라 다른 서비스로 전달한다.

예시:

| 요청 | 전달 대상 |
|---|---|
| GET /users | Lambda A |
| POST /users | ECS 서비스 |
| GET /products | EC2 서버 |

---

## 3. 인증 및 권한 처리

백엔드에서 직접 인증 로직을 구현하지 않아도 된다.

지원 예시:

- JWT
- Cognito
- IAM 인증
- Lambda Authorizer

즉 API Gateway가 먼저:

```text
"이 사용자가 접근 가능한가?"
```

검사 후 요청을 전달한다.

---

## 4. 트래픽 제어

과도한 요청으로부터 서버를 보호한다.

기능 예시:

- Rate Limit
- Throttling
- Quota

---

## 5. 요청/응답 변환

클라이언트와 백엔드의 데이터 형식이 달라도 중간에서 변환 가능하다.

예시:

```text
클라이언트 요청
↓
API Gateway 변환
↓
Lambda 이벤트 형식
```

---

## 6. 모니터링 및 로그

CloudWatch와 연동 가능:

- API 호출량
- 에러율
- 응답 지연시간
- 로그 기록

등을 확인할 수 있다.

---

# 대표적인 사용 예시

## 1. Lambda와 조합 (서버리스)

```text
브라우저
↓
API Gateway
↓
Lambda
↓
DynamoDB
```

서버리스 아키텍처에서 매우 자주 사용된다.

---

## 2. 마이크로서비스 앞단

```text
사용자
↓
API Gateway
↓
ECS/EKS 서비스들
```

여러 서비스의 진입점을 하나로 통합 가능하다.

---

# API Gateway와 ALB 차이

| 항목 | API Gateway | ALB |
|---|---|---|
| 목적 | API 관리 | 트래픽 분산 |
| 인증/권한 | 강력 | 상대적으로 단순 |
| Lambda 연동 | 매우 강력 | 가능 |
| API 기능 | 특화 | 일반 로드밸런싱 |
| 비용 방식 | 요청 기반 | 시간 + LCU 기반 |

---

# 비유로 이해하기

| 서비스 | 비유 |
|---|---|
| API Gateway | 건물 안내 데스크 + 보안 게이트 |
| ALB | 차량 분산 교차로 |
| Lambda | 실제 업무 처리 직원 |
| ECS/EKS | 컨테이너 작업실 |

---

# 한 줄 요약

API Gateway는:

> 외부 요청을 받아 인증하고,  
> 적절한 백엔드 서비스로 전달하며,  
> API를 통합 관리하는 AWS의 API 출입구 서비스이다.