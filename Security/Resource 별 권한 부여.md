# Resource 별 권한 부여

> 자주 쓰는 구성에서 **A가 B를 쓰려면** 무엇이 필요한지 정리.  
> **무엇을 하기 위해** = 목적 / **Actions** = IAM에 적는 API 이름 (`서비스:동작`).  
> IAM Role / Bucket Policy / Security Group이 섞여 있으니, 목적만 보고 “뭘 열어줘야 하지?”를 찾으면 됨.

---

## 한눈에 보는 참조 관계

```text
Route53 ──Alias──▶ CloudFront ──▶ WAF
                      │
                      ├─ Origin ──▶ S3
                      └─ Origin ──▶ ALB ──▶ ECS/EC2
                                            │
                         CodePipeline ──▶ CodeBuild ──push──▶ ECR
                                            │                    │
                                            └──── Deploy ────────┘
                                                      │
                                                   ECS pull
                                                      │
                         ECS / CodeBuild / ALB ──로그·메트릭──▶ CloudWatch
                                                                    │
                                                              Alarm ──▶ SNS 등

AWS Backup ──백업──▶ EBS / S3 / ECS 관련 볼륨 등
```

---

## 1. 엣지 · DNS · 보안

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| Route53 | CloudFront / ALB | 도메인 이름(`www.example.com`)을 CloudFront·ALB 주소로 연결하려고 | IAM 거의 불필요 (Alias A/AAAA 설정) |
| CloudFront | S3 | 사용자가 CloudFront로 파일 요청했을 때, S3에서 파일을 읽어 내려주려고 (버킷은 밖에 안 열고 CloudFront만 허용) | `s3:GetObject` (S3 **Bucket Policy** + OAC) |
| CloudFront | ALB | 동적 API/웹 요청을 CloudFront 뒤에 있는 ALB로 넘기려고 | IAM 아님. ALB **Security Group**에서 CloudFront 대역 허용 |
| 관리자 | WAF → CloudFront | 악성 요청을 CloudFront 앞에서 차단·필터하려고 Web ACL을 붙이려고 | `wafv2:AssociateWebACL`, `wafv2:GetWebACL`, `cloudfront:UpdateDistribution` (CloudFront용 ACL은 **us-east-1**) |
| 관리자 | WAF → ALB | ALB로 들어오는 요청을 WAF로 검사하려고 | `wafv2:AssociateWebACL`, `elasticloadbalancing:SetWebACL` |

**CloudFront가 S3 파일을 읽을 때 (Bucket Policy)**

```text
목적:     이 CloudFront만 S3 객체를 읽게 한다
Principal: cloudfront.amazonaws.com  (해당 Distribution ARN 조건)
Action:    s3:GetObject
Resource:  arn:aws:s3:::버킷이름/*
```

---

## 2. 로드밸런서 · 컴퓨트

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| ALB | ECS / EC2 | 로드밸런서가 받은 요청을 실제 서버(컨테이너)로 전달하려고 (네트워크 통로) | IAM 아님. **대상 SG**에 ALB SG로부터의 인바운드 허용 |
| ECS Service | ALB Target Group | 새로 뜬 태스크를 ALB 타겟에 등록·빼고, 헬스체크 대상으로 넣으려고 | `elasticloadbalancing:RegisterTargets`, `DeregisterTargets`, `DescribeTargetHealth` |
| ECS Task (실행) | ECR | 컨테이너를 띄우기 전에 **이미지 다운로드**하려고 | 아래 ECR pull 표 |
| ECS Task (실행) | CloudWatch Logs | 컨테이너 stdout/stderr를 CloudWatch에 **남기려고** | 아래 Logs 쓰기 표 |
| ECS Task (실행) | Secrets Manager | DB 비밀번호 등 시크릿을 태스크 시작 시 **주입**하려고 | `secretsmanager:GetSecretValue` (+ 암호화 시 `kms:Decrypt`) |
| ECS Task (실행) | SSM Parameter | 설정값을 Parameter Store에서 **읽어 주입**하려고 | `ssm:GetParameters`, `ssm:GetParameter` (+ `kms:Decrypt`) |
| ECS Task (앱) | S3 | 앱 코드가 파일 업로드·다운로드 등 **비즈니스 로직**으로 S3를 쓰려고 | `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` |
| EC2 | S3 / 기타 | EC2 위 앱/스크립트가 AWS API를 호출하려고 (역할 없으면 자격증명 없음) | Task Role과 같은 방식 → **Instance Profile**에 Actions 부여 |

### ECS Role 두 종류 (목적이 다름)

| Role | 무엇을 하기 위해 | 대표 Actions |
|---|---|---|
| **Task Execution Role** | ECS가 태스크를 **준비**할 때 (이미지 받고, 로그 보내고, 시크릿 읽기) | ECR pull, `logs:PutLogEvents`, Secrets/SSM 읽기 |
| **Task Role** | 컨테이너 **안 앱**이 운영 중 AWS를 호출할 때 | `s3:GetObject` 등 비즈니스 API |

```text
이미지 pull · 로그 전송 · 시크릿 주입  → Execution Role
앱이 S3 업로드/다운로드               → Task Role
```

### ECS Execution Role → ECR (이미지를 받아오려고)

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| Execution Role | ECR | 로그인 토큰을 발급받으려고 | `ecr:GetAuthorizationToken` |
| Execution Role | ECR Repository | 레이어가 있는지 확인하려고 | `ecr:BatchCheckLayerAvailability` |
| Execution Role | ECR Repository | 레이어 다운로드 URL을 받으려고 | `ecr:GetDownloadUrlForLayer` |
| Execution Role | ECR Repository | 이미지 매니페스트/레이어를 가져오려고 | `ecr:BatchGetImage` |

`GetAuthorizationToken` Resource는 보통 `*`  
나머지는 repo ARN: `arn:aws:ecr:리전:계정:repository/이름`

### ECS Execution Role → CloudWatch Logs (컨테이너 로그를 남기려고)

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| Execution Role | Log Group | 로그 그룹이 없을 때 만들려고 | `logs:CreateLogGroup` |
| Execution Role | Log Stream | 태스크별 로그 스트림을 만들려고 | `logs:CreateLogStream` |
| Execution Role | Log Stream | 실제 로그 한 줄을 전송하려고 | `logs:PutLogEvents` |
| Execution Role | Log Group | 기존 스트림 목록을 확인하려고 | `logs:DescribeLogStreams` |

Resource 예: `arn:aws:logs:리전:계정:log-group:/ecs/앱이름:*`

---

## 3. CloudWatch (로그 · 메트릭 · 알람)

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| ECS Task (실행) | CloudWatch Logs | ECS 컨테이너 표준 출력을 CloudWatch에 쌓으려고 | `logs:CreateLogStream`, `logs:PutLogEvents` (+ `logs:CreateLogGroup`) |
| CodeBuild | CloudWatch Logs | 빌드 과정의 출력을 나중에 디버깅하려고 | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` |
| EC2 / 앱 (직접 전송) | CloudWatch Logs | 에이전트·앱이 직접 로그를 밀어 넣으려고 | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, `logs:DescribeLogStreams` |
| EC2 / 앱 | CloudWatch Metrics | 내가 정의한 수치(큐 길이 등)를 그래프로 보려고 | `cloudwatch:PutMetricData` |
| ALB / ECS / AWS | CloudWatch Metrics | CPU·요청 수 등 **기본 지표**를 자동으로 보려 할 때 | 추가 IAM 거의 불필요 (서비스가 알아서 전송) |
| CloudWatch Alarm | SNS | 임계값 넘으면 슬랙/메일 등으로 **알림을 보내려고** | `sns:Publish` (+ SNS Topic Policy에서 CloudWatch 허용) |
| 사람 / 다른 서비스 | Logs 조회 | 콘솔·API로 과거 로그를 읽어 확인하려고 | `logs:DescribeLogGroups`, `logs:DescribeLogStreams`, `logs:GetLogEvents`, `logs:FilterLogEvents` |
| 사람 / 다른 서비스 | Logs Insights | 로그를 쿼리해서 에러·패턴을 찾으려고 | `logs:StartQuery`, `logs:GetQueryResults`, `logs:StopQuery` |
| 사람 | Metrics / Alarm | 대시보드·알람 상태를 조회하려고 | `cloudwatch:GetMetricData`, `cloudwatch:DescribeAlarms`, `cloudwatch:ListMetrics` |
| ECS/EC2 (Private) | CloudWatch Logs | 인터넷(NAT) 없이 Private에서 로그만 AWS로 보내려고 | Interface Endpoint(`logs`) + SG 443 + 위 쓰기 Actions |

```text
목적별 흐름
  로그 남기기     → logs:PutLogEvents
  커스텀 수치 기록 → cloudwatch:PutMetricData
  알람 알림       → sns:Publish
```

**같은 “로그 쓰기”라도 Role이 다른 이유**

| 상황 | 어떤 Role | 왜 |
|---|---|---|
| ECS `awslogs`로 stdout 수집 | **Execution Role** | ECS 에이전트가 대신 보내기 때문 |
| 앱 코드가 SDK로 로그/메트릭 | **Task Role** | 앱 프로세스가 직접 호출하기 때문 |
| CodeBuild 빌드 로그 | **Build Role** | 빌드 컨테이너 권한이 Build Role이기 때문 |

---

## 4. 스토리지 · 백업

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| ECS / EC2 | S3 | 파일을 **다운로드(읽기)** 하려고 | `s3:GetObject` |
| ECS / EC2 | S3 | 파일을 **업로드(쓰기)** 하려고 | `s3:PutObject` |
| ECS / EC2 | S3 | 파일을 **삭제** 하려고 | `s3:DeleteObject` |
| ECS / EC2 | S3 | 버킷 안 객체 **목록**을 보려고 | `s3:ListBucket` (Resource = 버킷 ARN) |
| CloudFront | S3 | CDN이 오리진 파일을 가져와 사용자에게 주려고 | `s3:GetObject` (Bucket Policy) |
| AWS Backup | Backup 서비스 | 백업 작업을 시작·중지·상태 확인하려고 | `backup:StartBackupJob`, `backup:StopBackupJob`, `backup:DescribeBackupJob` |
| AWS Backup | EBS | 디스크 스냅샷을 떠서 백업하려고 | `ec2:CreateSnapshot`, `ec2:DescribeVolumes`, `ec2:DescribeSnapshots`, `ec2:CreateTags` |
| AWS Backup | S3 | 버킷 내용을 읽어 백업본을 만들려고 | `s3:GetObject`, `s3:GetObjectVersion`, `s3:ListBucket`, `s3:ListBucketVersions`, `s3:GetBucketVersioning`, `s3:GetBucketNotification` |
| AWS Backup | S3 | 백업본에서 버킷으로 **복원(다시 쓰기)** 하려고 | `s3:PutObject`, `s3:AbortMultipartUpload`, `s3:ListMultipartUploadParts` 등 |

S3 Resource 구분:

```text
목록 보기 (ListBucket)  → arn:aws:s3:::버킷
파일 읽기/쓰기 (Object) → arn:aws:s3:::버킷/*
```

---

## 5. CI/CD 파이프라인

```text
Source → CodePipeline → CodeBuild → ECR → (Deploy) ECS
                │            │
                └─ Artifact S3
```

| From | To | 무엇을 하기 위해 | Actions |
|---|---|---|---|
| CodePipeline | S3 Artifact | 스테이지 사이에 빌드 결과물(zip 등)을 저장·전달하려고 | `s3:GetObject`, `s3:PutObject`, `s3:GetBucketLocation`, `s3:ListBucket`, `s3:AbortMultipartUpload` |
| CodePipeline | CodeBuild | 파이프라인이 빌드 프로젝트를 **실행**하고 상태를 보려고 | `codebuild:StartBuild`, `codebuild:BatchGetBuilds`, `codebuild:StopBuild` |
| CodePipeline | ECS | 새 Task Definition을 등록하고 서비스에 **배포(롤링)** 하려고 | `ecs:DescribeServices`, `ecs:DescribeTaskDefinition`, `ecs:DescribeTasks`, `ecs:ListTasks`, `ecs:RegisterTaskDefinition`, `ecs:UpdateService` |
| CodePipeline | IAM | ECS 태스크에 붙일 Role을 “대신 넘겨” 주려고 (없으면 배포 실패) | `iam:PassRole` |
| CodeBuild | ECR | 빌드한 Docker 이미지를 레지스트리에 **푸시**하려고 | `ecr:GetAuthorizationToken` + push Actions (아래) |
| CodeBuild | ECR | Dockerfile `FROM`용 base 이미지를 **받아** 오려고 | `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` |
| CodeBuild | S3 | 소스/아티팩트를 읽고 결과물을 다시 올리려고 | `s3:GetObject`, `s3:PutObject`, `s3:ListBucket` |
| CodeBuild | CloudWatch Logs | 빌드 로그를 남겨 실패 원인을 보려고 | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` |
| ECS | ECR | 배포된 태스크가 새 이미지를 **받아 실행**하려고 | §2 ECR pull과 동일 |
| ECS | CloudWatch Logs | 배포 후 앱 로그를 남기려고 | `logs:CreateLogStream`, `logs:PutLogEvents` |

### CodeBuild가 이미지를 올릴 때 (push)

```text
목적: ECR에 로그인 → 레이어 업로드 → 이미지 태그 등록

ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:InitiateLayerUpload
ecr:UploadLayerPart
ecr:CompleteLayerUpload
ecr:PutImage
```

### `iam:PassRole` — 무엇을 하기 위해?

```text
목적: CodePipeline이 “이 Role을 ECS Task에 붙여라”고 ECS에게 권한을 위임

CodePipeline
  └─ iam:PassRole ──▶ Execution Role / Task Role
                         └─ ECS Task에 부착
```

없으면: 이미지는 만들어도 **ECS 배포 단계에서 PassRole 오류**.  
권장 조건: `iam:PassedToService = ecs-tasks.amazonaws.com`

---

## 6. VPC · 네트워크 (통로를 열기 위해 — IAM이 아닐 때가 많음)

| From | To | 무엇을 하기 위해 | 설정 / Actions |
|---|---|---|---|
| 인터넷 사용자 | ALB | HTTPS로 사이트에 접속하게 하려고 | ALB SG Inbound TCP 443 (IAM 아님) |
| ALB | ECS/EC2 | ALB → 서버로 트래픽이 실제로 가게 하려고 | 대상 SG에 ALB SG 허용 (IAM 아님) |
| ECS/EC2 (Private) | 인터넷 | Private 서버가 외부 API·패키지를 받으려고 | NAT Gateway + 라우트 테이블 (IAM 아님) |
| ECS/EC2 | S3 | Private에서 S3를 쓰되 NAT 비용을 줄이려고 | Gateway Endpoint(경로) + IAM `s3:GetObject` 등 |
| ECS/EC2 | ECR | Private에서 이미지를 pull/push하려고 | Interface Endpoint + SG 443 + ECR Actions |
| ECS/EC2 | CloudWatch Logs | Private에서 로그만 보내려고 | Interface Endpoint + SG 443 + `logs:PutLogEvents` |

---

## 7. 역할 치트시트 (이 Role은 무엇을 하게 하려고?)

| 리소스 | Role | 무엇을 하게 하려고 |
|---|---|---|
| ECS Task | Execution Role | 이미지 받고, 로그 보내고, 시크릿 주입 (인프라 준비) |
| ECS Task | Task Role | 앱이 S3·메트릭 등 AWS API 호출 (비즈니스) |
| EC2 | Instance Profile | EC2 위 프로세스가 AWS API 호출 |
| CodePipeline | Pipeline Role | 빌드 시작, 아티팩트 저장, ECS 배포, Role 넘기기 |
| CodeBuild | Build Role | 빌드·이미지 push·빌드 로그 남기기 |
| AWS Backup | Backup Role | 스냅샷/백업 생성·복원 |
| CloudWatch Alarm | SNS 연동 | 알람 시 알림 발송 (`sns:Publish`) |
| CloudFront → S3 | Bucket Policy | CDN만 S3 파일 읽기 (`s3:GetObject`) |

---

## 자주 막히는 조합 TOP

| 증상 | 무엇을 못 해서? | 흔한 원인 |
|---|---|---|
| `CannotPullContainerError` | 이미지를 받아오지 못함 | Execution Role에 ECR pull Actions 없음 |
| ECS는 도는데 로그 없음 | 로그를 CloudWatch에 못 보냄 | Execution Role에 `logs:PutLogEvents` 없음 / Log Group 불일치 |
| CodePipeline PassRole 오류 | ECS에 Role을 넘겨 배포하지 못함 | Pipeline Role에 `iam:PassRole` 없음 |
| CloudFront 200, S3 직접 403 | (의도일 수 있음) 사람/인터넷은 S3 직접 읽기 금지 | Bucket Policy가 CloudFront만 `s3:GetObject` 허용 |
| S3 AccessDenied (앱) | 앱이 파일을 읽지/쓰지 못함 | **Task Role**에 S3 Actions 없음 (Execution에만 넣은 경우) |
| ALB 건강 · 타겟 Unhealthy | ALB → 서버 네트워크가 막힘 | 대상 SG 인바운드 미개방 (IAM 문제 아님) |
| Private에서 로그만 실패 | 로그 API 경로가 없음 | Logs Endpoint 없음 — Actions는 있어도 패킷이 안 감 |
