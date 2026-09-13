# Nuclei template test report - exposed-ray-dashboard-unauthenticated-chromadb-rest

Prepared by: security-researcher
Date: 2026-09-13
Template under test: exposed-ray-dashboard-unauthenticated-chromadb-rest (nuclei v3.8.0)
Scope: loopback-only mock targets. No external or production systems were touched.

## 1. What the template checks

One template, four read-only GET requests. It fires if any of them match.

| # | Endpoint | Service | What a match means |
|---|----------|---------|--------------------|
| 1 | GET /api/jobs/ | Ray dashboard | job listing readable without auth (job_id, status, entrypoint) |
| 2 | GET /api/version | Ray dashboard | Ray version endpoint reachable anon |
| 3 | GET /api/v1/heartbeat | ChromaDB REST | API responds unauthenticated |
| 4 | GET /api/v1/collections | ChromaDB REST | collection enumeration allowed, anon |

Side note: chroma really does use `/api/v1/heartbeat` with a missing "a". That spelling is from their own router definition, not a template typo.

## 2. Where the matchers came from

Response shapes were taken from the current upstream handlers, not only from docs:

- `ray/dashboard/modules/job/job_head.py`:
  `@routes.get("/api/jobs/")` returns a JSON array of job objects (job_id, status, entrypoint, driver_info, type).
  `@routes.get("/api/version")` returns `{"version": ..., "ray_version": ..., "ray_commit": ..., "session_name": ...}`.
- `chromadb/server/fastapi/__init__.py`, `setup_v1_routes()`:
  `/api/v1/heartbeat` returns `{"nanosecond heartbeat": <int>}`.
  `/api/v1/collections` returns a bare JSON array of collection models (uuid id, name, tenant, database).

One quirk worth recording: current chroma main serves the collections list unwrapped, while the old 0.4.x docs showed a `{"collections": [...]}` envelope. The matcher keys off the id/name/tenant fields instead of the wrapper key, so both forms are covered.

## 3. Validation run

### 3.1 Parse and syntax check

```
$ python3 -c "import yaml; d=yaml.safe_load(open('/tmp/exposed_ray_chroma.yaml')); print(len(d['http']), len(d['info']['description'].split()))"
4 15

$ nuclei -duc -validate -t /tmp/exposed_ray_chroma.yaml
[INF] All templates validated successfully
```

4 requests, 15-word description, no comments in the file.

### 3.2 Positive test

Local mock servers (python http.server) on 127.0.0.1:8899 emulating an open Ray dashboard and a default-config chroma deployment.

```
$ nuclei -duc -silent -nc -u http://127.0.0.1:8899 -t /tmp/exposed_ray_chroma.yaml

[exposed-ray-dashboard-unauthenticated-chromadb-rest] [http] [high] http://127.0.0.1:8899/api/jobs/ [""job_id": "02000000"",""entrypoint": "python train.py""]
[exposed-ray-dashboard-unauthenticated-chromadb-rest] [http] [high] http://127.0.0.1:8899/api/version [""ray_version": "3.1.0""]
[exposed-ray-dashboard-unauthenticated-chromadb-rest] [http] [high] http://127.0.0.1:8899/api/v1/heartbeat [""nanosecond heartbeat": 1789320131035773434""]
[exposed-ray-dashboard-unauthenticated-chromadb-rest] [http] [high] http://127.0.0.1:8899/api/v1/collections [""name": "docs_vectors""]
```

All four checks matched, and the extractors pulled real values back.

### 3.3 Negative control

Same template against a generic JSON API that serves a plain `{"version":"1.2.3"}` on /api/version and `{"status":"ok"}` everywhere else (no ray markers, no heartbeat body, no uuid collections).

```
$ nuclei -duc -silent -nc -u http://127.0.0.1:8900 -t /tmp/exposed_ray_chroma.yaml

(no output)
NEGATIVE-CONTROL: NO MATCH
```

No false positive. The per-request `matchers-condition: and` is doing its job.

## 4. Evidence

### 4.1 Ray - job listing (unauth)

```
GET /api/jobs/ HTTP/1.1
Host: 127.0.0.1:8899

HTTP/1.1 200 OK
Content-Type: application/json

[{"job_id": "02000000", "status": "RUNNING", "entrypoint": "python train.py", "driver_info": null, "type": "SUBMISSION"}]
```

### 4.2 Ray - version (unauth)

```
GET /api/version HTTP/1.1
Host: 127.0.0.1:8899

HTTP/1.1 200 OK
Content-Type: application/json

{"version": "v1", "ray_version": "3.1.0", "ray_commit": "abc123", "session_name": "session_2026-01-01"}
```

### 4.3 Chroma - heartbeat (unauth)

```
GET /api/v1/heartbeat HTTP/1.1
Host: 127.0.0.1:8899

HTTP/1.1 200 OK
Content-Type: application/json

{"nanosecond heartbeat": 1789320131035773434}
```

### 4.4 Chroma - collections (unauth)

```
GET /api/v1/collections HTTP/1.1
Host: 127.0.0.1:8899

HTTP/1.1 200 OK
Content-Type: application/json

[{"id": "123e4567-e89b-12d3-a456-426614174000", "name": "docs_vectors", "configuration": {}, "tenant": "default_tenant", "database": "default_database"}]
```

## 5. Notes and caveats

- Matchers were validated against synthetic responses. The response shapes came from current upstream source on 2026-09-13, so they track recent versions. Old Ray releases served /api/version from a different module; the version regex accepts both the "version" and "ray_version" key spellings to cover that.
- An idle Ray cluster returns an empty job list, so request #1 won't fire there - request #2 is the fallback for that case. Per-request matchers are ANDed; the template as a whole is OR over the four checks.
- When scanning a live target, point the base URL at the right origin: Ray dashboard defaults to port 8265, chroma to 8000.
- All checks are GETs, read-only, no state changes on the target.
- Artifacts from this session: /tmp/exposed_ray_chroma.yaml, /tmp/mock_positive.py, /tmp/mock_negative.py (mock servers were stopped after the run).

## 6. Result

Template is syntactically valid, matches all four expected exposures against the mocks, and stays silent on the negative control. Ready for a sanity run against a staging Ray/chroma instance before use in production scanning. 
