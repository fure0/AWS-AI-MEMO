# VPC Endpoint와 Endpoint 개념 정리

## 1. Endpoint란?

Endpoint = End + Point

의미:
- 통신이 도달하는 최종 지점
- IT에서는 요청(Request)이 도착하는 접점(주소)

즉:
> 클라이언트가 요청을 보내는 목적지

---

## 2. Endpoint 예시

### Web API

예:
https://api.example.com/users

- API 요청을 받는 주소
- `/users` 라는 기능으로 접근하는 입구

---

### AWS 서비스

예:
S3 Endpoint

https://s3.amazonaws.com

- S3 서비스에 접근하기 위한 주소

---

### Database

예:

db-xxxxx.ap-northeast-1.rds.amazonaws.com

- DB 접속 주소
- DB Endpoint

---

## 3. Endpoint와 Server 관계

Client
  |
  | Request
  ↓
Server


- Server:
  실제 처리를 수행하는 곳

- Endpoint:
  Server에 접근하기 위한 입구


즉:

Endpoint ≠ Server

Endpoint = Server로 들어가는 접점

---

# VPC Endpoint란?

정의:

> VPC 내부에서 AWS 서비스로 접근하기 위한 프라이빗 연결 통로

즉:

AWS 서비스의 Endpoint를 인터넷을 거치지 않고 VPC 내부에서 접근하게 만드는 기능

---

# 일반적인 AWS 접근 방식

EC2
 |
 Internet Gateway
 |
 Internet
 |
 AWS Service Endpoint
 |
 S3


문제:

- 인터넷 경유
- 보안 관리 필요
- NAT Gateway 비용 발생 가능

---

# VPC Endpoint 사용

EC2
 |
 VPC Endpoint
 |
 S3


장점:

- 인터넷 불필요
- AWS 내부 네트워크 사용
- 보안 향상
- NAT Gateway 비용 절감 가능

---

# VPC Endpoint 종류

## 1. Gateway Endpoint

방식:

Route Table 기반 연결


구조:

EC2
 |
Route Table
 |
Gateway Endpoint
 |
S3 / DynamoDB


특징:

- 대상:
  - S3
  - DynamoDB

- 비용:
  - 무료

- 동작:
  - Route Table에 경로 추가


---

## 2. Interface Endpoint

방식:

ENI(Elastic Network Interface)를 생성


구조:

EC2
 |
Private IP
 |
Interface Endpoint
 |
AWS Service


특징:

- 대상:
  - 대부분 AWS 서비스

예:
- SQS
- SNS
- API Gateway
- Secrets Manager


비용:

- 시간당 비용
- 데이터 처리 비용


---

# 왜 Gateway Endpoint와 Interface Endpoint가 나뉘었나?

이유:

구현 방식이 다르기 때문

---

# Gateway Endpoint 등장 이유

초기 AWS 주요 서비스:

- S3
- DynamoDB


문제:

- 트래픽이 매우 많음
- NAT 비용 증가
- 인터넷 경유 비효율


해결:

"라우팅 레벨에서 AWS 내부로 바로 연결하자"


결과:

Gateway Endpoint


특징:

- 빠름
- 대량 트래픽 적합
- 무료

---

# Interface Endpoint 등장 이유

이후 요구:

- SQS
- SNS
- API Gateway
- 기타 AWS 서비스


문제:

모든 AWS 서비스를 라우팅 방식으로 처리하기 어려움


해결:

"VPC 안에 서비스 접근용 네트워크 인터페이스를 만들자"


결과:

Interface Endpoint


특징:

- Private IP 제공
- DNS 기반 접근
- 범용적

---

# Gateway Endpoint vs Interface Endpoint

| 항목 | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| 방식 | Route Table 기반 | ENI 기반 |
| 연결 | 네트워크 경로 변경 | Private IP 생성 |
| 대상 | S3, DynamoDB | 대부분 AWS 서비스 |
| 비용 | 무료 | 유료 |
| 유연성 | 낮음 | 높음 |

---

# 비유

## Gateway Endpoint

고속도로 IC

- 길 자체를 연결
- 대량 이동에 적합
- 빠르고 저렴


## Interface Endpoint

건물 전용 출입구

- 서비스마다 입구 생성
- 유연하지만 비용 발생

---

# 최종 정리

Endpoint:

> 요청이 도착하는 최종 접점


VPC Endpoint:

> AWS 서비스 Endpoint를 VPC 내부에서 프라이빗하게 접근하도록 만든 연결


Gateway Endpoint:

> S3/DynamoDB를 위한 네트워크 레벨 최적화 방식


Interface Endpoint:

> 대부분 AWS 서비스를 위한 범용 프라이빗 연결 방식


핵심:

Gateway Endpoint = 특수 목적 최적화

Interface Endpoint = 범용 솔루션