# deploy-v2: what changed from deploy/, and why

`deploy-v2/` is `deploy/` plus the fixes found on 10 October 2026, when every one of the 30
capstone repositories (SE1 to SE6) was built and started with this kit (docker-compose on
Podman) and checked through the page. `deploy/` is left as it was. Each fix below was hit by
at least one team; each would have stopped teams in the hands-on session.

| What went wrong | Fix in deploy-v2 | Found on |
|---|---|---|
| Angular 21 builds waited forever for an answer to the CLI's usage-data question | `angular.Dockerfile` sets `CI=true` and `NG_CLI_ANALYTICS=false` | SE4-team2 |
| Older Angular builders write `dist/<app>`, not `dist/<app>/browser`; the image found no page | `angular.Dockerfile` serves `dist/*/browser` when it exists, else the folder holding `index.html` | SE1-team1 |
| Health checks called `localhost`, which resolves to IPv6 `::1` first; a service listening on `0.0.0.0` looked unhealthy and held up the stack | Every Dockerfile health check calls `127.0.0.1` | SE6-team1 |
| A Java service with a native library (ONNX Runtime) could not load it on Alpine (no glibc) | `java.Dockerfile` runtime is `eclipse-temurin:<version>-jre` (Ubuntu), not `-jre-alpine` | SE6-team4 |
| Encryption keys were made up as 48 hex characters; services expecting base64 of 32 bytes refused them | `apply.py` makes up `*_KEY` secrets as base64 of 32 bytes (`openssl rand -base64 32`) | SE2-team1 |
| `EMAIL_ENCRYPTION_KEY` was treated as a mail login to type in, because the name contains MAIL | Names with ENCRYPTION, LOOKUP, SIGNING or CIPHER are always generated | SE1-team5 |
| On EKS every pod reads the Secret `app-secrets`, but nothing created it (CreateContainerConfigError) | `apply.py eks` writes `deploy/secrets.env` (git-ignored) and prints the `kubectl create secret` command; the Jenkinsfile stops with a clear message when the Secret is missing | kit review |

## Patterns in the teams' own code (not kit changes; fixed by the teams in the hands-on)

- A value read only from a file (`Config/application.yaml`, `config/services.json`, `.env.example`) with no
  environment fallback. Inside a container the file is not there. Read the environment first, the file second.
- The frontend build needs a file from the repository root (`Config/`, `Contracts/`, `.env`). Set
  `build_from: .` (or the folder that holds it) for the frontend in `project.yaml`.
- `/actuator/health` behind Spring Security: permit it, or the Docker health check and Kubernetes probes fail.
- Database SSL tied to `NODE_ENV=production`: right for RDS, wrong for the local Postgres container. Make it a setting.
