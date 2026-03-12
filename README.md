# Ollama på NAIS

## Security: CVE Mitigation

This repository includes two Dockerfiles:

- **`Dockerfile`** - Uses pre-compiled Ollama binaries (faster builds, but vulnerable to CVE-2025-22871, CVE-2025-47907, CVE-2025-61723)
- **`Dockerfile.build-from-source`** - Builds Ollama from source with patched Go 1.24.4 (recommended for production)

### Quick Start

**Build secure image:**
```bash
docker build -f Dockerfile.build-from-source -t jamph-ollama:secure .
```

