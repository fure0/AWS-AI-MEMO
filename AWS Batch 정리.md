# AWS Batch란?

AWS Batch는 AWS에서 제공하는 배치 처리(Job Processing)를 자동으로 관리해주는 서비스이다.

쉽게 말하면:

> 대량의 작업을 Job Queue에 넣어두면, 필요한 만큼 실행 환경(컨테이너/서버)을 준비해서 작업을 실행하고 종료 후 정리해주는 서비스

---

# 기존 PHP Batch → AWS Batch 전환 가능 여부

기존 환경:

```text
렌탈 서버
 └ Cron
     └ PHP Batch 실행
```

변경 목표:

```text
AWS Batch
 └ TypeScript Batch 실행
```

가능하다.

다만 AWS Batch는 Docker 컨테이너 기반으로 실행되므로 실행 방식이 변경된다.

---

# TypeScript를 AWS Batch에서 실행하는 구조

```text
TypeScript 코드
      ↓
Build
      ↓
JavaScript(Node.js)
      ↓
Docker 이미지 생성
      ↓
AWS Batch Job 실행
```

예:

```dockerfile
FROM node:20

WORKDIR /app

COPY . .

RUN npm install
RUN npm run build

CMD ["node", "dist/batch.js"]
```

---

# Cron은 AWS에서 어떻게 대체하는가?

기존:

```text
Cron
 ↓
PHP Batch 실행
```

AWS:

```text
EventBridge Scheduler (Cron)
        ↓
AWS Batch Job 실행
        ↓
Container 실행
```

변경:

- Cron → EventBridge Scheduler
- 서버에서 직접 실행 → AWS Batch Job 실행

---

# AWS Batch 실행 구조

```text
Job Definition
      ↓
Job 실행 요청
      ↓
Container 실행
      ↓
Batch 코드 수행
      ↓
종료
```

---

# AWS Batch Job과 Container 관계

질문:

> AWS Batch는 Batch 한 개당 한 컨테이너를 구축하는가?

정확히는:

```text
Job 1개 실행
      =
Container 1개 실행
```

이다.

구조:

```text
Job Definition
      ↓
Job
      ↓
Docker Container 생성
```

일반적인 경우:

```text
하나의 Job
 =
하나의 Container 실행
```

이라고 이해하면 된다.

---

# 실행 후 Container는 삭제되는가?

YES.

AWS Batch의 Container는 일회성 실행 환경이다.

흐름:

```text
Job 요청
   ↓
Container 생성
   ↓
TypeScript Batch 실행
   ↓
작업 완료
   ↓
Container 종료
   ↓
Container 삭제
```

즉:

- 계속 살아있는 서버 Container ❌
- 작업 실행용 임시 Container ⭕

---

# ECS Service와 AWS Batch 차이

## ECS Service

```text
Container 실행
      ↓
계속 유지
      ↓
API 서버 / Web 서버
```

용도:

- Backend API
- Web Application

---

## AWS Batch

```text
Container 생성
      ↓
작업 실행
      ↓
종료
```

용도:

- 데이터 처리
- 파일 변환
- 정산 처리
- 로그 분석
- 대량 작업

---

# AWS Batch 장점

## 1. 서버 관리 제거

기존:

```text
서버 준비
OS 관리
Cron 관리
프로세스 관리
```

필요

AWS Batch:

```text
Job 등록
 ↓
AWS가 실행 환경 관리
```

---

## 2. 자동 스케일링

작업 증가:

```text
Job 증가
 ↓
Container 증가
 ↓
병렬 처리
```

작업 종료:

```text
Container 제거
```

---

## 3. 비용 최적화

작업이 없을 때:

```text
실행 환경 없음
→ 비용 최소화
```

필요할 때만 실행 가능

---

# Lambda / ECS Scheduled Task와 비교

| 항목 | AWS Batch | Lambda | ECS Scheduled Task |
|---|---|---|---|
| 실행 방식 | Container Job | Function | Container Task |
| 장시간 작업 | 가능 | 제한 있음 | 가능 |
| 서버 관리 | 거의 없음 | 없음 | 일부 필요 |
| 대량 병렬 처리 | 강함 | 제한 | 가능 |
| 용도 | 대규모 Batch | 짧은 이벤트 | 단순 배치 |

---

# 최종 아키텍처

기존:

```text
렌탈 서버
 └ Cron
     └ PHP Batch
```

변경 후:

```text
EventBridge Scheduler
        ↓
AWS Batch Job
        ↓
Docker Container
        ↓
Node.js(TypeScript)
        ↓
작업 완료
        ↓
Container 삭제
```

---

# 핵심 요약

- TypeScript Batch는 AWS Batch에서 실행 가능
- 기존 Cron은 EventBridge Scheduler로 대체
- AWS Batch Job 실행 시 Container 생성
- Job 완료 후 Container 삭제
- Batch는 장시간/대량 작업용 실행 환경
- 항상 서버를 켜두는 방식이 아니라 필요할 때만 실행하는 구조