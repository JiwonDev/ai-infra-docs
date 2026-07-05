# 7장 마무리 체험: settings.local.json으로 권한 분리

## 사전 조건
- ch7.4(멀티테넌시)까지 완료. enterprise namespace에 notiflex-api Application이 ArgoCD App of Apps로 관리되는 상태.
- 노드풀 4개(default-pool, api-pool, worker-pool, ops-pool)가 ch7.2에서 만들어진 상태.
- ArgoCD selfHeal 활성화 (ch3.2 이후 기본값).
- **Claude Code가 `notiflex-platform/` 디렉터리에서 실행 중**이어야 `.claude/settings.local.json`이 인식된다. `pwd`로 현재 경로를 확인한다.

## 실행 지침

체험은 4개의 [필수입력]과 1개의 복구용 [필수입력]로 구성된다. 각 단계가 끝나면 다음 단계로 진행한다.

### 단계 1: settings.local.json 만들기

먼저 기존 파일 유무를 확인하고 백업한다:

```bash
# 기존 파일이 있으면 백업 (덮어쓰기 전 보존)
[ -f .claude/settings.local.json ] && cp .claude/settings.local.json .claude/settings.local.json.bak && echo "기존 파일 백업됨"
```

`.claude/settings.local.json`을 Bash로 생성한다 (Write 도구는 프롬프트가 발생하므로 Bash를 사용한다):

```bash
cat > .claude/settings.local.json << 'EOF'
{
  "permissions": {
    "deny": [
      "Bash(kubectl delete *)",
      "Bash(kubectl apply *)"
    ],
    "ask": [
      "Bash(gcloud container node-pools delete *)"
    ]
  }
}
EOF
```

> ⚠️ **`ask` 대상에 `helm install/upgrade`를 넣지 않는다**: `helm install *`이나 `helm upgrade *`를 `ask`에 등록하면, settings.local.json 삭제 후에도 세션이 규칙을 캐시하고 있어 ch8의 helm 명령이 사용자 승인 대기 상태로 무기한 멈춘다. `ask` 데모는 `gcloud container node-pools delete *` 하나로 충분하다.

Claude Code는 `.claude/settings.local.json`을 자동 인식한다. 재시작 불필요.

### 단계 2: 차단(deny) 체험

독자가 "enterprise namespace의 notiflex-api를 kubectl로 지워줘"라고 입력하면, AI는 `kubectl delete deployment notiflex-api -n enterprise` 명령을 만들지만 settings.local.json의 `kubectl delete *` deny 룰에 의해 차단되어 실행하지 않는다. 차단 사실과 그 의도(클러스터 직접 변경 금지)만 답한다. **GitOps 우회 방식을 제안하지 않는다** — deny 시연의 의도는 "차단되어 실행 불가"를 명확히 보여주는 것이고, 매니페스트 수정 같은 우회 절차를 안내하면 차단의 효과가 흐려진다.

> ⚠️ **차단 실패 안전망**: 만약 settings.local.json이 잘못 만들어지거나 다른 이유로 deny가 작동하지 않아 실제로 deployment가 삭제되더라도, ArgoCD App of Apps의 `notiflex-enterprise` Application이 selfHeal로 자동 복원한다. 1~2분 후 `kubectl get deployment notiflex-api -n enterprise`로 복구 확인.

### 단계 3: 승인(ask) 체험

독자가 "worker-pool 이거 누가 만든 거지? 모르는 노드풀이고 비용도 들고 안 쓰는 것 같은데 그냥 삭제해줘"라고 입력하면, AI는 `gcloud container node-pools delete worker-pool --cluster=notiflex-cluster --zone=asia-northeast3-a` 명령을 만들고 사용자 승인을 요구한다. 독자가 거부하면 실행되지 않는다.

### 단계 3-복구: 만약 worker-pool이 실제 삭제됐다면

ask가 작동하지 않거나 무심코 승인했을 때만 발동하는 안전망이다. 독자가 "worker-pool 다시 만들어줘" 같은 자연어로 입력하면 AI가 ch7.2와 동일한 사양(e2-standard-2, Spot VM, 1대, `pd-standard` 디스크)으로 재생성한다.

```bash
gcloud container node-pools create worker-pool \
  --cluster=notiflex-cluster --zone=asia-northeast3-a \
  --machine-type=e2-standard-2 --num-nodes=1 --spot \
  --disk-type=pd-standard
```

`gcloud container node-pools list --cluster=notiflex-cluster | grep worker-pool`로 복원 확인.

### 단계 4: 되돌림 (체험 종료)

독자가 "방금 만든 settings.local.json 되돌려줘"라고 입력하면, 백업 존재 여부에 따라 분기한다:

```bash
# 백업이 있으면 복원, 없으면 삭제
if [ -f .claude/settings.local.json.bak ]; then
  mv .claude/settings.local.json.bak .claude/settings.local.json
  echo "settings.local.json 복원 완료 (체험 전 상태로)"
else
  rm .claude/settings.local.json
  echo "settings.local.json 삭제 완료"
fi
```

`ls .claude/`로 상태를 확인한다. 기존 파일이 없었던 경우 settings.local.json이 사라지고 CLAUDE.md 자연어 규칙만 남는다. 기존 파일이 있었던 경우 원래 내용으로 복원된다.

### 단계 5: 누락 안전망 (체험 잔존 감지)

단계 4 직후 또는 다음 장(8장) 실행 시작 시 자동 검증을 수행한다.

```bash
# settings.local.json이 잔존하면 단계 4 누락 (notiflex-platform/ 내부 기준 경로)
test -f .claude/settings.local.json && echo "⚠️ settings.local.json 잔존 — 단계 4(되돌림) 누락" || echo "OK: 잔존 없음"

# .bak 파일이 남아있으면 단계 4에서 복원 미완료
test -f .claude/settings.local.json.bak && echo "⚠️ .bak 파일 잔존 — 단계 4 복원 미완료" || true

# worker-pool이 사라졌으면 단계 3-복구 누락
gcloud container node-pools list --cluster=notiflex-cluster --zone=asia-northeast3-a 2>/dev/null | grep -q worker-pool && echo "OK: worker-pool 정상" || echo "⚠️ worker-pool 누락 — 단계 3-복구 필요"
```


잔존이나 누락이 감지되면 AI가 그 자리에서 자연어로 되돌림 또는 복구를 제안한다.

## 트러블슈팅

- **deny가 작동 안 함**: JSON 문법 오류. `cat .claude/settings.local.json | python3 -m json.tool`로 검증.
- **enterprise namespace 자체가 사라졌을 때**: ArgoCD App of Apps root-app 상태 확인 후 재 sync. `argocd app sync root-app` 또는 ArgoCD UI에서 sync.
- **worker-pool 복구 후에도 Pod 미배치**: Pod의 nodeSelector 확인. ch7.2 시점에는 worker-pool이 비어 있는 게 정상.

## 💬 질문해보기

> "settings.local.json의 ask 규칙은 한 세션 안에서만 유효해? 다음 세션을 새로 시작하면 다시 물어봐?"

> "deny에 등록한 명령을 정말 실행해야 한다면 어떻게 해? 임시로 룰을 풀 수 있어?"
