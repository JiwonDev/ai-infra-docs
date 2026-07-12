# 04장 관측 명령 모음

placeholder는 실제 Service 이름, pinned chart version, values 경로로 바꿉니다. Helm은 render에만 사용하고 배포는 `infra-gitops`와 Argo CD가 수행합니다.

## 메트릭 사전 확인

각 터미널에서 `homeserver-v2.yaml` kubeconfig를 지정하고, 그 안의 `default` context와 홈서버 노드 역할을 먼저 확인합니다.

```bash
export KUBECONFIG="$HOME/.kube/homeserver-v2.yaml"
export KUBE_CONTEXT="default"
export METRICS_NAMESPACE="monitoring"
export LOGS_NAMESPACE="observability"

kubectl config current-context
kubectl config get-contexts
kubectl --context "$KUBE_CONTEXT" get nodes -o wide \
  -L node-role.kubernetes.io/control-plane,node-role.kubernetes.io/etcd
kubectl --context "$KUBE_CONTEXT" get nodes \
  -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
kubectl --context "$KUBE_CONTEXT" get storageclass
kubectl --context "$KUBE_CONTEXT" -n longhorn-system \
  get volumes.longhorn.io,replicas.longhorn.io
kubectl --context "$KUBE_CONTEXT" get applications -n argocd
```

현재 등록된 node가 모두 `Ready`인지 확인합니다. Longhorn StorageClass와 volume 상태가 정상이어야 하며 win notebook worker에는 Longhorn replica를 배치하지 않습니다.

metrics-server가 설치되어 있으면 현재 사용량도 확인합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" top nodes
```

## chart render

chart version과 values key를 확인하고 manifest를 render합니다.

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts --force-update
helm repo add grafana \
  https://grafana.github.io/helm-charts --force-update
helm repo update

export KPS_CHART_VERSION="REPLACE_WITH_PINNED_VERSION"
export KPS_VALUES="REPLACE_WITH_KPS_VALUES_PATH"
export LOKI_CHART_VERSION="REPLACE_WITH_PINNED_VERSION"
export LOKI_VALUES="REPLACE_WITH_LOKI_VALUES_PATH"
export ALLOY_CHART_VERSION="REPLACE_WITH_PINNED_VERSION"
export ALLOY_VALUES="REPLACE_WITH_ALLOY_VALUES_PATH"
export METRICS_NAMESPACE="${METRICS_NAMESPACE:-monitoring}"
export LOGS_NAMESPACE="${LOGS_NAMESPACE:-observability}"

helm show values prometheus-community/kube-prometheus-stack \
  --version "$KPS_CHART_VERSION" \
  | grep -nE 'k3s|kubeEtcd|kubeScheduler|kubeControllerManager|kubeProxy'

helm template monitoring \
  prometheus-community/kube-prometheus-stack \
  --version "$KPS_CHART_VERSION" \
  --namespace "$METRICS_NAMESPACE" \
  --include-crds \
  -f "$KPS_VALUES" >/tmp/kps-rendered.yaml
helm template loki grafana/loki \
  --version "$LOKI_CHART_VERSION" \
  --namespace "$LOGS_NAMESPACE" \
  -f "$LOKI_VALUES" >/tmp/loki-rendered.yaml
helm template alloy grafana/alloy \
  --version "$ALLOY_CHART_VERSION" \
  --namespace "$LOGS_NAMESPACE" \
  -f "$ALLOY_VALUES" >/tmp/alloy-rendered.yaml
```

K3s target 근거도 함께 수집합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" -n kube-system \
  get service,endpoints,endpointslices -o wide
kubectl --context "$KUBE_CONTEXT" -n kube-system get pods -o wide
```

세 render 파일이 오류 없이 생성되어야 합니다. 현재 Longhorn PVC와 retention을 확인하고, Prometheus size limit은 PVC 확장 때 함께 추가합니다.

## Grafana와 Prometheus 접속

Application, Pod, PVC, Service를 확인합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" get applications -n argocd
kubectl --context "$KUBE_CONTEXT" get pods,pvc -n monitoring -o wide
kubectl --context "$KUBE_CONTEXT" get servicemonitors -A
kubectl --context "$KUBE_CONTEXT" get prometheus -n monitoring -o yaml
kubectl --context "$KUBE_CONTEXT" get daemonset -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter -o wide
kubectl --context "$KUBE_CONTEXT" get svc -n monitoring
```

Grafana와 Prometheus는 서로 다른 터미널에서 localhost로 엽니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
export GRAFANA_SERVICE="monitoring-grafana"
kubectl --context "$KUBE_CONTEXT" \
  port-forward --address 127.0.0.1 \
  "svc/$GRAFANA_SERVICE" -n monitoring 3000:80
```

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
export PROMETHEUS_SERVICE="monitoring-kube-prometheus-prometheus"
kubectl --context "$KUBE_CONTEXT" \
  port-forward --address 127.0.0.1 \
  "svc/$PROMETHEUS_SERVICE" -n monitoring 9090:9090
```

Prometheus API는 다른 터미널에서 확인합니다.

```bash
curl -s http://127.0.0.1:9090/-/ready
curl -s http://127.0.0.1:9090/api/v1/status/flags \
  | jq '.data | {retentionTime:."storage.tsdb.retention.time",retentionSize:."storage.tsdb.retention.size"}'
curl -s http://127.0.0.1:9090/api/v1/targets \
  | jq '.data.activeTargets[] | {health, scrapeUrl, lastError}'
```

Grafana와 Prometheus PVC는 Longhorn으로 `Bound`되어야 합니다. Longhorn replica는 stable Ubuntu VM/K3s storage node에 의도한 복제 수로 배치되고, Prometheus는 15일 retention과 size limit이 설정되지 않은 상태를 표시해야 합니다.

## Loki와 Alloy 확인

Alloy 배치와 Loki PVC를 먼저 확인합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
LOGS_NAMESPACE="${LOGS_NAMESPACE:-observability}"
kubectl --context "$KUBE_CONTEXT" get pods,pvc,daemonset -n "$LOGS_NAMESPACE" -o wide
kubectl --context "$KUBE_CONTEXT" logs -n "$LOGS_NAMESPACE" \
  -l app.kubernetes.io/name=alloy --tail=100
kubectl --context "$KUBE_CONTEXT" get daemonset -n "$LOGS_NAMESPACE" \
  -l app.kubernetes.io/name=alloy -o yaml \
  | grep -E -- '--storage.path|mountPath:|hostPath:|readOnly:'
kubectl --context "$KUBE_CONTEXT" get svc -n "$LOGS_NAMESPACE"
```

Loki HTTP Service를 localhost로 엽니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
LOGS_NAMESPACE="${LOGS_NAMESPACE:-observability}"
export LOKI_QUERY_SERVICE="loki"
kubectl --context "$KUBE_CONTEXT" \
  port-forward --address 127.0.0.1 \
  "svc/$LOKI_QUERY_SERVICE" -n "$LOGS_NAMESPACE" 3100:3100
```

다른 터미널에서 readiness, retention, label, Notiflex 로그를 확인합니다.

```bash
curl -s http://127.0.0.1:3100/ready
curl -s 'http://127.0.0.1:3100/config?mode=diff' \
  | grep -E 'retention_period|retention_enabled|delete_request_store'
curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq '.data'
curl -G -s http://127.0.0.1:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="notiflex"}' \
  --data-urlencode 'limit=20' | jq '.data.result'
```

Alloy의 `DESIRED`는 현재 scheduling 조건과 맞아야 합니다. `/var/log`는 read-only이고 writable storage path는 노드별 hostPath에 연결되어야 합니다. Loki는 14일 retention을 표시하고 Notiflex log stream을 반환해야 합니다. 로그가 급증하면 보관 기간 또는 수집량을 줄입니다.

## Alertmanager 확인과 추후 receiver 연결

현재 PrometheusRule과 Alertmanager 상태를 확인합니다. AlertmanagerConfig와 receiver는 추후 연결할 때 확인합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" get prometheusrule -n monitoring --show-labels
kubectl --context "$KUBE_CONTEXT" get alertmanager -n monitoring -o yaml
kubectl --context "$KUBE_CONTEXT" get alertmanagerconfig -A --show-labels
kubectl --context "$KUBE_CONTEXT" get svc -n monitoring | grep alertmanager
```

Alertmanager를 localhost로 엽니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
export ALERTMANAGER_SERVICE="monitoring-kube-prometheus-alertmanager"
kubectl --context "$KUBE_CONTEXT" \
  port-forward --address 127.0.0.1 \
  "svc/$ALERTMANAGER_SERVICE" -n monitoring 9093:9093
```

다른 터미널에서 readiness와 활성 alert를 확인합니다.

```bash
curl -s http://127.0.0.1:9093/-/ready
curl -s http://127.0.0.1:9093/api/v2/alerts | jq '.'
```

현재는 PrometheusRule 로드와 alert 상태까지만 확인합니다. 추후 AlertmanagerConfig와 receiver를 연결하면 selector, route, matcher strategy와 Secret 참조를 함께 확인합니다.

## synthetic 시험(추후 receiver 연결 뒤)

`HomeServerAlertPipelineTest` 시험 rule을 Git에 추가하고 Argo CD가 동기화한 뒤 조회합니다. Prometheus와 Alertmanager port-forward는 유지합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" get applications -n argocd
kubectl --context "$KUBE_CONTEXT" get prometheusrule -n monitoring
curl -G -s http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=ALERTS{alertname="HomeServerAlertPipelineTest"}' \
  | jq '.data.result'
curl -s http://127.0.0.1:9093/api/v2/alerts \
  | jq '.[] | select(.labels.alertname == "HomeServerAlertPipelineTest")'
```

`Pending`에서 `Firing`으로 바뀌고 실제 receiver에 메시지가 도착해야 합니다. 표현식을 `vector(0) > 0`으로 바꾼 뒤 `Resolved` 메시지도 확인합니다. 시험 rule은 Git에서 제거합니다.

## restart 시험

`observability-test` overlay를 Git에 추가하고 Argo CD가 동기화한 뒤 restart counter를 확인합니다.

```bash
KUBE_CONTEXT="${KUBE_CONTEXT:-default}"
kubectl --context "$KUBE_CONTEXT" get deployment,pods \
  -n observability-test -l app=crash-oneline
kubectl --context "$KUBE_CONTEXT" get pods \
  -n observability-test -l app=crash-oneline -w
```

Prometheus port-forward를 유지한 다른 터미널에서 restart counter를 확인합니다.

```bash
curl -G -s http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=increase(kube_pod_container_status_restarts_total{namespace="observability-test"}[5m])' \
  | jq '.data.result'
```

`CrashLoopBackOff`와 증가하는 `RESTARTS`가 Prometheus 값에 반영되는지 확인합니다. alert 전달은 추후 receiver 연결 뒤 synthetic 시험으로 검증합니다.
