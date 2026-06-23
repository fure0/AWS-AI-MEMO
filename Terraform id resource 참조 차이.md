# Terraform에서 `.id` 와 `.arn` 참조 차이

Terraform에서 AWS 리소스를 서로 참조할 때 어떤 경우에는:

```hcl
aws_xxx.resource.id
```

를 사용하고,

어떤 경우에는:

```hcl
aws_xxx.resource.arn
```

을 사용하는 이유는 AWS 리소스를 식별하는 방식과 사용하는 목적이 다르기 때문이다.

---

# 핵심 차이

| 구분 | 의미 | 용도 |
|---|---|---|
| `.id` | 해당 AWS 리소스의 식별자 | AWS API 내부 연결 |
| `.arn` | AWS 전체에서 리소스를 표현하는 주소 | 서비스 간 참조, IAM 권한 |

간단히:

```
.id
= AWS 서비스가 리소스를 찾기 위한 값

.arn
= AWS 전체 환경에서 리소스를 표현하는 완전한 주소
```

---

# 1. `.id` 를 사용하는 경우

AWS API가 특정 리소스를 지정할 때 ID를 요구하는 경우 사용한다.

예:

```hcl
resource "aws_subnet" "private" {
  vpc_id = aws_vpc.main.id
}
```

결과:

```
aws_vpc.main.id

↓

vpc-0123456789abcdef
```

Subnet 생성 API는:

```json
{
  "VpcId": "vpc-0123456789abcdef"
}
```

처럼 VPC ID를 요구한다.

---

## `.id` 사용 예

### VPC → Subnet

```hcl
vpc_id = aws_vpc.main.id
```

### Security Group 연결

```hcl
security_group_id = aws_security_group.app.id
```

### ECS / EC2 내부 연결

```hcl
subnet_id = aws_subnet.private.id
```

---

# 2. `.arn` 을 사용하는 경우

IAM 정책이나 AWS 서비스 간 연결처럼
"정확히 어떤 리소스인지" 표현해야 하는 경우 사용한다.

예:

```hcl
resource "aws_iam_policy" "s3_access" {

  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"

        Action = [
          "s3:GetObject"
        ]

        Resource = aws_s3_bucket.data.arn
      }
    ]
  })
}
```

S3 ARN:

```
arn:aws:s3:::my-bucket
```

IAM은:

```
어떤 서비스?
어느 계정?
어느 리전?
어떤 리소스?
```

정보가 필요하기 때문에 ARN을 사용한다.

---

# 왜 ARN이 필요한가?

"ID도 유니크한데 왜 ARN이 필요한가?"

답:

## 유니크 범위(scope)가 다르기 때문이다.

예:

계정 A:

```
i-012345
```

계정 B:

```
i-012345
```

가능하다.

또:

리전 A:

```
ap-northeast-1
i-012345
```

리전 B:

```
us-east-1
i-012345
```

가능하다.

---

ARN:

```
arn:aws:ec2:ap-northeast-1:123456789012:instance/i-012345
```

에는:

```
서비스
리전
AWS 계정
리소스 ID
```

정보가 모두 포함되어 있다.

---

# 3. ARN이 있으면 항상 ARN을 쓰면 안 되는 이유

## 이유 1. 모든 AWS 리소스가 ARN을 가지는 것은 아님

일부 리소스는 ARN이 없다.

예:

```
aws_route_table_association
```

같은 연결 리소스는:

```
id
```

만 존재하는 경우가 있다.

---

## 이유 2. AWS API가 ID를 요구하는 경우가 있음

예:

```hcl
resource "aws_security_group_rule" "allow" {

  security_group_id =
    aws_security_group.app.id

}
```

AWS API는:

```json
{
  "GroupId": "sg-123456"
}
```

형태를 요구한다.

ARN이 존재하더라도 API 입력 형식이 ID이면 `.id` 를 사용한다.

---

# 4. Terraform 내부에서도 `.id` 가 중요함

Terraform State는 기본적으로 리소스를 ID로 추적한다.

예:

```json
{
  "resource": "aws_instance.web",
  "id": "i-0123456789"
}
```

Terraform 입장에서는:

```
현재 관리 중인 AWS 리소스를 어떻게 찾을 것인가?
```

가 필요하고,

기본 식별자가 `.id` 이다.

---

# 판단 기준

## 속성이 `_id` 로 끝난다

대부분:

```hcl
xxx_id = aws_resource.name.id
```

사용

예:

```hcl
vpc_id = aws_vpc.main.id

subnet_id = aws_subnet.private.id

security_group_id = aws_security_group.app.id
```

---

## 속성이 `_arn` 으로 끝난다

대부분:

```hcl
xxx_arn = aws_resource.name.arn
```

사용

예:

```hcl
target_arn = aws_lambda_function.api.arn

role_arn = aws_iam_role.app.arn
```

---

# 최종 정리

```
.id

- AWS 내부 리소스 연결
- AWS API 입력값
- Terraform state 식별자
- xxx_id 속성에 사용


.arn

- AWS 전체에서 리소스 주소
- IAM Policy
- EventBridge
- Lambda/SNS/SQS 연결
- xxx_arn 속성에 사용
```

결론:

> `.id` 는 AWS 서비스 안에서 리소스를 찾는 키  
> `.arn` 은 AWS 전체에서 리소스를 표현하는 주소

Terraform에서는 무조건 ARN을 쓰는 것이 아니라,
해당 속성이 요구하는 값(id인지 arn인지)에 맞춰 사용하는 것이 정석이다.