---
paths:
  - "**/Dockerfile*"
  - "**/*.dockerfile"
  - "**/docker-compose*.{yml,yaml}"
  - "**/compose*.{yml,yaml}"
---

- Structure Dockerfile to maximize layer cache hits: deps first, code last
- Use BuildKit features (--mount=type=cache) to cache package manager downloads
- Build separate dev and prod images from the same Dockerfile using targets
- Use dive or docker-slim to analyze and minimize image size
- Scan images with trivy or grype in CI before pushing to registry
- Use COPY --chown to set file ownership instead of separate RUN chown
- Set WORKDIR before COPY to establish a clean working directory
- Use OCI labels for image metadata (source, version, description)