# Chapter Assets

Store images and other binary assets by chapter-specific folder.

Recommended pattern:

```text
chapters/
  01_AI_시대_개발자의_인프라.md
  02_GKE_대신_홈서버_k3s_환경구성.md
  assets/
    01-ai-era-infra/
      kubernetes-tools-ecosystem.png
    02-environment-setup/
      homeserver-rack.jpg
```

Rules:

- Use one asset folder per document or chapter.
- Keep asset folder names lowercase ASCII slugs.
- Give files descriptive names instead of generic names like `img.png`.
- Link from the chapter document with a relative path, such as:

```markdown
![Kubernetes tools ecosystem](assets/01-ai-era-infra/kubernetes-tools-ecosystem.png)
```
