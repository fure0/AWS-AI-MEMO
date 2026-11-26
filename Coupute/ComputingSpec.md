# Computing Spec 정리 (EC2 / ECS / RDS)

> 기준: **서울 (ap-northeast-2)** · On-Demand · Linux / RDS는 **PostgreSQL Single-AZ**  
> 월 ≈ **730시간**, 원화 **$1 ≈ ₩1,350** (대략)  
> ECS on EC2 = 아래 EC2 스펙·비용과 동일. Fargate는 맨 아래.

---

## 이름 읽는 법

```text
t4g.small
│ │ │
│ │ └─ 크기 (nano → micro → small → medium → large → xlarge …)
│ └─── 세대 + CPU 종류 (g = Graviton/ARM, a = AMD, 숫자만 = Intel)
└───── 패밀리 (용도)
```

| 패밀리 | 의미 | 언제 |
|---|---|---|
| **t** | 버스트형. 평소 CPU 낮고 가끔 튀어도 됨 (크레딧) | 웹/API 초중반, 개발, 작은 서비스 |
| **m** | 범용. CPU·메모리 균형 | 일반 앱 서버 기본값 |
| **c** | 컴퓨트 최적화. CPU 비중 큼 | 배치, 인코딩, CPU 바운드 |
| **r** | 메모리 최적화. RAM 비중 큼 | DB, 캐시, 인메모리 |
| **g** (접미) | Graviton (ARM). 같은 급이면 보통 더 저렴 | 호환되면 우선 고려 |

RDS는 앞에 `db.`만 붙음 → `db.t4g.small`

---

## EC2 / ECS (자주 쓰는 스펙)

**vCPU·RAM은 스펙(인스턴스 타입)마다 고정**이다.  
`t4g.small`을 고르면 무조건 2 vCPU + 2 GiB. CPU만 늘리거나 RAM만 따로 고를 수 없다.  
바꾸고 싶으면 **다른 타입**을 고른다 (`t4g.medium` = 2 vCPU + 4 GiB 등).

**vCPU란?**  
Virtual CPU. 인스턴스에 할당된 **가상 CPU 코어 개수**.  
`2` = 코어 2개짜리, `4` = 코어 4개짜리라고 보면 된다.  
숫자가 클수록 동시에 일을 더 많이 처리할 수 있는 연산 능력(대략)이 커진다.  
(물리 CPU 1코어와 1:1은 아니고, AWS가 나눠 준 가상 단위.)

| | vCPU / RAM |
|---|---|
| **EC2** (`t4g.small` 등) | 타입별 **고정 세트** |
| **ECS on EC2** | 서버(EC2) 스펙은 고정. Task는 그 안에서 cpu/memory **예약량**만 지정 |
| **ECS Fargate** | 인스턴스 이름 없음. 허용된 **조합** 중에서 vCPU+메모리 선택 (완전 자유는 아님) |

| 스펙 | vCPU | RAM | 시간당 | 월 (대략) | 원화 (대략) | 짧은 설명 |
|---|---|---|---|---|---|---|
| **t4g.micro** | 2 | 1 GiB | $0.0104 | ~$7.6 | ~₩10,000 | 아주 작은 웹/사이드카. Graviton |
| **t4g.small** | 2 | 2 GiB | $0.0208 | ~$15 | ~₩20,000 | 가장 흔함. 가벼운 API/웹 |
| **t4g.medium** | 2 | 4 GiB | $0.0416 | ~$30 | ~₩41,000 | 트래픽 조금 있는 앱 |
| **t4g.large** | 2 | 8 GiB | $0.0832 | ~$61 | ~₩82,000 | 메모리 더 필요한 버스트형 |
| **t3.small** | 2 | 2 GiB | $0.026 | ~$19 | ~₩26,000 | Intel 버전. t4g.small보다 비쌈 |
| **t3.medium** | 2 | 4 GiB | $0.052 | ~$38 | ~₩51,000 | Intel. 레거시/x86 필요할 때 |
| **m6g.large** | 2 | 8 GiB | $0.094 | ~$69 | ~₩93,000 | 범용 기본. 상시 CPU 쓸 때 t보다 안정 |
| **m6g.xlarge** | 4 | 16 GiB | $0.188 | ~$137 | ~₩185,000 | 중형 앱 서버 |
| **c6g.large** | 2 | 4 GiB | $0.077 | ~$56 | ~₩76,000 | CPU 작업용 (배치·연산) |
| **r6g.large** | 2 | 16 GiB | $0.122 | ~$89 | ~₩120,000 | 메모리 많이 쓰는 앱/캐시성 워크로드 |
| **r6g.xlarge** | 4 | 32 GiB | $0.244 | ~$178 | ~₩240,000 | 큰 메모리 서버 |

**고르는 감각**

| 상황 | 추천 |
|---|---|
| 개발 / 트래픽 적음 | `t4g.small` ~ `t4g.medium` |
| 상시 부하 있는 웹 | `m6g.large` 이상 (t는 크레딧 고갈 주의) |
| CPU 많이 씀 | `c6g.*` |
| 메모리 많이 씀 | `r6g.*` |
| x86만 됨 (일부 바이너리) | `t3` / `m5` 등 (g 없는 쪽) |

---

## RDS (자주 쓰는 스펙)

PostgreSQL · **Single-AZ** · 인스턴스 요금만 (스토리지·백업·Multi-AZ 별도).

| 스펙 | vCPU | RAM | 시간당 | 월 (대략) | 원화 (대략) | 짧은 설명 |
|---|---|---|---|---|---|---|
| **db.t4g.micro** | 2 | 1 GiB | $0.025 | ~$18 | ~₩25,000 | 개발·테스트 DB |
| **db.t4g.small** | 2 | 2 GiB | $0.051 | ~$37 | ~₩50,000 | 작은 프로덕션 / 스테이징 |
| **db.t4g.medium** | 2 | 4 GiB | $0.102 | ~$74 | ~₩101,000 | 소규모 서비스 DB |
| **db.t3.medium** | 2 | 4 GiB | $0.112 | ~$82 | ~₩110,000 | Intel. t4g.medium보다 약간 비쌈 |
| **db.r6g.large** | 2 | 16 GiB | $0.270 | ~$197 | ~₩266,000 | 본격 프로덕션. 메모리형 |
| **db.r6g.xlarge** | 4 | 32 GiB | $0.540 | ~$394 | ~₩532,000 | 트래픽·커넥션 많은 DB |

참고:

- **Multi-AZ**면 인스턴스 요금 대략 **×2** (대기 인스턴스)
- t 계열 RDS는 Unlimited 모드 → 평균 CPU가 baseline 넘으면 **CPU 크레딧 추가 과금**
- 스토리지(gp3 등)는 GB-월로 **별도**

---

## ECS Fargate (인스턴스 타입 없음)

태스크에 **vCPU + 메모리**만 지정. (서울 Linux x86)

| 항목 | 단가 | 원화 (대략) |
|---|---|---|
| vCPU | $0.04656 / vCPU-시간 | ~₩63 / vCPU-시간 |
| Memory | $0.00511 / GB-시간 | ~₩7 / GB-시간 |

예: **1 vCPU + 2 GB** 상시 ≈ **$41.5/월 (~₩56,000)**  
ARM이면 더 저렴 (`$0.03725` / `$0.00409`).

| EC2 vs Fargate | |
|---|---|
| EC2 | `t4g.small`처럼 스펙 고름. 싸게 굳히기 쉬움. 서버 관리 필요 |
| Fargate | 스펙 이름 없음. 운영 편함. 같은 용량이면 보통 EC2보다 비쌈 |

---

## 한줄 비교 (같은 “small급” 감각)

| | EC2 `t4g.small` | RDS `db.t4g.small` | Fargate 1vCPU+2GB |
|---|---|---|---|
| 월 대략 | ~$15 (₩20,000) | ~$37 (₩50,000) | ~$42 (₩56,000) |
| 비고 | 앱 서버 | DB 전용 (+스토리지 별도) | 관리형 컨테이너 |

---

## EC2 / ECS / RDS 말고 스펙을 고르는 리소스

| 리소스 | 고르는 것 | 예 | 비고 |
|---|---|---|---|
| **ElastiCache** | 노드 타입 | `cache.t4g.micro`, `cache.r6g.large` | Redis/Memcached. RDS랑 비슷한 네이밍 |
| **OpenSearch** | 인스턴스 타입 | `t3.small.search`, `r6g.large.search` | 검색/로그 클러스터 |
| **MS Kinesis / MSK** | 브로커 타입 | `kafka.t3.small`, `kafka.m5.large` | 카프카 브로커 |
| **Amazon MQ** | 인스턴스 타입 | `mq.t3.micro`, `mq.m5.large` | ActiveMQ/RabbitMQ |
| **Redshift** | 노드 타입 | `ra3.xlplus`, `dc2.large` | 데이터 웨어하우스 |
| **EMR** | 마스터/코어 노드 = EC2 타입 | `m6g.xlarge` 등 | 빅데이터. 사실상 EC2 스펙 |
| **SageMaker** | 노트북/트레이닝 인스턴스 | `ml.t3.medium`, `ml.g5.xlarge` | ML |
| **AWS Batch** | 컴퓨트 환경의 EC2/Fargate | EC2 타입 또는 Fargate vCPU·메모리 | |
| **DocumentDB / Neptune** | 인스턴스 클래스 | `db.t4g.medium`, `db.r6g.large` | RDS와 비슷한 `db.*` |
| **MemoryDB** | 노드 타입 | `db.t4g.small`, `db.r6g.large` | Redis 호환 내구성 DB |

### 스펙 “타입”이 아니라 다른 단위로 고르는 것

| 리소스 | 고르는 것 | 예 |
|---|---|---|
| **Lambda** | 메모리(MB) → CPU는 메모리에 비례 | 128MB ~ 10GB |
| **Fargate** | vCPU + 메모리 **조합** | 1 vCPU + 2GB |
| **Aurora Serverless v2** | ACU (용량 단위) | 0.5 ~ N ACU |
| **DynamoDB** | On-Demand 또는 RCU/WCU | 인스턴스 타입 없음 |
| **ALB / NAT / S3** | 인스턴스 스펙 없음 | 사용량·시간 과금 |

자주 쓰는 웹 스택만 보면: **EC2/ECS + RDS + (있으면) ElastiCache** 에서 스펙을 제일 많이 고르게 됨.
