# Route 53의 A 레코드와 CNAME 레코드

## A 레코드 (Address Record)

도메인 이름을 **IP 주소에 직접 연결**합니다.

예시:

```text
example.com -> 192.0.2.10
```

Route 53 설정:

```text
이름: example.com
타입: A
값: 192.0.2.10
```

사용자가 `example.com`에 접속하면 DNS는 바로 `192.0.2.10`을 반환합니다.

### 사용 예

- EC2에 직접 연결
- 고정 IP를 가진 서버 연결
- Route 53 Alias를 통한 ALB, CloudFront, S3 연결

---

## CNAME 레코드 (Canonical Name)

도메인 이름을 **다른 도메인 이름에 연결**합니다.

예시:

```text
www.example.com
      ↓
example.com
      ↓
192.0.2.10
```

Route 53 설정:

```text
이름: www.example.com
타입: CNAME
값: example.com
```

DNS 조회 과정:

```text
www.example.com ?
→ example.com 입니다.
→ example.com ?
→ 192.0.2.10 입니다.
```

즉, CNAME은 다른 도메인을 가리키며 최종 IP는 대상 도메인을 통해 조회됩니다.

### 사용 예

- `www.example.com` → `example.com`
- 서브도메인 → ALB DNS 이름
- 서브도메인 → CloudFront DNS 이름

---

## A 레코드와 CNAME 비교

| 항목 | A 레코드 | CNAME |
|--------|--------|--------|
| 연결 대상 | IP 주소 | 다른 도메인 |
| DNS 조회 횟수 | 1번 | 추가 조회 발생 |
| 루트 도메인(example.com) 사용 | 가능 | 불가능 |
| IP 변경 대응 | 직접 수정 필요 | 대상 도메인만 수정하면 됨 |

---

## Route 53에서 자주 사용하는 구성

### 1. EC2에 직접 연결

```text
example.com
    ↓
A Record
    ↓
54.123.45.67
```

---

### 2. www를 루트 도메인으로 연결

```text
www.example.com
    ↓
CNAME
    ↓
example.com
```

---

### 3. ALB(Application Load Balancer) 연결

ALB는 IP 주소가 변경될 수 있으므로 직접 A 레코드로 연결하지 않습니다.

ALB DNS 이름:

```text
my-alb-123.ap-northeast-1.elb.amazonaws.com
```

일반 DNS 방식:

```text
www.example.com
    ↓
CNAME
    ↓
my-alb-123.ap-northeast-1.elb.amazonaws.com
```

---

### 4. Route 53 Alias 사용 (AWS 권장)

```text
example.com
    ↓
A Record (Alias)
    ↓
ALB
```

Alias는 Route 53의 AWS 전용 기능으로 AWS 리소스와 직접 연결할 수 있습니다.

---

## Alias 레코드란?

Alias는 AWS 리소스(ALB, CloudFront, S3 등)를 대상으로 하는 Route 53 전용 기능입니다.

```text
example.com
    ↓
A (Alias)
    ↓
ALB
```

장점:

- 루트 도메인 사용 가능
- AWS 리소스 IP 변경 자동 대응
- 추가 DNS 조회 없음
- AWS 서비스와 통합

---

## AWS 시험(SAA) 핵심 정리

```text
A 레코드
= 이름 → IP 주소

CNAME
= 이름 → 도메인 이름

Alias
= 이름 → AWS 리소스
```

### 기억할 것

- A 레코드: 도메인을 IP 주소에 연결
- CNAME: 도메인을 다른 도메인에 연결
- 루트 도메인(example.com)은 CNAME 사용 불가
- Route 53에서는 루트 도메인에 Alias 사용
- ALB, CloudFront, S3 연결 시 Alias 사용이 일반적

---

## 실무에서 가장 흔한 구성

```text
example.com
    ↓
A (Alias)
    ↓
ALB

www.example.com
    ↓
CNAME
    ↓
example.com
```

이 구성은 AWS 환경에서 가장 많이 사용되는 DNS 설정 패턴입니다.