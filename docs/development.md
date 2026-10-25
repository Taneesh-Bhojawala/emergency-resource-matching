# Development and troubleshooting

[Back to the project overview](../README.md)

## Build and test

### Backend

Install Java 21 and set `JAVA_HOME` when building outside Docker. The Maven wrapper downloads Maven if necessary.

```powershell
./scripts/build.ps1
```

On Linux/macOS:

```bash
./mvnw verify
```

The backend tests cover shard routing, matching eligibility/distance and error status handling. `build.ps1 -Offline` is intended only for an already populated local dependency cache.

### Frontend

Install Node.js 22, then run:

```powershell
npm --prefix frontend ci
npm --prefix frontend test
npm --prefix frontend run build
```

### Running-system tests

Run against local synthetic data. These scripts create records, and failure drills deliberately interrupt project components. Run them sequentially.

| Script | Checks |
| --- | --- |
| `scripts/smoke.mjs` | Creation, asynchronous matching, lookup, lifecycle and input errors |
| `scripts/load-test.mjs` | Concurrent requests competing for one resource; exactly one winner |
| `scripts/integration.mjs` | Competing allocations, cross-shard crash recovery, stale tokens, outbox and shard distribution |
| `scripts/broker-recovery.mjs` | Persistence during a Compose broker outage and matching after recovery |
| `scripts/backup-restore.mjs` | Per-shard backups restored into separate test databases |
| `scripts/kubernetes-drills.mjs` | Primary promotion, standby catch-up, worker scaling and pod replacement |
| `scripts/kubernetes-read-fallback.mjs` | Explicit replica read when a primary service is unreachable |
| `scripts/kubernetes-observability.mjs` | Heartbeats, metrics, internal token enforcement, dead-lettering and duplicate events |

For basic checks against either deployment:

```powershell
$env:BASE_URL = 'http://localhost:8088/api'
node scripts/smoke.mjs
node scripts/load-test.mjs
```

For the Compose integration tests, first enable the localhost coordinator testing port:

```powershell
docker compose --env-file .env -f infra/compose/compose.yaml -f infra/compose/testing.yaml up -d
node scripts/integration.mjs
node scripts/broker-recovery.mjs
node scripts/backup-restore.mjs
```

Kubernetes tests require additional preparation described in the [operations runbook](runbook.md). In particular, run the Kubernetes smoke test with its documented evidence name before observability checks. Generated reports are stored locally under the Git-ignored `docs/evidence/` directory.

### Continuous integration

The GitHub Actions workflow runs on pushes and pull requests. It builds and tests the backend and frontend, starts the Compose stack, and executes smoke, concurrency, integration, broker-recovery and backup-restore checks. It uploads generated verification artifacts. Check the repository's Actions page for the result of a particular run; the presence of a workflow is not proof that it passed.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Website does not open | Confirm Docker is running and the Compose frontend is up, or restart the Kubernetes port forward |
| Port 8088 is already in use | Stop the competing frontend/forward, or forward Kubernetes as `8089:8080` and use port 8089 |
| Initial `502`/`503` response | Check readiness and logs; Java services and database migrations may still be starting |
| Request shows `NO_RESOURCE` | Check exact type, availability, coordinates and the 50 km radius, then explicitly retry |
| Resource cannot be made available | An active assignment owns it; complete the associated response first |
| Database authentication fails after editing `.env` | Existing PostgreSQL volumes retain their initialized credentials; changing `.env` alone does not change those accounts |
| Kubernetes image is missing | Build the Compose images before running the deployment script |
| Failure drill reports a timeout | Check pod status, logs and available machine resources; verify the drill restored the affected components |

Useful commands:

```powershell
docker compose --env-file .env -f infra/compose/compose.yaml logs --tail 100 matching-service
kubectl --context kind-emergency -n emergency get pods
kubectl --context kind-emergency -n emergency logs deployment/matching-service --tail=100
```

