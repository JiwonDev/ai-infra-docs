# 6장. [전환기] Enterprise를 위한 기반 정비

> **스토리**: 고객이 늘면서 느려지고, 보안도 허술하다 → 대형 고객사를 받기 전에 기반을 정비하자

Valkey 캐시로 성능을 개선하고, Google Secret Manager로 시크릿을 안전하게 관리합니다. Blue/Green에서 Canary 배포로 전환하여 더 안전한 배포 전략을 적용합니다.

---

| 절 | 제목 |
|:---:|------|
| 6.1 | <small>Pod 간 상태 공유: Valkey 캐시</small> |
| 6.2 | <small>시크릿 관리: Google Secret Manager</small> |
| 6.3 | <small>점진적 배포: Canary</small> |
| 💡 | <small>마무리: `claude-context/`로 현재 아키텍처 정리하기</small> |

> 자세한 내용은 책을 참고하세요.
