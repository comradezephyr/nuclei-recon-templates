# Custom Nuclei Recon Templates

Curated, zero-false-positive Nuclei v3 templates focusing on modern cloud-native and AI/ML infrastructure exposures.

## Template Categories

### AI & MLOps Infrastructure (`exposures/ai-infra/`)
* `qdrant-api-exposure.yaml`: Detects unauthenticated Qdrant vector database REST APIs and extracts collection names.
* `flowise-chatflows-exposure.yaml`: Detects unauthenticated Flowise chatflow APIs and extracts flow names and IDs.
* `langflow-api-exposure.yaml`: Detects unauthenticated Langflow flow APIs and extracts flow names.
* `chromadb-api-exposure.yaml`: Detects exposed ChromaDB REST endpoints (heartbeat + collection enumeration) and extracts collection names.
* `exposed-ray-chroma-api.yaml`: Detects unauthenticated Ray dashboard APIs; extracts job IDs, entrypoints, and ray version.

### General Configurations (`exposures/configs/`)
* `exposed-laravel-env.yaml`: Strict detection of publicly accessible `.env` configuration files.

## Usage

Run a category against a list of targets:

```bash
nuclei -l targets.txt -t exposures/ai-infra/
nuclei -l targets.txt -t exposures/configs/
```

Run a single template:

```bash
nuclei -u https://target.example.com -t exposures/ai-infra/langflow-api-exposure.yaml
```

## Validation

Every template passes `nuclei -validate` on Nuclei v3 and is exercised against loopback-only mock targets (positive match + negative control) before inclusion. See `exposures/ai-infra/test-reports.md` for the Ray/Chroma validation report.
