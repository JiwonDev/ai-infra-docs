# 환경 초기화 (저장소 + GCP)

ch9 완료 후, 다음 run을 시작하기 전에 이전 run에서 생성된 리소스를 모두 정리한다.

## 1. 저장소 초기화 (권장)

notiflex-platform 저장소를 초기 상태로 되돌린다. 처음부터 만들어가는 경험을 위해 초기화를 권장한다.

```bash
cd notiflex-platform
git rm -rf .
git commit -m "reset: 저장소 초기화"
git push
```

> ⚠️ `gh repo delete`는 `delete_repo` scope가 필요하므로 사용하지 않는다. 위 방식으로 초기화하면 히스토리는 남지만 작업 상태는 깨끗해진다.

## 2. GCP 리소스 정리

### 정리 대상

| 순서 | 리소스 | 확인 명령 | 삭제 명령 |
|------|--------|----------|----------|
| 1 | GKE Cluster (operation 확인) | `gcloud container operations list --zone=asia-northeast3-a --filter="status=RUNNING"` | 진행 중인 operation 있으면 대기 후 삭제 |
| 1 | GKE Cluster | `gcloud container clusters list` | `gcloud container clusters delete notiflex-cluster --zone=asia-northeast3-a --quiet --async` |
| 2 | Artifact Registry | `gcloud artifacts repositories list --location=asia-northeast3` | `gcloud artifacts repositories delete notiflex --location=asia-northeast3 --quiet` |
| 3 | Secret Manager | `gcloud secrets list` | `gcloud secrets delete <name> --quiet` |
| 4 | IAM bindings (WI) | `gcloud iam service-accounts get-iam-policy <SA>` | `gcloud iam service-accounts remove-iam-policy-binding <SA> --member=... --role=...` |
| 5 | IAM bindings (프로젝트) | `gcloud projects get-iam-policy <PROJECT> --format=json` | `gcloud projects remove-iam-policy-binding <PROJECT> --member=... --role=...` |
| 6 | Service Accounts | `gcloud iam service-accounts list` | `gcloud iam service-accounts delete <SA> --quiet` |

### 정리 순서가 중요한 이유

- **클러스터를 먼저 삭제**: 클러스터가 SA/Secret을 참조하고 있으므로, 클러스터 삭제를 먼저 시작(async)한 뒤 나머지를 정리한다.
- **진행 중인 operation 확인**: 클러스터 삭제 시 `RESIZE_CLUSTER` 등 다른 operation이 진행 중이면 `"Cluster is running incompatible operation"` 에러가 발생한다. `gcloud container operations list --zone=... --filter="status=RUNNING"`으로 확인하고, 완료를 기다린 후 삭제한다.
- **WI binding을 SA 삭제 전에 제거**: SA를 삭제하면 binding이 orphan으로 남을 수 있다.
- **IAM binding을 SA 삭제 전에 제거**: 같은 이유. binding이 존재하지 않으면 "Policy binding not found" 에러가 발생하지만 무시하고 진행해도 된다.

### run에서 생성되는 리소스 목록

| 챕터 | 리소스 | 이름 패턴 |
|------|--------|----------|
| ch2 | GKE Cluster | `notiflex-cluster` |
| ch2 | Artifact Registry | `notiflex` |
| ch3 | Service Account | `github-ci@` |
| ch3 | IAM binding | `github-ci@ → roles/artifactregistry.writer` |
| ch6 | Service Account | `notiflex-sa@` |
| ch6 | Secret Manager | `valkey-password` (실제 이름은 `gcloud secrets list`로 확인) |
| ch6 | IAM binding (WI) | `notiflex-sa@ → roles/iam.workloadIdentityUser` |
| ch6 | IAM binding (프로젝트) | `notiflex-sa@ → roles/secretmanager.secretAccessor` |

## 3. 고아(Orphan) 리소스 정리

클러스터가 삭제되어도 일부 GCP 리소스는 자동으로 삭제되지 않는다. 특히 Gateway API와 PVC에서 생성된 리소스가 남는다.

### Gateway API 고아 리소스

Gateway API(ch5.2)에서 생성한 리소스는 클러스터 삭제 시 남을 수 있다:

```bash
PROJECT=$(gcloud config get-value project)   # ch2.3에서 설정한 기본 프로젝트
REGION=asia-northeast3

# 1. Forwarding Rules
gcloud compute forwarding-rules list --project=$PROJECT --format="table(name,region)"
# 있으면 삭제 (regional):
# gcloud compute forwarding-rules delete <NAME> --region=$REGION --quiet

# 2. Target HTTP Proxies
gcloud compute target-http-proxies list --project=$PROJECT --format="table(name,region)"
# regional이면:
# gcloud compute target-http-proxies delete <NAME> --region=$REGION --quiet

# 3. URL Maps
gcloud compute url-maps list --project=$PROJECT --format="table(name,region)"
# regional이면:
# gcloud compute url-maps delete <NAME> --region=$REGION --quiet

# 4. Addresses
gcloud compute addresses list --project=$PROJECT --format="table(name,region)"
# regional이면:
# gcloud compute addresses delete <NAME> --region=$REGION --quiet

# 5. Backend Services
gcloud compute backend-services list --project=$PROJECT --format="table(name,region)"
# regional이면:
# gcloud compute backend-services delete <NAME> --region=$REGION --quiet

# 6. Health Checks (⚠️ regional vs global 구분 필수)
gcloud compute health-checks list --project=$PROJECT --format="table(name,type,region)"
# regional이면 --region 필수:
# gcloud compute health-checks delete <NAME> --region=$REGION --quiet
# global이면 --region 없이:
# gcloud compute health-checks delete <NAME> --quiet
```

> ⚠️ Health Check 삭제 시 주의: `gcloud compute health-checks list`에서 `REGION` 컬럼이 표시되면 regional이다. regional Health Check을 `--region` 없이 삭제하면 "not found" 에러가 발생한다.

### PVC 고아 리소스 (Persistent Disk)

Kafka(ch8.1), Valkey(ch6.1) 등 StatefulSet의 PVC에서 생성된 Persistent Disk는 클러스터 삭제 후에도 남는다:

```bash
# 확인
gcloud compute disks list --project=$PROJECT --format="table(name,zone,status)" \
  --filter="zone:asia-northeast3"

# 삭제 (zone 필수)
# gcloud compute disks delete <NAME> --zone=asia-northeast3-a --quiet
```

> ⚠️ 남은 디스크는 과금된다. `gke-sysnet4admin_book_gitaiops-cluster-*` 패턴 이름의 디스크가 남아있으면 삭제한다.

## 4. 정리 확인

모든 정리 후 아래 명령으로 잔존 리소스를 확인한다:

```bash
PROJECT=$(gcloud config get-value project)

# 저장소
ls notiflex-platform/   # .git만 남아있어야 함

# 클러스터
gcloud container clusters list --project=$PROJECT

# Service Accounts (notiflex/github-ci 관련만)
gcloud iam service-accounts list --project=$PROJECT | grep -E 'notiflex|github-ci'

# Secrets
gcloud secrets list --project=$PROJECT | grep notiflex

# Artifact Registry
gcloud artifacts repositories list --location=asia-northeast3 --project=$PROJECT | grep notiflex
```

모든 결과가 비어있으면 정리 완료.

```bash
PROJECT=$(gcloud config get-value project)

# Gateway API 고아 리소스
gcloud compute forwarding-rules list --project=$PROJECT --format="table(name)"
gcloud compute target-http-proxies list --project=$PROJECT --format="table(name)"
gcloud compute url-maps list --project=$PROJECT --format="table(name)"
gcloud compute addresses list --project=$PROJECT --format="table(name)"
gcloud compute backend-services list --project=$PROJECT --format="table(name)"
gcloud compute health-checks list --project=$PROJECT --format="table(name)"

# Persistent Disks (PVC 잔존)
gcloud compute disks list --project=$PROJECT --filter="zone:asia-northeast3" --format="table(name)"
```

## 주의사항

- `cicd-repo`, `notiflex-repo`, `student-demo` 등 다른 프로젝트의 Artifact Registry는 삭제하지 않는다.
- `Compute Engine default service account`는 삭제하지 않는다.
- 클러스터 삭제는 3~5분 소요. `--async`로 시작하고 나머지 리소스를 병렬로 정리한다.
