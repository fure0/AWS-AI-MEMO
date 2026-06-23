# AWS WAF는 CloudFront와 ALB 중 어디에 붙이는가?

일반적으로는 **CloudFront에 AWS WAF를 붙이는 구성이 가장 많이 사용되고 권장되는 패턴**이다.

다만 서비스 구조와 목적에 따라 ALB에 붙이거나 둘 다 사용하는 경우도 있다.

---

# 1. CloudFront + WAF + ALB (가장 일반적인 구성)

구조:

    Internet
       ↓
    CloudFront
       ↓
    WAF
       ↓
    ALB
       ↓
    ECS / EC2 / Lambda

## 특징

CloudFront Edge Layer에서 먼저 요청을 검사하고 차단한다.

## 장점

- 공격 트래픽을 Backend까지 전달하지 않음
- DDoS / Bot / 스캔 공격 대응에 유리
- ALB와 Backend 부하 감소
- 글로벌 사용자에게 빠른 응답 가능
- CloudFront 캐싱 활용 가능
- AWS Shield Standard 자동 적용

## 주로 사용하는 경우

- 공개 웹 서비스
- 글로벌 서비스
- API 서비스
- 정적 + 동적 콘텐츠 서비스

---

# 2. ALB + WAF 구성

구조:

    Internet
       ↓
    ALB + WAF
       ↓
    Backend

## 특징

CloudFront 없이 ALB에서 직접 WAF 검사를 수행한다.

## 사용하는 경우

- CloudFront가 필요 없는 서비스
- 내부 시스템
- 소규모 서비스
- 특정 지역 한정 서비스
- 빠른 구축이 필요한 경우

## 단점

- 공격 요청이 ALB까지 도달
- CloudFront Edge 방어 효과 없음
- 캐싱 기능 없음
- 글로벌 성능 개선 효과 없음

---

# 3. CloudFront + WAF + ALB + WAF 구성

구조:

    Internet
       ↓
    CloudFront + WAF
       ↓
    ALB + WAF
       ↓
    Backend

## 사용하는 경우

대규모 서비스나 보안 요구사항이 높은 경우 사용한다.

예:

CloudFront WAF:
- 국가 차단
- IP 차단
- Bot 차단
- Rate Limit

ALB WAF:
- 특정 API 보호
- 서비스 내부 규칙 적용
- Backend 앞단 추가 방어

## 단점

- 비용 증가
- 운영 복잡성 증가

---

# 실무에서 많이 사용하는 권장 패턴

일반적인 인터넷 서비스:

    Client
      ↓
    CloudFront
      ↓
    AWS WAF
      ↓
    ALB
      ↓
    ECS / EC2

CloudFront에서 최대한 공격을 제거하고,
ALB와 Backend는 정상 트래픽만 처리하도록 구성한다.

---

# ALB 직접 접근 차단

CloudFront + WAF를 사용하더라도 ALB가 인터넷에 그대로 노출되면 우회 접근이 가능하다.

보통 추가 설정:

- ALB Security Group 제한
- CloudFront Origin 검증
- CloudFront에서 오는 요청만 허용
- CloudFront IP Range 기반 허용

등으로 ALB 직접 접근을 막는다.

---

# 선택 기준

| 상황 | 추천 구성 |
|---|---|
| 인터넷 공개 서비스 | CloudFront + WAF |
| 글로벌 서비스 | CloudFront + WAF |
| API 서비스 | CloudFront + WAF |
| 내부 시스템 | ALB + WAF |
| 소규모 서비스 | ALB + WAF |
| 높은 보안 요구 | CloudFront + WAF + ALB + WAF |

---

# 결론

대부분의 AWS 웹 서비스에서는:

    CloudFront
       ↓
    WAF
       ↓
    ALB
       ↓
    Application

구성이 표준적인 선택이다.

CloudFront가 필요 없는 환경이라면:

    ALB + WAF

구성도 충분히 사용할 수 있다.