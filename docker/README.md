# Docker image build

Build DocuSeal from the repository root context:

```bash
docker build -f docker/Dockerfile -t docuseal .
```

The Dockerfile lives under `docker/` so Vercel does not auto-detect a container deployment at the repo root.