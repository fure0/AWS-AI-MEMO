# API Gateway vs EventBridge

## 한 줄 요약

- API Gateway  
  → 외부 요청을 받는 API 진입점

- EventBridge  
  → 내부 이벤트를 여러 서비스에 전달하는 이벤트 허브

---

# 비유

## API Gateway = 건물의 접수 데스크

```text
사용자 → API Gateway → Lambda/ECS/EC2
```

- 사용자의 요청을 받음
- 적절한 서비스로 전달
- 응답을 다시 사용자에게 반환

예:
- 로그인
- 회원가입
- 상품 조회

---

## EventBridge = 사내 방송 시스템

```text
서비스 → EventBridge → 여러 서비스
```

- 이벤트 발생 사실을 전달
- 여러 시스템이 동시에 반응 가능

예:
- 주문 생성
- 파일 업로드 완료
- EC2 종료

---

# 핵심 차이

| 항목 | API Gateway | EventBridge |
|---|---|---|
| 목적 | API 제공 | 이벤트 전달 |
| 통신 방식 | Request/Response | Event 기반 |
| 호출 주체 | 사용자/클라이언트 | 서비스/애플리케이션 |
| 처리 방식 | 동기 처리 성향 | 비동기 처리 성향 |
| 특징 | 즉시 응답 중요 | 여러 시스템으로 전파 가능 |

---

# 관점 차이

## API Gateway

```text
클라이언트 요청 1개
        ↓
API Gateway
        ↓
백엔드 처리
```

느낌:
> "누군가 요청했다"

- 외부 요청 중심
- 보통 요청 1회 기준 처리
- 호출한 쪽이 응답을 기다림

---

## EventBridge

```text
주문 생성 이벤트
        ↓
EventBridge
   ├─ 이메일 발송
   ├─ 재고 감소
   ├─ 통계 기록
   └─ 배송 처리
```

느낌:
> "무언가 발생했다"

- 이벤트 중심
- 하나의 이벤트를 여러 시스템이 소비 가능
- 느슨한 결합(Loose Coupling)

---

# 함께 사용하는 구조

```text
사용자
   ↓
API Gateway
   ↓
Lambda/ECS
   ↓
EventBridge
   ↓
여러 후속 시스템
```

예시 흐름:

1. 사용자가 주문 요청
2. API Gateway 가 요청 수신
3. Lambda 가 주문 생성
4. EventBridge 에 `OrderCreated` 이벤트 발행
5. 여러 서비스가 각각 처리

- 이메일 발송
- 재고 감소
- 분석 시스템 기록
- 배송 처리

---

# ECS/EKS 와 연결 예시

## API Gateway

```text
사용자 → API Gateway → ECS 서비스
```

- 외부 API 진입점 역할

---

## EventBridge

```text
ECS 작업 완료 → EventBridge → 후속 작업 실행
```

- 서비스 간 이벤트 연결 역할

---

# 최종 정리

- API Gateway  
  → "외부 요청을 받는 입구"

- EventBridge  
  → "내부 이벤트를 전달하는 허브"

- API Gateway  
  → "누군가 요청했다"

- EventBridge  
  → "무언가 발생했다"