# TypeScript async/await vs AWS Lambda 차이와 장단점

## 1. 핵심 차이

결론부터 말하면:

- async/await = 코드 레벨의 비동기 처리
- AWS Lambda = 인프라/실행 환경 레벨의 서버리스 실행 방식

둘은 대체 관계가 아니라 목적이 다르다.

| 구분 | async/await | AWS Lambda |
|---|---|---|
| 레벨 | 코드 내부 | 실행 환경 |
| 목적 | 비동기 처리 | 서버 없이 코드 실행 |
| 실행 위치 | 애플리케이션 서버 내부 | AWS Lambda 환경 |
| 스케일링 | 서버에 의존 | 자동 확장 |

정리하면:

- async/await
  - 하나의 서버 안에서 작업을 효율적으로 처리

- Lambda
  - 실행 자체를 서버리스로 분리


---

# 2. AWS Lambda를 사용하는 장점

## 1) 서버 관리 불필요

Lambda를 사용하면 서버 운영 부담이 줄어든다.

관리하지 않아도 되는 것:

- EC2 서버 관리
- OS 패치
- 서버 프로비저닝
- 스케일링 관리

운영 부담 감소 효과가 있다.


---

## 2) 자동 스케일링

Lambda는 요청량에 따라 실행 환경을 자동으로 늘린다.

예:

요청 1개

Lambda 1개 실행

요청 1000개

여러 Lambda 실행


반면 async/await는:

Node.js 서버 내부에서 처리하기 때문에 서버 자체의 한계가 있다.


---

## 3) 사용한 만큼 과금

Lambda는 실행 시간 기준 과금이다.

장점:

- 서버를 항상 켜둘 필요 없음
- 사용하지 않을 때 비용 없음

특히:

- 배치 작업
- 이벤트 처리

에서 유리하다.


---

## 4) 이벤트 기반 구조에 적합

Lambda는 이벤트 기반 아키텍처와 잘 맞는다.

예:

- S3 파일 업로드
- API Gateway 요청
- SQS 메시지 처리
- CloudWatch 이벤트


구조:

Event 발생

↓

Lambda 실행

↓

처리 완료


---

# 3. AWS Lambda의 단점

## 1) Cold Start

Lambda가 처음 실행될 때 초기화 시간이 발생한다.

영향:

- 첫 요청 지연
- API 응답 시간 증가 가능


---

## 2) 실행 시간 제한

Lambda 최대 실행 시간:

15분


따라서:

부적합:

- 긴 배치
- 대규모 데이터 처리

적합:

- 짧은 이벤트 처리
- 단순 작업


---

## 3) 상태 유지 어려움

Lambda는 기본적으로 Stateless 구조이다.

즉:

Lambda 실행

↓

처리

↓

종료


메모리 상태 유지가 보장되지 않는다.

필요한 경우:

- Redis
- DynamoDB
- RDS

같은 외부 저장소 사용


---

## 4) 디버깅 어려움

일반 서버:

Local 실행

↓

Debug

↓

수정


Lambda:

배포

↓

CloudWatch 로그 확인

↓

수정


방식이라 개발 경험이 다르다.


---

## 5) 아키텍처 복잡도 증가

Lambda 사용 시 추가 고려사항:

- IAM 권한
- Trigger 설정
- Retry 처리
- Timeout 설정
- Event 구조 설계


간단한 서비스에서는 오히려 복잡할 수 있다.


---

# 4. async/await만 사용하면 충분한 경우

다음 상황에서는 Lambda가 필요 없다.

예:

- 항상 실행 중인 API 서버
- ECS/EC2 Backend
- DB 조회
- 외부 API 호출
- 일반적인 비즈니스 로직


예:

```typescript
const user = await getUser();

const order = await getOrder();
```


이런 처리는 Backend 내부 async/await로 충분하다.


---

# 5. Lambda가 적합한 상황

## 1) 이벤트 기반 처리

예:

회원 가입

↓

Event 발생

↓

Lambda

↓

메일 발송


---

## 2) 비동기 작업 분리

기존 구조:

API 요청

↓

- 결제 처리
- 이메일 발송
- 로그 저장
- 알림 처리

↓

응답


개선:

API 요청

↓

응답


비동기:

SQS

↓

Lambda

↓

이메일

로그

알림


---

## 3) 주기적 배치

구조:

CloudWatch Scheduler

↓

Lambda

↓

Batch 처리


---

## 4) 트래픽 예측이 어려운 서비스

예:

- 이벤트 서비스
- 갑작스러운 요청 증가

Lambda는 자동 확장이 가능하다.


---

# 6. 실제 서비스 아키텍처 예

일반적인 Backend:

Client

↓

ALB

↓

ECS(Node.js API)

↓

Database


여기서:

- API 요청 처리
- 비즈니스 로직

→ async/await 사용


---

무거운 작업:

Client

↓

API

↓

SQS

↓

Lambda

↓

처리


처럼 분리한다.


---

# 7. 쉽게 비유하면

## async/await

"한 명의 개발자가 일을 효율적으로 처리하는 방식"

- 기다리는 시간을 활용해서 다른 작업 수행


## Lambda

"필요할 때마다 새로운 작업자를 자동으로 투입하는 방식"

- 요청 증가
- 실행 환경 증가


---

# 8. 최종 정리

## async/await

- 애플리케이션 내부 비동기 처리 기술
- Backend 개발 기본 기술
- 서버 내부 작업 효율화 목적


## Lambda

- 실행 환경을 분리하는 AWS 아키텍처 선택
- 이벤트 처리에 적합
- 배치 처리에 적합
- 비동기 작업 분리에 적합


결론:

> async/await는 "코드를 어떻게 실행할 것인가"

> Lambda는 "어디에서 실행할 것인가"

를 결정하는 기술이다.