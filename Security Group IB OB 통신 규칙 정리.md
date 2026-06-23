# AWS Security Group Inbound / Outbound 통신 규칙 정리

AWS Security Group은 **Stateful(상태 저장) 방화벽**이다.

즉, 요청(Request)을 허용하면 해당 요청에 대한 응답(Response)은 자동으로 허용된다.

---

## 기본 원칙

| 통신 방향 | 필요한 설정 |
|---|---|
| A → B 요청 | A의 Outbound + B의 Inbound 필요 |
| B → A 응답 | 기존 세션이면 자동 허용 |
| B → A 새로운 요청 | B의 Outbound + A의 Inbound 필요 |

---

## 예시: ALB → ECS 통신

구조:

```
Client
   |
   ↓
 ALB
   |
   ↓
 ECS
```

---

## 1. Client → ALB

요청:

```
Client → ALB : HTTPS 443
```

ALB Security Group:

```
Inbound
TCP 443
Source: 0.0.0.0/0
```

필요

응답:

```
ALB → Client
```

은 기존 TCP 세션의 응답이므로 별도 Outbound 설정 불필요

(Security Group은 Stateful)

---

## 2. ALB → ECS

요청:

```
ALB → ECS : TCP 8080
```

ALB Security Group:

```
Outbound
TCP 8080
Destination: ECS Security Group
```

필요


ECS Security Group:

```
Inbound
TCP 8080
Source: ALB Security Group
```

필요

---

## ECS 응답

```
ECS → ALB
```

은 기존 요청에 대한 Response이므로:

```
ECS Outbound
```

별도 허용 필요 없음

(Security Group Stateful)

---

# 일반적인 AWS 구성 예

## ALB Security Group

```
Inbound:
  HTTPS 443
  Source: Internet (0.0.0.0/0)


Outbound:
  TCP 8080
  Destination: ECS Security Group
```

---

## ECS Security Group

```
Inbound:
  TCP 8080
  Source: ALB Security Group


Outbound:
  Default Allow All
```

---

# 주의: Security Group vs Network ACL

## Security Group

- Stateful
- 응답 트래픽 자동 허용
- 요청 방향 중심으로 설정

## Network ACL

- Stateless
- 요청/응답 모두 명시 필요
- Inbound / Outbound 규칙을 양쪽 모두 맞춰야 함

```
Security Group = Stateful
Network ACL    = Stateless
```