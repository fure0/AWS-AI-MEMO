# AWS WAF 룰 정리

> Web ACL 안에 넣는 **규칙(Rule)** 종류, 자주 쓰는 것, 권장 구성.  
> 구조 복습: `Web ACL` ⊃ `Rule` / `Rule Group` (Managed 포함)

```text
요청 → Web ACL (위에서부터 순서대로 검사)
         ├─ Rule / Rule Group …
         └─ Default Action (보통 Allow)
```

---

## 1. 룰이 하는 일 (Action)

| Action | 의미 | 언제 |
|---|---|---|
| **Allow** | 통과 | 화이트리스트 IP, 예외 경로 |
| **Block** | 차단 | 공격·악성 패턴 |
| **Count** | 통과시키되 **기록만** | 신규 룰 테스트 (오탐 확인용) |
| **CAPTCHA** | 사람 확인 페이지 | 로그인·봇 의심 |
| **Challenge** | 조용한 JS 챌린지 | 봇 완화 (사용자 덜 거슬림) |

**권장:** 새 Managed Rule은 먼저 **Count** → CloudWatch/Sampled requests로 오탐 확인 → **Block**으로 전환.

---

## 2. 룰 종류

| 종류 | 설명 | 과금 감각 |
|---|---|---|
| **Custom Rule** | 내가 만든 단일 규칙 (IP, 경로, 헤더, 문자열 등) | Rule 1개 ≈ $1/월 |
| **Rule Group (직접 생성)** | 내 Rule들을 묶은 세트 | 그룹 $1 + 안쪽 Rule마다 $1 |
| **AWS Managed Rule Group** | AWS가 유지보수하는 규칙 묶음 | ACL에 붙인 그룹 1개 ≈ $1/월 (안쪽 개수 과금 X) |
| **Marketplace Managed Rule** | 서드파티 룰셋 | 판매자 요금 + WAF 기본 요금 |
| **Rate-based Rule** | 같은 IP(또는 키)가 N초에 너무 많이 치면 차단 | Custom Rule과 동일 계열 |

---

## 3. Custom Rule에서 자주 쓰는 조건

| 조건 | 무엇을 검사 | 예 |
|---|---|---|
| **IP set** | 출발 IP | 회사 IP만 Allow, 악성 IP Block |
| **Geo match** | 국가 | 서비스 안 하는 국가 Block |
| **Rate-based** | 요청 속도 | 5분에 2000회 초과 IP Block |
| **URI path** | 경로 | `/admin`은 특정 IP만 |
| **HTTP method** | GET/POST 등 | 불필요한 METHOD Block |
| **String / Regex** | 헤더·쿼리·바디 문자열 | 특정 User-Agent, 공격 패턴 |
| **Size constraint** | 바디/헤더 크기 | 비정상적으로 큰 요청 |
| **Label** | 앞 룰이 붙인 라벨 | Anonymous IP 라벨 달린 요청만 Rate limit |

---

## 4. 자주 쓰는 · 권장 AWS Managed Rule

### 거의 항상 권장 (Baseline)

| Managed Rule Group | 무엇을 막나 | 메모 |
|---|---|---|
| **Core rule set (CRS)** `AWSManagedRulesCommonRuleSet` | XSS, LFI, 이상한 요청 등 OWASP 계열 기본 공격 | **가장 기본.** 오탐 가능성 있음 → Count부터 |
| **Known bad inputs** `AWSManagedRulesKnownBadInputsRuleSet` | Log4j 등 알려진 악성 입력 패턴 | CRS와 함께 기본으로 많이 씀 |
| **Amazon IP reputation list** `AWSManagedRulesAmazonIpReputationList` | AWS가 악성으로 본 IP | 넓게 막기 좋음. 우선순위 앞쪽 |
| **Anonymous IP list** `AWSManagedRulesAnonymousIpList` | VPN / Proxy / Tor / 호스팅 IP | 소비자 VPN은 오탐 가능. Count 후 판단 |

### 앱 종류에 따라 추가

| Managed Rule Group | 무엇을 막나 | 언제 |
|---|---|---|
| **SQL database** `AWSManagedRulesSQLiRuleSet` | SQL Injection | RDS/DB 있는 웹·API |
| **Linux / Unix** | OS 명령 주입 등 | 리눅스 서버 백엔드 |
| **Windows** | 윈도우 계열 공격 | 윈도우 백엔드일 때 |
| **PHP** | PHP 취약 패턴 | PHP 앱 |
| **WordPress** | WP 특화 | WordPress |

### 돈 더 내는 고급 (필요할 때)

| Managed Rule Group | 무엇을 막나 | 메모 |
|---|---|---|
| **Bot Control** | 알려진 봇·스크래퍼 | **요청당 과금.** `/login`, API 등 **민감 경로만** scope-down |
| **ATP** (Account Takeover) | 로그인 자격증명 스터핑 | 로그인 URL에만 |
| **ACFP** (Account Creation Fraud) | 가짜 회원가입 | 가입 URL에만 |

---

## 5. 권장 룰 순서 (위 → 아래)

위에서 많이 걸러야 뒤쪽(유료·무거운 룰) 비용·부하가 줄어든다.

```text
1. (선택) 명시적 Allow — 회사 IP, 헬스체크
2. Rate-based (전체) — 너무 빠른 IP 차단
3. Rate-based (경로별) — /login, /signup 더 빡세게
4. Geo / IP set Block
5. IP Reputation + Anonymous IP
6. AWS Managed Baseline — CRS, Known Bad Inputs
7. 앱별 Managed — SQLi, Linux …
8. 오탐 예외 Allow / Count 조정
9. (유료) Bot Control / Fraud — 경로 좁혀서
```

---

## 6. 적절한 설정 (실무 체크리스트)

### 붙이는 위치
| 구성 | 권장 |
|---|---|
| CloudFront + ALB | **CloudFront에 WAF** (엣지에서 먼저 차단, 오리진 부하↓) |
| ALB만 | ALB에 WAF |
| 둘 다 붙이기 | 가능하지만 비용·중복 검사. 보통 한곳 |

CloudFront용 Web ACL은 **us-east-1**에 생성.

### 기본 운영
1. Default Action = **Allow**
2. Baseline Managed Rule = 먼저 **Count**
3. WAF → Sampled requests / CloudWatch로 오탐 확인
4. 문제 없는 룰부터 **Block**
5. 오탐 나는 경로만 **예외** (URI scope / label 기반 Allow)

### CRS에서 자주 오탐 나는 것
| 룰 성격 | 왜 터지나 | 대응 |
|---|---|---|
| **SizeRestrictions_BODY** | 바디 8KB 초과 (업로드·큰 JSON) | 업로드 API만 예외 |
| **XSS_*_BODY** | HTML/XML 비슷한 업로드·에디터 | 해당 경로 예외 또는 Count 유지 |

### Rate-based 감각 (시작점)
| 대상 | 예시 시작값 | 메모 |
|---|---|---|
| 전체 사이트 | 5분에 2000~3000 req/IP | 트래픽 보고 조정 |
| `/login`, `/signup` | 더 낮게 (예: 5분에 100~300) | Anonymous IP 라벨과 조합 가능 |
| 정적 파일 | Rate보다 Bot scope에서 제외 | 이미지·CSS는 Bot Control 대상에서 빼기 |

### Anonymous IP
- **Tor/VPN 차단:** B2B·어드민이면 Block 검토
- **HostingProviderIPList:** 브라우저 사용자만 받는 페이지에 유용. **서버→서버 API·웹훅**이면 Block 주의 (클라우드 IP가 정상일 수 있음)

### Bot Control
- 전체 Distribution에 무차별 적용 ❌ (비용)
- `/api/*`, `/login` 등만 scope-down ✅

---

## 7. 최소 추천 세트 (작은 웹/API)

| 우선순위 | 구성 | Action |
|---|---|---|
| 1 | Rate-based (전체) | Block |
| 2 | Amazon IP Reputation | Block (또는 Count→Block) |
| 3 | Core Rule Set (CRS) | Count → Block |
| 4 | Known Bad Inputs | Count → Block |
| 5 | (DB 있으면) SQLi Rule Set | Count → Block |
| 6 | `/login` Rate-based | Block |

나중에: Anonymous IP(Count), Bot Control(로그인만).

---

## 8. 과금 다시 보기 (룰 관련)

| 항목 | 대략 |
|---|---|
| Web ACL | $5/월 |
| Rule 또는 Managed Rule Group 1개 | $1/월 |
| 요청 검사 | $0.60 / 100만 |
| Bot Control 등 | 추가 구독 + 요청당 |

직접 만든 Rule 5개 = $5.  
Managed Group 1개(안 규칙 수십 개) = 보통 **$1**.  
→ 기본 방어는 **Managed Rule Group**이 가성비 좋음.

---

## 9. 한줄 요약

| 질문 | 답 |
|---|---|
| 뭘 꼭 넣지? | CRS + Known Bad Inputs + IP Reputation + Rate-based |
| 새 룰은? | 무조건 Count부터 |
| 어디에 붙이지? | CloudFront 있으면 엣지(CloudFront) |
| 비싸지는 건? | Bot/Fraud → 경로 좁히기 |
| 오탐 나면? | 그 룰만 Count/예외, 전체 끄지 않기 |
