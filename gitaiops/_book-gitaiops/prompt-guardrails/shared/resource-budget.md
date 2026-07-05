# 리소스 예산표

e2-medium 2노드 기준 총 가용: vCPU 4000m, RAM 8GB
시스템 예약 후 실사용 가능: ~vCPU 3200m, RAM ~6.5GB

## 챕터별 누적 CPU requests

| 시점 | 추가 컴포넌트 | CPU requests | 누적 합계 | 잔여 |
|------|-------------|-------------|----------|------|
| ch2 완료 | notiflex-api ×2 | 50m | 100m | 3100m |
| ch3 완료 | ArgoCD (server, controller, repo, redis, dex) | ~500m | 600m | 2600m |
| ch4 완료 | Prometheus 100m + Grafana 50m + Alertmanager 25m + operator 25m + Loki 10m + Fluent Bit ×2 50m + kube-state-metrics 10m | ~320m | 920m | 2280m |
| ch5 완료 | Argo Rollouts controller | ~100m | 1020m | 2180m |
| ch6 완료 | Valkey 50m + CSI DaemonSet ×2 140m + GCP Provider ×2 100m | ~290m | 1310m | 1890m |
| ch7 완료 | 노드 3개 추가 → 가용 +6000m, enterprise rollout 25m | +5975m | — | ~8130m |
| ch8 완료 | Strimzi operator 200m + Kafka broker 500m + Tempo 25m | ~725m | — | ~7405m |

## 위험 구간

**ch6이 가장 위험**: ch4 관측 가능성 스택이 깔린 상태에서 Valkey + CSI Driver 추가.
2노드 e2-medium은 여기서 CPU 95%+ 도달. run-36에서 대규모 Pending 발생.

CSI DaemonSet은 GKE managed이므로 리소스 패치 불가:
- `csi-secrets-store-gke`: 노드당 70m (2노드 = 140m)
- `csi-secrets-store-provider-gke`: 노드당 50m (2노드 = 100m)
- **합계: 240m** — 이 값은 줄일 수 없다.

→ ch6 진입 전, ch4 관측 가능성 스택의 CPU를 선제 축소해야 한다:
  Prometheus 100→5m, Alertmanager 25→5m, Grafana 50→5m, operator 25→5m, Loki 10→5m
→ B/G 배포 중이면 replicas를 2→1로 축소하여 Pod 수를 줄인다.

**ch7 이후 완화**: 노드 풀 추가로 가용량 급증.

## AI가 이 파일을 사용하는 방법

각 챕터 실행 전, 이 표의 "잔여" 값을 확인한다.
잔여가 추가할 컴포넌트의 requests보다 적으면,
실행 전에 독자에게 "리소스가 부족할 수 있다"고 안내하고
requests 축소 또는 기존 컴포넌트 조정을 선제적으로 수행한다.
