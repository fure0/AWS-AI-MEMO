# Resource 모니터링 · 로그 보는 방법 (AWS 콘솔)

> 목적: **제대로 돌아가나?** 를 콘솔에서 확인하는 입구 + 화면이 의미하는 것.  
> 공통 도구는 **CloudWatch**(메트릭·로그·알람). 리소스마다 “어디 메뉴로 들어가냐”가 다름.

```text
빠른 감각
  트래픽·에러율·지연   → 각 서비스 Monitoring 탭 또는 CloudWatch Metrics
  앱/컨테이너 로그 문구 → CloudWatch Logs (Log Group)
  누가 API를 호출했나   → CloudTrail
  배포가 됐나           → CodePipeline / ECS 배포 상태
```

---

## 공통: CloudWatch로 가는 법

| 보고 싶은 것 | 콘솔 경로 | 의미 |
|---|---|---|
| 숫자 그래프 (CPU, 요청 수) | CloudWatch → **Metrics** | 시간에 따른 수치 |
| 텍스트 로그 | CloudWatch → **Log groups** | 줄 단위 기록 |
| 임계값 알림 | CloudWatch → **Alarms** | 조건 넘으면 알림 |
| 한눈에 보드 | CloudWatch → **Dashboards** | 그래프 모음 |

---

## CloudFront

**어디:** CloudFront → Distributions → 해당 Distribution → **Monitoring** / **Reports**

| 보는 것 | 의미 |
|---|---|
| **Requests** | 사용자 요청이 들어왔는지 (트래픽 있음?) |
| **Bytes downloaded** | 사용자에게 보낸 데이터량 |
| **Error rate (4xx/5xx)** | 실패 비율. 갑자기 오르면 오리진·권한·캐시 문제 의심 |
| **Cache hit rate** | 엣지에서 바로 응답한 비율. 높을수록 오리진 부담↓ |
| **Origin latency** | 오리진(S3/ALB) 응답이 느린지 |

로그를 켜 둔 경우: S3 또는 CloudWatch Logs로 접근 로그 저장 (Distribution 설정에서 활성화).

**정상 감각:** 요청은 있고, 5xx가 거의 없고, hit rate가 크게 무너지지 않음.

---

## WAF

**어디:** WAF & Shield → Web ACLs → 해당 ACL → **Overview** / **Sampled requests**

| 보는 것 | 의미 |
|---|---|
| **Allowed / Blocked / Counted** | 통과·차단·집계만 한 요청 수 |
| **Sampled requests** | 최근에 매칭된 요청 샘플 (어떤 규칙에 걸렸는지) |
| **Rule metrics** | 규칙별 히트 수 |

CloudWatch에도 `AllowedRequests`, `BlockedRequests` 메트릭이 생김.

**정상 감각:** 서비스는 되는데 Block이 비정상적으로 폭증하면 오탐(정상 유저 차단) 가능. Count 모드로 먼저 보는 경우가 많음.

---

## Route53

**어디:** Route53 → Hosted zones → 레코드 확인  
헬스체크 쓰면: Route53 → **Health checks**

| 보는 것 | 의미 |
|---|---|
| **레코드 값** | 도메인이 어디로 가는지 (ALB/CloudFront Alias 맞는지) |
| **Health check status** | 대상이 살아 있다고 DNS가 판단하는지 (Healthy/Unhealthy) |
| CloudWatch (헬스체크) | `HealthCheckStatus`, `ConnectionTime` 등 |

쿼리 로그는 기본 OFF. 켜면 CloudWatch Logs로 DNS 조회 기록.

**정상 감각:** Alias가 의도한 대상이고, 헬스체크를 쓰면 Healthy.

---

## ALB

**어디:** EC2 → Load Balancers → 해당 ALB → **Monitoring**  
타겟: EC2 → Target Groups → **Targets** / **Monitoring**

| 보는 것 | 의미 |
|---|---|
| **RequestCount** | ALB로 들어온 요청 수 |
| **HTTPCode_Target_2XX/4XX/5XX** | 백엔드(ECS/EC2)가 준 응답 코드 |
| **HTTPCode_ELB_4XX/5XX** | ALB 자체에서 난 에러 (대상 없음, 타임아웃 등) |
| **TargetResponseTime** | 백엔드 응답 지연 |
| **HealthyHostCount / UnHealthyHostCount** | 타겟 몇 대가 건강한지 |
| Target Group → **Targets** | 각 인스턴스/태스크 IP의 healthy 여부 |

액세스 로그: ALB 속성에서 S3로 켤 수 있음.

**정상 감각:** HealthyHost ≥ 1, ELB 5xx·Target 5xx가 낮음, Unhealthy가 계속 0에 가까움.

```text
Unhealthy만 보면
  SG 막힘 / 헬스체크 경로 404 / 앱이 안 뜸
```

---

## ECS

**어디:** ECS → Clusters → Service → **Health & metrics** / **Tasks** / **Logs**  
또는 CloudWatch Logs → `/ecs/...` Log Group

| 보는 것 | 의미 |
|---|---|
| **Running / Desired count** | 원하는 개수만큼 태스크가 떠 있는지 |
| **Tasks 상태** | `RUNNING`이면 정상. `PENDING`/`STOPPED`면 원인 이벤트 확인 |
| **Events** 탭 | “스케일 실패”, “이미지 pull 실패” 같은 서비스 메시지 |
| **CPU / Memory utilization** | 서비스가 스펙 대비 얼마나 쓰는지 |
| **Logs** (awslogs) | 컨테이너가 찍은 앱 로그 |

태스크가 바로 죽을 때: Task → **Stopped reason** / CloudWatch 로그 + ECR 권한·포트·헬스체크 확인.

**정상 감각:** Running = Desired, 최근 Events에 에러 없음, 로그에 기동 메시지 있음.

---

## EC2

**어디:** EC2 → Instances → 해당 인스턴스 → **Monitoring**  
상세 모니터링·로그: CloudWatch Agent 또는 시스템 로그

| 보는 것 | 의미 |
|---|---|
| **Instance state** | `running`이어야 함. `stopped`/`terminated`면 꺼진 것 |
| **Status checks** | 시스템·인스턴스 상태 검사 통과 여부 (2/2 checks passed) |
| **CPUUtilization** | CPU 사용률 |
| **NetworkIn/Out** | 네트워크 트래픽 |
| **StatusCheckFailed** | 하드웨어/OS 수준 문제 시 1 |

앱 로그는 OS 안 파일 또는 CloudWatch Agent → Log Group.

**정상 감각:** running + status check 통과 + CPU가 100%에 장시간 고정되지 않음.

---

## AWS Backup

**어디:** AWS Backup → **Backup jobs** / **Restore jobs** / **Backup vaults**

| 보는 것 | 의미 |
|---|---|
| **Backup jobs** | 백업이 `Completed`인지 `Failed`인지 |
| **Last backup** (리소스별) | 최근에 백업이 돌았는지 |
| **Recovery points** (Vault) | 복원 가능한 백업본 목록 |
| **Restore jobs** | 복원 시도 성공/실패 |

**정상 감각:** 스케줄대로 Completed, Failed가 반복되지 않음.

---

## CodePipeline

**어디:** CodePipeline → Pipelines → 해당 파이프라인

| 보는 것 | 의미 |
|---|---|
| **스테이지 초록/빨강** | Source → Build → Deploy 각 단계 성공/실패 |
| **실행 이력 (Execution history)** | 최근 배포가 언제·성공했는지 |
| 실패 스테이지 클릭 | 실패 Action → **Details** / CodeBuild 로그 링크 |

Build 실패 시: CodeBuild → Build history → **Phase details** / CloudWatch Logs.

**정상 감각:** 최근 실행이 Succeeded, Deploy까지 초록.

---

## CloudTrail

**어디:** CloudTrail → **Event history**  
Trail을 만들었으면: S3 버킷 또는 CloudWatch Logs (Trail 설정 목적지)

| 보는 것 | 의미 |
|---|---|
| **Event history** | 최근 90일 **누가 어떤 API를 호출했는지** (콘솔·CLI·SDK) |
| **User name / Event name** | 예: `UpdateService`, `ConsoleLogin`, `DeleteBucket` |
| **Resource type** | 대상이 ECS인지 S3인지 |
| **Error code** | 실패한 API 호출이면 AccessDenied 등 |

모니터링(실시간 감시)보다는 **감사·사고 추적**에 가깝다.  
“서비스 살아 있나?”는 ALB/ECS Monitoring이 더 적합.

**정상 감각:** 이상한 시간대·모르는 IAM 사용자의 Delete/Update가 없는지.

---

## (참고) 같이 자주 보는 것

### S3
**어디:** S3 → 버킷 → **Metrics**  
| 의미 | 객체 수·용량·요청 수. 앱 로그는 버킷 안 파일 또는 액세스 로그 버킷 |

### ECR
**어디:** ECR → Repositories → 이미지 태그 / **Metrics**(스캔)  
| 의미 | 이미지가 push 됐는지, 취약점 스캔 결과 |

### CloudWatch Alarm 예시 (웹 스택)
| 알람 후보 | 의미 |
|---|---|
| ALB `HTTPCode_Target_5XX` ↑ | 앱/서버 에러 |
| ALB `UnHealthyHostCount` ≥ 1 | 타겟 죽음 |
| ECS `RunningTaskCount` < Desired | 태스크 부족 |
| CloudFront `5xxErrorRate` ↑ | CDN/오리진 장애 |

---

## 장애 났을 때 보는 순서 (웹 앱 기준)

```text
1. ALB Target Healthy?     → 서버가 안 받음
2. ECS Running = Desired?  → 컨테이너가 안 뜸
3. ECS/CloudWatch Logs     → 앱 에러 메시지
4. CloudFront 5xx / WAF Block → 엣지·차단 이슈
5. CodePipeline 최근 배포   → 방금 배포가 원인인지
6. CloudTrail              → 누가 설정을 바꿨는지
```

---

## 한줄 요약

| 리소스 | 콘솔에서 첫로 볼 곳 | 한 줄 의미 |
|---|---|---|
| CloudFront | Distribution → Monitoring | 요청·에러·캐시 |
| WAF | Web ACL → Overview / Sampled | 막았나·뭘 막았나 |
| Route53 | Hosted zone / Health checks | 어디로 가나·대상 살아 있나 |
| ALB | LB Monitoring + Target health | 요청·5xx·타겟 건강 |
| ECS | Service Tasks / Logs | 개수·상태·앱 로그 |
| EC2 | Instance Monitoring + Status | 켜짐·CPU·헬스 |
| AWS Backup | Backup jobs | 백업 성공/실패 |
| CodePipeline | Pipeline 실행 화면 | 배포 파이프 성공/실패 |
| CloudTrail | Event history | 누가 무슨 API 호출 |
