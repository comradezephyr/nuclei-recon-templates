# Custom Nuclei Recon Templates

Curated, zero-false-positive Nuclei v3 templates focusing on modern cloud-native and AI/ML infrastructure exposures.

## Template Categories

### AI & MLOps Infrastructure (`exposures/ai-infra/`)
* `ray-dashboard-exposure.yaml`: Detects unauthenticated Ray cluster dashboards and extracts active `job_id` / entrypoints.
* `chromadb-api-exposure.yaml`: Detects exposed ChromaDB vector database instances and parses collection names.

### General Configurations (`exposures/configs/`)
* `exposed-laravel-env.yaml`: Strict detection of publicly accessible `.env` configuration files.

## Usage

```bash
nuclei -u [https://target.com](https://target.com) -t exposures/ai-infra/
