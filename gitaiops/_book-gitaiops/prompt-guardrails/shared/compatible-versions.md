# 검증된 버전 조합

마지막 검증: run-48 (2026-04-16)

## 핵심 버전

| 컴포넌트 | 버전 | 제약 조건 |
|---------|------|----------|
| Go | 1.25 | OTel SDK v1.42+가 요구 |
| golang Docker | 1.25 | Go 버전과 일치 필수 (scratch 베이스) |
| GKE CSI driver | secrets-store-gke.csi.k8s.io | GKE managed, provider=gke |
| ArgoCD | 3.3.6 (stable) | CRD server-side apply 필수 |
| Argo Rollouts | latest | ArgoCD와 동일 클러스터 |
| Strimzi | 0.51.0 | Kafka 4.x만 지원 (3.x 불가) |
| Kafka | 4.1.0 | Strimzi 0.51+ 호환 (4.1~4.2 검증됨) |
| Strimzi API | kafka.strimzi.io/v1 | v1beta2 deprecated, replicas/resources는 KafkaNodePool에서만 |
| IBM/sarama | 1.47.0 | v1.46.2+에서 Kafka 4.x Version 상수(`V4_0_0_0`, `V4_1_0_0`) 제공. v1.45 이하는 상수 부재로 빌드 실패 |
| OTel SDK | 1.43.0 | Go 1.25+ 필수 |
| gRPC | 1.80.0 | OTel OTLP exporter 호환 |
| valkey-go | 1.0.73 | Go 1.22+ |
| kube-prometheus-stack | latest | requests 축소 필수 (e2-medium 제약) |
| Loki | latest | SingleBinary 모드 |
| Fluent Bit | latest | Loki output plugin |
| Tempo | latest | monolithic 모드 |
| Valkey (Helm) | bitnami/valkey | resourcesPreset=none 필수 |

## 버전 간 의존 관계

```
Go 1.25 ← OTel SDK 1.43 ← gRPC 1.80
                          ← otlptracegrpc
Go 1.25 ← IBM/sarama 1.47 ← Kafka 4.1 ← Strimzi 0.51 (v1 API)
Go 1.22+ ← valkey-go 1.0.x
```

## AI가 이 파일을 사용하는 방법

코드나 매니페스트를 생성할 때 이 테이블의 버전을 사용한다.
독자의 `JOURNEY.md`에 기록된 실제 버전과 이 파일이 충돌하면 **JOURNEY.md(독자의 실제 상태)를 우선**한다.

## 버전 업데이트 절차

1. 새 테스트 런에서 버전 변경이 필요하면 이 파일을 업데이트한다
2. 해당 챕터의 가드레일도 함께 수정한다
3. 독자의 JOURNEY.md "현재 버전" 테이블도 업데이트한다
