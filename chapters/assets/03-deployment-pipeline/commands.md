# 03장 실행 명령

`homeserver-v2`는 kubeconfig 파일 이름이고 파일 안의 실제 context 이름은 `default`입니다. 먼저 `KUBECONFIG`를 해당 파일로 지정하고, 모든 `kubectl` 명령은 `--context default`를 사용합니다. `<...>` 값은 실제 홈서버 값으로 바꿉니다. token과 private key는 shell 환경에만 두고 Git에 저장하지 않습니다.

## 실행 전 목표 상태

이 장에서는 private GitHub `JiwonDev/infra-gitops`의 다음 파일로 Notiflex 배포·promotion·rollback을 구성하고 검증합니다.

| 경로 | 역할 |
| --- | --- |
| `clusters/homeserver/apps/notiflex.yaml` | Notiflex child Application |
| `apps/notiflex/base/deployment.yaml` | Deployment와 probe |
| `apps/notiflex/base/service.yaml` | cluster 내부 Service |
| `apps/notiflex/base/pdb.yaml` | `minAvailable: 1` PDB |
| `apps/notiflex/overlays/homeserver/kustomization.yaml` | Harbor digest와 홈서버 patch |
| `apps/notiflex/overlays/homeserver/version.env` | `/health`에 표시할 source SHA |

child Application은 `notiflex-home`, Deployment는 `notiflex-api` 이름을 사용합니다. 명령을 실행하기 전에 경로와 이름이 실제 manifest와 일치하는지 확인합니다.

## Argo CD bootstrap

### 클러스터 확인

```bash
export KUBECONFIG="${HOME}/.kube/homeserver-v2.yaml"
kubectl config get-contexts
test "$(kubectl config current-context)" = 'default'
kubectl --context default cluster-info
kubectl --context default get nodes -o wide \
  -L node-role.kubernetes.io/control-plane,node-role.kubernetes.io/etcd
```

`k3s-a`, `k3s-b`, `k3s-c`가 모두 `Ready`인 server/control-plane/etcd/worker이고 win notebook Ubuntu VM이 `Ready`인 worker인지 확인합니다.

### 설치

```bash
set -euo pipefail
ARGO_CD_VERSION=v3.4.5

kubectl --context default create namespace argocd \
  --dry-run=client -o yaml |
  kubectl --context default apply --server-side -f -

kubectl --context default apply -n argocd \
  -f "https://raw.githubusercontent.com/argoproj/argo-cd/${ARGO_CD_VERSION}/manifests/install.yaml" \
  --server-side=true --force-conflicts=true

kubectl --context default wait \
  --for=condition=Available deployment/argocd-server \
  -n argocd --timeout=300s
kubectl --context default get pods -n argocd
```

### Private GitHub read-only credential

```bash
set -euo pipefail
read -rp 'Read-only deploy key file: ' INFRA_GITOPS_DEPLOY_KEY_FILE
test -f "${INFRA_GITOPS_DEPLOY_KEY_FILE}"

kubectl --context default -n argocd create secret generic repo-infra-gitops \
  --from-literal=type=git \
  --from-literal=url=git@github.com:JiwonDev/infra-gitops.git \
  --from-file=sshPrivateKey="${INFRA_GITOPS_DEPLOY_KEY_FILE}" \
  --dry-run=client -o yaml |
  kubectl --context default apply -f -

kubectl --context default -n argocd label secret repo-infra-gitops \
  argocd.argoproj.io/secret-type=repository --overwrite
unset INFRA_GITOPS_DEPLOY_KEY_FILE
```

`homeserver-root`와 child Application manifest의 `repoURL`도 `git@github.com:JiwonDev/infra-gitops.git`으로 맞춥니다.

### Root app과 관리 UI

```bash
set -euo pipefail
cd D:/pycharm/infra-gitops

kubectl --context default apply \
  -f clusters/homeserver/bootstrap/root-app.yaml
kubectl --context default get applications -n argocd
```

관리 PC의 별도 terminal에서 UI를 엽니다.

```bash
kubectl --context default port-forward \
  --address 127.0.0.1 svc/argocd-server -n argocd 8443:443
```

`https://127.0.0.1:8443`에서 `homeserver-root`와 `notiflex-home`이 `Synced / Healthy`인지 확인합니다.

## Registry credential

### 모든 K3s node의 Harbor CA

모든 K3s server와 agent에 같은 Harbor CA와 `registries.yaml`을 적용합니다.

```bash
sudo install -m 0644 ./harbor-ca.crt /etc/rancher/k3s/harbor-ca.crt
sudoedit /etc/rancher/k3s/registries.yaml
```

```yaml
configs:
  "<HARBOR_REGISTRY>":
    tls:
      ca_file: /etc/rancher/k3s/harbor-ca.crt
```

`k3s-a`, `k3s-b`, `k3s-c` 같은 server는 quorum을 유지하도록 한 대씩 `k3s` 서비스를 재시작합니다.

```bash
sudo systemctl restart k3s
sudo systemctl is-active k3s
```

win notebook Ubuntu VM 같은 agent에서는 `k3s-agent` 서비스를 재시작합니다.

```bash
sudo systemctl restart k3s-agent
sudo systemctl is-active k3s-agent
```

```bash
kubectl --context default get nodes
kubectl --context default get pods -n kube-system -o wide
```

### Runner push 인증

runner의 OS, Docker Engine, BuildKit이 Harbor CA를 신뢰하게 합니다. 계정과 비밀번호는 GitHub Actions repository secret으로 전달합니다.

```bash
set -euo pipefail
: "${HARBOR_REGISTRY:?set HARBOR_REGISTRY}"
: "${HARBOR_PUSH_USER:?set HARBOR_PUSH_USER}"
: "${HARBOR_PUSH_PASSWORD:?set HARBOR_PUSH_PASSWORD}"

sudo install -m 0644 ./harbor-ca.crt \
  /usr/local/share/ca-certificates/harbor-ca.crt
sudo update-ca-certificates

sudo install -d -m 0755 \
  "/etc/docker/certs.d/${HARBOR_REGISTRY}" \
  /etc/buildkit/certs
sudo install -m 0644 ./harbor-ca.crt \
  "/etc/docker/certs.d/${HARBOR_REGISTRY}/ca.crt"
sudo install -m 0644 ./harbor-ca.crt \
  /etc/buildkit/certs/harbor-ca.crt

sudo tee /etc/buildkit/buildkitd.toml >/dev/null <<EOF
[registry."${HARBOR_REGISTRY}"]
  ca = ["/etc/buildkit/certs/harbor-ca.crt"]
EOF

docker buildx rm homeserver-builder 2>/dev/null || true
docker buildx create \
  --name homeserver-builder \
  --driver docker-container \
  --buildkitd-config /etc/buildkit/buildkitd.toml \
  --use
docker buildx inspect --bootstrap

printf '%s' "${HARBOR_PUSH_PASSWORD}" |
  docker login "${HARBOR_REGISTRY}" \
    --username "${HARBOR_PUSH_USER}" --password-stdin
unset HARBOR_PUSH_PASSWORD
```

### OCI pull Secret

Harbor에는 pull 전용 계정을 따로 둡니다.

```bash
set -euo pipefail
: "${HARBOR_REGISTRY:?set HARBOR_REGISTRY}"
read -rp 'Harbor pull user: ' HARBOR_PULL_USER
read -rsp 'Harbor pull password: ' HARBOR_PULL_PASSWORD && printf '\n'

kubectl --context default create namespace notiflex \
  --dry-run=client -o yaml |
  kubectl --context default apply -f -

kubectl --context default -n notiflex \
  create secret docker-registry harbor-pull-secret \
  --docker-server="${HARBOR_REGISTRY}" \
  --docker-username="${HARBOR_PULL_USER}" \
  --docker-password="${HARBOR_PULL_PASSWORD}" \
  --dry-run=client -o yaml |
  kubectl --context default apply -f -

unset HARBOR_PULL_USER HARBOR_PULL_PASSWORD
kubectl --context default -n notiflex get secret harbor-pull-secret \
  -o jsonpath='{.type}{"\n"}'
```

출력은 `kubernetes.io/dockerconfigjson`이어야 합니다. Secret 원문은 출력하거나 `infra-gitops`에 commit하지 않습니다.

## 수동 promotion과 rollback

### Digest 확인과 promotion

```bash
set -euo pipefail
: "${HARBOR_IMAGE:?set HARBOR_IMAGE}"
IMAGE="${HARBOR_IMAGE}"
TAG='sha-<12자리_COMMIT>'

docker buildx imagetools inspect "${IMAGE}:${TAG}"
export DIGEST='sha256:<확인한_manifest_digest>'

cd D:/pycharm/infra-gitops/apps/notiflex/overlays/homeserver
kustomize edit set image "notiflex-api=${IMAGE}@${DIGEST}"
printf 'APP_VERSION=%s\n' "${TAG}" > version.env

kustomize build . > /tmp/notiflex.yaml
kubeconform -strict -summary /tmp/notiflex.yaml
git diff --check
git diff

git add kustomization.yaml version.env
git commit -m "deploy: notiflex ${TAG} ${DIGEST:0:19}"
git push origin main
```

### Rolling Update와 `/health`

상태 확인은 별도 terminal에서 실행합니다.

```bash
kubectl --context default get application notiflex-home -n argocd -w
```

```bash
set -euo pipefail
kubectl --context default rollout status \
  deployment/notiflex-api -n notiflex --timeout=300s
kubectl --context default get pods -n notiflex -o wide
kubectl --context default port-forward \
  svc/notiflex-api -n notiflex 8080:80
```

port-forward가 열린 상태에서 다른 terminal로 version을 확인합니다.

```bash
EXPECTED_VERSION='sha-<12자리_COMMIT>'
curl -fsS http://127.0.0.1:8080/health |
  jq -e --arg version "${EXPECTED_VERSION}" \
    '.status == "ok" and .version == $version'
```

### Rollback

```bash
set -euo pipefail
PRE_REVERT_IMAGE="$(kubectl --context default get deployment \
  notiflex-api -n notiflex \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"

cd D:/pycharm/infra-gitops
git log --oneline -- apps/notiflex
git revert --no-edit '<PROMOTION_COMMIT>'
REVERT_REVISION="$(git rev-parse HEAD)"
git push origin main

for attempt in {1..60}; do
  APP_REVISION="$(kubectl --context default get application \
    notiflex-home -n argocd -o jsonpath='{.status.sync.revision}')"
  APP_SYNC="$(kubectl --context default get application \
    notiflex-home -n argocd -o jsonpath='{.status.sync.status}')"
  DEPLOYED_IMAGE="$(kubectl --context default get deployment \
    notiflex-api -n notiflex \
    -o jsonpath='{.spec.template.spec.containers[0].image}')"

  if [ "${APP_REVISION}" = "${REVERT_REVISION}" ] && \
     [ "${APP_SYNC}" = 'Synced' ] && \
     [ "${DEPLOYED_IMAGE}" != "${PRE_REVERT_IMAGE}" ]; then
    break
  fi
  sleep 5
done

test "${APP_REVISION}" = "${REVERT_REVISION}"
test "${APP_SYNC}" = 'Synced'
test "${DEPLOYED_IMAGE}" != "${PRE_REVERT_IMAGE}"
kubectl --context default rollout status \
  deployment/notiflex-api -n notiflex --timeout=300s
```

기존 port-forward는 Rolling Update 중 선택된 Pod가 교체되면 종료될 수 있습니다. 기존 process를 종료하고 rollout 완료 뒤 새 terminal에서 다시 엽니다.

```bash
kubectl --context default port-forward \
  svc/notiflex-api -n notiflex 8080:80
```

다른 terminal에서 이전 version을 확인합니다.

```bash
set -euo pipefail
EXPECTED_ROLLBACK_VERSION='sha-<이전_12자리_COMMIT>'
curl -fsS http://127.0.0.1:8080/health |
  jq -e --arg version "${EXPECTED_ROLLBACK_VERSION}" \
    '.status == "ok" and .version == $version'
```

revert 뒤 Application은 `Synced / Healthy`, `/health` version은 이전 SHA여야 합니다.

## Gitea local mirror 확인

Gitea는 private GitHub 원본을 읽는 pull mirror로만 구성합니다. mirror용 GitHub credential은 read-only 권한만 사용합니다. 인터넷 연결이 있을 때 다음 명령으로 양쪽 `main`을 비교합니다.

```bash
set -euo pipefail
: "${GITEA_SOURCE_URL:?set GITEA_SOURCE_URL}"
GITHUB_SHA="$(git ls-remote \
  git@github.com:JiwonDev/notiflex-platform.git refs/heads/main | cut -f1)"
GITEA_SHA="$(git ls-remote \
  "${GITEA_SOURCE_URL}" refs/heads/main | cut -f1)"

test -n "${GITHUB_SHA}"
test -n "${GITEA_SHA}"
test "${GITEA_SHA}" = "${GITHUB_SHA}"
printf 'mirror main SHA: %s\n' "${GITHUB_SHA}"
```

SHA가 다르면 Gitea pull mirror의 마지막 오류와 GitHub read-only credential을 확인합니다. 인터넷 단절 중에는 GitHub SHA를 조회할 수 없으므로 마지막 동기화 SHA만 복구 참고값으로 사용합니다. 연결이 돌아오면 GitHub `main`을 기준으로 다시 동기화합니다.

## GitHub Actions CI

workflow는 private GitHub `JiwonDev/notiflex-platform`에 둡니다. source checkout은 GitHub를 사용하고 image push는 Harbor로 보냅니다.
workflow는 `main`의 `app/**`, 실제 build script, workflow 파일 변경만 받습니다. job은 `[self-hosted, Linux, X64, homeserver-build]` label을 사용하고 `contents: read`를 기본 권한으로 둡니다. 아래 Multi-arch build와 GitOps promotion은 credential 정리를 위해 서로 다른 step에서 실행합니다.

```yaml
concurrency:
  group: notiflex-main-promotion
  cancel-in-progress: true
```

`homeserver-build` label은 항상 켜 두는 win notebook Ubuntu VM의 trusted runner에만 부여합니다. fork PR과 외부 branch는 이 runner에서 실행하지 않습니다. Windows 재부팅과 VM 유지보수는 실행 중인 job이 없을 때 진행합니다.

## Multi-arch build

trusted GitHub Actions self-hosted runner에서 실행합니다.

```bash
set -euo pipefail
: "${HARBOR_REGISTRY:?set HARBOR_REGISTRY}"
: "${HARBOR_IMAGE:?set HARBOR_IMAGE}"
: "${HARBOR_PUSH_USER:?set HARBOR_PUSH_USER}"
: "${HARBOR_PUSH_PASSWORD:?set HARBOR_PUSH_PASSWORD}"
: "${GITHUB_SHA:?missing GITHUB_SHA}"
SHORT_SHA="$(git rev-parse --short=12 HEAD)"
TAG="sha-${SHORT_SHA}"
IMAGE="${HARBOR_IMAGE}"

git fetch origin main
test "${GITHUB_SHA}" = "$(git rev-parse origin/main)"

printf '%s' "${HARBOR_PUSH_PASSWORD}" |
  docker login "${HARBOR_REGISTRY}" \
    --username "${HARBOR_PUSH_USER}" --password-stdin
unset HARBOR_PUSH_PASSWORD
trap 'docker logout "${HARBOR_REGISTRY}" >/dev/null 2>&1 || true' EXIT

docker buildx use homeserver-builder
docker buildx inspect --bootstrap
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  --tag "${IMAGE}:${TAG}" \
  --metadata-file build-metadata.json \
  --push app/

trivy image --severity HIGH,CRITICAL --exit-code 1 "${IMAGE}:${TAG}"

DIGEST="$(jq -er '."containerimage.digest"' build-metadata.json)"
case "${DIGEST}" in sha256:*) ;; *) exit 1 ;; esac

docker buildx imagetools inspect --raw "${IMAGE}@${DIGEST}" > manifest.json
jq -e '
  [.manifests[].platform | "\(.os)/\(.architecture)"] as $platforms
  | ($platforms | index("linux/arm64")) != null
  and ($platforms | index("linux/amd64")) != null
' manifest.json

if [ -n "${GITHUB_ENV:-}" ]; then
  printf 'IMAGE=%s\nTAG=%s\nDIGEST=%s\n' \
    "${IMAGE}" "${TAG}" "${DIGEST}" >> "${GITHUB_ENV}"
fi
```

이미지는 Harbor에 push된 뒤 Trivy로 검사됩니다. 검사 실패 시 image는 남지만 promotion은 실행하지 않습니다. `DIGEST`는 검사를 통과하고 두 아키텍처를 포함한 최상위 manifest digest여야 합니다.

## GitOps promotion

`INFRA_GITOPS_SSH_KEY`에는 `JiwonDev/infra-gitops`의 write-enabled deploy key private key를, `GITHUB_KNOWN_HOSTS`에는 검증한 GitHub host key를 넣습니다.

```bash
set -euo pipefail
: "${IMAGE:?missing IMAGE}"
: "${TAG:?missing TAG}"
: "${DIGEST:?missing DIGEST}"
: "${GITHUB_SHA:?missing GITHUB_SHA}"
: "${INFRA_GITOPS_SSH_KEY:?missing INFRA_GITOPS_SSH_KEY}"
: "${GITHUB_KNOWN_HOSTS:?missing GITHUB_KNOWN_HOSTS}"

git fetch origin main
test "${GITHUB_SHA}" = "$(git rev-parse origin/main)"

install -d -m 0700 "${HOME}/.ssh"
printf '%s\n' "${GITHUB_KNOWN_HOSTS}" > "${HOME}/.ssh/known_hosts"
chmod 0600 "${HOME}/.ssh/known_hosts"

eval "$(ssh-agent -s)"
trap 'ssh-agent -k >/dev/null' EXIT
printf '%s\n' "${INFRA_GITOPS_SSH_KEY}" | tr -d '\r' | ssh-add -

git clone git@github.com:JiwonDev/infra-gitops.git infra-gitops
cd infra-gitops/apps/notiflex/overlays/homeserver
REPO_ROOT="$(git rev-parse --show-toplevel)"

kustomize edit set image "notiflex-api=${IMAGE}@${DIGEST}"
printf 'APP_VERSION=%s\n' "${TAG}" > version.env

kustomize build . > /tmp/notiflex.yaml
kubeconform -strict -summary /tmp/notiflex.yaml
git diff --check

UNEXPECTED="$(git -C "${REPO_ROOT}" diff --name-only |
  grep -Ev '^apps/notiflex/overlays/homeserver/(kustomization.yaml|version.env)$' || true)"
test -z "${UNEXPECTED}"

git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
git add kustomization.yaml version.env
git diff --cached --quiet && exit 0
git commit -m "deploy: notiflex ${TAG} ${DIGEST}"

if ! git push origin HEAD:main; then
  git fetch origin main
  git rebase origin/main
  kustomize build . > /tmp/notiflex.yaml
  kubeconform -strict -summary /tmp/notiflex.yaml

  UNEXPECTED="$(git -C "${REPO_ROOT}" diff --name-only origin/main...HEAD |
    grep -Ev '^apps/notiflex/overlays/homeserver/(kustomization.yaml|version.env)$' || true)"
  test -z "${UNEXPECTED}"
  git push origin HEAD:main
fi
```

첫 push 뒤에는 한 번만 rebase와 재검증을 수행합니다. conflict와 두 번째 push 실패는 job 실패로 남깁니다.

## E2E 확인 명령

### GitHub Actions와 promotion commit

workflow run ID와 build에 사용한 source SHA, promotion commit, 최상위 manifest digest를 전달합니다.

```bash
set -euo pipefail
: "${GITHUB_RUN_ID:?set GITHUB_RUN_ID}"
: "${SOURCE_SHA:?set SOURCE_SHA}"
: "${PROMOTION_SHA:?set PROMOTION_SHA}"
: "${EXPECTED_DIGEST:?set EXPECTED_DIGEST}"
: "${INFRA_GITOPS_DIR:=D:/pycharm/infra-gitops}"
case "${EXPECTED_DIGEST}" in sha256:*) ;; *) exit 1 ;; esac

RUN_JSON="$(gh run view "${GITHUB_RUN_ID}" \
  --repo JiwonDev/notiflex-platform \
  --json headSha,conclusion)"
jq -e --arg sha "${SOURCE_SHA}" \
  '.headSha == $sha and .conclusion == "success"' <<<"${RUN_JSON}"

git -C "${INFRA_GITOPS_DIR}" fetch origin main
git -C "${INFRA_GITOPS_DIR}" cat-file -e "${PROMOTION_SHA}^{commit}"
git -C "${INFRA_GITOPS_DIR}" merge-base --is-ancestor \
  "${PROMOTION_SHA}" origin/main

EXPECTED_VERSION="sha-${SOURCE_SHA:0:12}"
git -C "${INFRA_GITOPS_DIR}" show \
  "${PROMOTION_SHA}:apps/notiflex/overlays/homeserver/version.env" |
  grep -Fx "APP_VERSION=${EXPECTED_VERSION}"
git -C "${INFRA_GITOPS_DIR}" show \
  "${PROMOTION_SHA}:apps/notiflex/overlays/homeserver/kustomization.yaml" |
  grep -F "digest: ${EXPECTED_DIGEST}"
```

### ArgoCD와 rollout

```bash
set -euo pipefail
: "${PROMOTION_SHA:?set PROMOTION_SHA}"

APP_REVISION="$(kubectl --context default get application \
  notiflex-home -n argocd -o jsonpath='{.status.sync.revision}')"
APP_SYNC="$(kubectl --context default get application \
  notiflex-home -n argocd -o jsonpath='{.status.sync.status}')"
APP_HEALTH="$(kubectl --context default get application \
  notiflex-home -n argocd -o jsonpath='{.status.health.status}')"

test "${APP_REVISION}" = "${PROMOTION_SHA}"
test "${APP_SYNC}" = 'Synced'
test "${APP_HEALTH}" = 'Healthy'
kubectl --context default rollout status \
  deployment/notiflex-api -n notiflex --timeout=300s

PDB_MIN="$(kubectl --context default get pdb \
  notiflex-api -n notiflex -o jsonpath='{.spec.minAvailable}')"
test "${PDB_MIN}" = '1'
```

### 배포 digest와 Pod 분산

```bash
set -euo pipefail
: "${HARBOR_IMAGE:?set HARBOR_IMAGE}"
: "${EXPECTED_DIGEST:?set EXPECTED_DIGEST}"
case "${EXPECTED_DIGEST}" in sha256:*) ;; *) exit 1 ;; esac

DEPLOYED_IMAGE="$(kubectl --context default get deployment \
  notiflex-api -n notiflex \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
EXPECTED_IMAGE="${HARBOR_IMAGE}@${EXPECTED_DIGEST}"

test "${DEPLOYED_IMAGE}" = "${EXPECTED_IMAGE}"
printf 'deployed image: %s\n' "${DEPLOYED_IMAGE}"

mapfile -t POD_NODES < <(
  kubectl --context default get pods -n notiflex \
    -l app=notiflex-api \
    -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' |
    sed '/^$/d' | sort -u
)
EXPECTED_REPLICAS="${EXPECTED_REPLICAS:-2}"
test "${#POD_NODES[@]}" -eq "${EXPECTED_REPLICAS}"
printf 'pod nodes: %s\n' "$(IFS=,; printf '%s' "${POD_NODES[*]}")"

kubectl --context default get pods -n notiflex \
  -l app=notiflex-api \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\t"}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

Pod의 `imageID`는 아키텍처별 하위 digest로 보일 수 있습니다.

### `/health` version

별도 terminal에서 port-forward를 유지합니다.

```bash
kubectl --context default port-forward \
  svc/notiflex-api -n notiflex 8080:80
```

```bash
: "${SOURCE_SHA:?set SOURCE_SHA}"
EXPECTED_VERSION="sha-${SOURCE_SHA:0:12}"
curl -fsS http://127.0.0.1:8080/health |
  jq -e --arg version "${EXPECTED_VERSION}" \
    '.status == "ok" and .version == $version'
```

모든 비교가 통과한 뒤 promotion commit을 revert하고 이전 digest와 version으로 같은 검사를 반복합니다.
