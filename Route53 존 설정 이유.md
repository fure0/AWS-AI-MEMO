# Route 53에서 Hosted Zone(존)은 왜 필요한가?

## 결론

**Hosted Zone은 Route 53이 특정 도메인의 DNS 정보를 관리한다는 선언이다.**

즉,

- Hosted Zone = 특정 도메인의 DNS 관리 공간
- Record = 그 공간 안에 들어가는 개별 DNS 설정

이라고 생각하면 된다.

---

## 비유

집 주소 관리로 비유하면,

### Hosted Zone

```
김씨네 아파트 관리사무소
```

### Record

```
101호 → 김철수
102호 → 김영희
103호 → 박민수
```

관리사무소(Hosted Zone)가 먼저 있어야 입주자 정보(Record)를 등록할 수 있다.

---

## 실제 DNS 구조

예를 들어,

```
example.com
```

도메인을 구매했다고 하자.

Route 53에 Hosted Zone을 생성하면 AWS는 다음과 같이 생각한다.

```
example.com에 대한 DNS 정보는
내가 관리할게
```

이제 이 공간 안에 DNS 레코드를 추가할 수 있다.

| 레코드 타입 | 이름 | 값 |
|------------|------|-----|
| A | example.com | 1.2.3.4 |
| CNAME | www | example.com |
| MX | @ | mail.example.com |

---

## Hosted Zone이 없다면?

만약 다음과 같은 레코드만 있다고 가정해보자.

```
A 레코드
www → 1.2.3.4
```

그러면 AWS는 다음을 알 수 없다.

```
www.example.com 인가?
www.google.com 인가?
www.mycompany.net 인가?
```

어느 도메인에 속한 레코드인지 구분할 수 없기 때문이다.

그래서 먼저

```
Hosted Zone
└ example.com
```

을 만들고,

그 안에

```
example.com
├ A      @      → 1.2.3.4
├ CNAME  www    → example.com
├ MX     @      → mail.example.com
```

과 같은 레코드를 등록하는 것이다.

---

## AWS 관점에서 이해하기

예를 들어 EC2 서버가 있다고 하자.

```
EC2
↓
54.10.20.30
```

사용자가

```
example.com
```

으로 접속하면,

```
example.com
↓
Route 53 Hosted Zone
↓
A Record
↓
54.10.20.30
```

순서로 IP를 찾아간다.

여기서

- Hosted Zone = "example.com의 DNS를 AWS가 관리하는 영역"
- A Record = "example.com을 54.10.20.30으로 연결하라"

를 의미한다.

---

## 한 줄 정리

**Hosted Zone은 DNS를 관리하는 '영역(관리 범위)'이고, Record는 그 영역 안에 저장되는 '실제 규칙'이다.**

그래서 Route 53에서는 항상

```
Hosted Zone 생성
→ Record 생성
```

순서로 작업하게 된다.