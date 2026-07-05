# ai-infra-docs

AI InfraOps 실습과 정리를 위한 문서 저장소입니다.

## 구조

| 경로 | 용도 |
| --- | --- |
| `chapters/` | 직접 작성하는 본문 문서 |
| `chapters/assets/` | 본문에서 사용하는 이미지와 바이너리 자산 |
| `gitaiops/_book-gitaiops/` | GitAIOps 가이드, 가드레일, 결과 템플릿 참고 자료 |
| `gitaiops/notiflex-platform/` | 실습용 플랫폼 저장소 submodule |

## 작성 규칙

- 상위 폴더명과 레포성 패키지명은 소문자 kebab-case를 사용한다.
- 장 문서 파일명은 `01_...md`, `02_...md`처럼 번호로 정렬한다.
- 문서별 이미지는 `chapters/assets/<문서번호-slug>/` 아래에 둔다.
- 이미지 파일명은 `img.png` 대신 내용을 설명하는 이름을 사용한다.
- `_book-gitaiops` 자료는 참고/가드레일 용도로 사용하고, 실습 결과는 `chapters/`와 `notiflex-platform`에 남긴다.

## 예시

```text
chapters/
  01_AI_시대_개발자의_인프라.md
  02_GKE_대신_홈서버_k3s_환경구성.md
  assets/
    01-ai-era-infra/
      kubernetes-tools-ecosystem.png
    02-environment-setup/
      homeserver-rack.jpg
gitaiops/
  _book-gitaiops/
  notiflex-platform/
```
