# 03-docker · 4. Sources, versions, verification status

## Tool versions used

| Item | Version | How obtained |
|---|---|---|
| macOS / arch | 27.2 (build 26B5091g) / arm64 | `sw_vers`, `uname -m` |
| Docker Engine (client=server) | 29.7.2, API 1.55, linux/arm64 | `docker version` |
| containerd / runc / docker-init | v2.2.5 / 1.3.6 / 0.19.0 | `docker version` (Server → Components) |
| Docker Desktop VM | kernel `7.0.12-linuxkit`, cgroup v2, 8 CPUs, 8.3 GB, OS "Docker Desktop" | `docker info` |
| Image store | containerd snapshotter (`overlayfs`, `io.containerd.snapshotter.v1`) | `docker info` |
| Docker Compose | v5.4.0 (`docker-compose` is a symlink to the same plugin) | `docker compose version`, `ls -l /usr/local/bin/docker-compose` |
| buildx | v0.36.1-desktop.1 | `docker buildx version` |
| Base images (resolved at run time) | `alpine:3.22` (`sha256:5291449c…`), `node:24-alpine` (Node 24.21.0, Alpine 3.24.2, `sha256:ebfe2f90…`; npm 11.19.0), `nginx:latest` → nginx/1.31.6 on Debian 13 (`sha256:abe47724…`), `python:3.11-slim` → Python 3.11.16, pip 24.0, `nginx:alpine`, `redis:alpine` (versions not recorded) | build/run output |
| Other host tools | git 2.54.0, bash 3.2.57 (macOS), Homebrew poppler 26.09.0 (`pdftotext`, to read the PDFs) | |

**Version differences that matter**
- Docker Engine ≥ 29 on fresh installs uses the containerd image store; older material and the classic `overlay2` explanations differ (see the official containerd page below).
- Compose: `docker-compose` (v1) vs `docker compose` (v2+, currently v5.x here). Top-level `version:` is obsolete.
- Course/PDF base tags (`node:14/16`, `python:3.9`, `nginx:1.21`) are older than the tags the Dockerfiles in `session6-7-docker` use (`node:24-alpine`, `python:3.11-slim`).

## Official sources fetched (before writing)

| Topic | URL |
|---|---|
| Architecture, images/containers, namespaces | https://docs.docker.com/get-started/docker-overview/ |
| Dockerfile instructions | https://docs.docker.com/reference/dockerfile/ (and `#healthcheck` anchor) |
| Multi-stage builds | https://docs.docker.com/build/building/multi-stage/ |
| Build cache | https://docs.docker.com/build/cache/ |
| Build context, `.dockerignore` | https://docs.docker.com/build/concepts/context/ |
| Best practices (pinning, USER, RUN, CMD) | https://docs.docker.com/build/building/best-practices/ |
| `docker run` flags | https://docs.docker.com/reference/cli/docker/container/run/ (and `#init`) |
| `docker stop` | https://docs.docker.com/reference/cli/docker/container/stop/ |
| Port publishing | https://docs.docker.com/engine/network/port-publishing/ |
| Resource constraints | https://docs.docker.com/engine/containers/resource_constraints/ |
| Storage drivers, layers, copy-on-write | https://docs.docker.com/engine/storage/drivers/ |
| containerd image store | https://docs.docker.com/engine/storage/containerd/ |
| `docker system prune` | https://docs.docker.com/reference/cli/docker/system/prune/ |
| Docker Desktop networking (Linux VM) | https://docs.docker.com/desktop/features/networking/ |
| Compose `version` | https://docs.docker.com/reference/compose-file/version-and-name/ |
| Compose start order / `depends_on` | https://docs.docker.com/compose/how-tos/startup-order/ |
| Compose v1 vs v2 naming | https://docs.docker.com/compose/intro/history/ |
| Namespaces | https://man7.org/linux/man-pages/man7/namespaces.7.html |
| PID namespaces / init signals | https://man7.org/linux/man-pages/man7/pid_namespaces.7.html |
| cgroups | https://man7.org/linux/man-pages/man7/cgroups.7.html |
| Exit status 128+N | local `man bash` (bash 3.2.57). The GNU manual URL returned HTTP 429 twice (`https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html`), so it was not fetched. |

Fetch failures: `https://docs.docker.com/compose/intro/migrate/` returned 404; `…/compose/releases/migrate/` did not contain v1/v2 details; `https://docs.docker.com/reference/compose-file/services/` failed once. Content that would have come from them is marked UNVERIFIED below.

Course files read (all at `course/… @ fe23e99`): `session6-7-docker/{docker.md, Dockerfile (empty), docker-compose-app/docker-compose.yml, multi-stage-dockerfile/{Dockerfile,package.json,server.js}, nginx-web/{Dockerfile,index.html}, node-app/{Dockerfile,package.json,server.js}, python-app/{Dockerfile,app.py}, docker-basic-cmd.pdf, docker-advance-cmd.pdf, docker-interview-qa.pdf}`. The three PDFs were read as extracted text (`pdftotext -layout`); table layout in the advance PDF is garbled by the extraction and by the PDF itself.

## UNVERIFIED items

1. Roles of containerd vs runc (from the course PDF, not from docs.docker.com).
2. Why `docker stop` on Docker Desktop waited ~3 s, not the documented 10 s; the daemon setting was not inspected. Why `npm start` exits with code 1 on SIGTERM.
3. `--memory-swap == --memory` semantics; the Lab 1 "fix" (bigger limit) was not run.
4. Whether Compose v1 is end-of-life (and since when).
5. Why quoting `"8080:80"` in YAML is recommended (the Compose docs were not fetched for it).
6. Effect of a missing `.dockerignore` (host `node_modules` copied into the image): reasoned from the docs' definition of the build context, not run.
7. Whether the interview PDF's compose YAML is mis-indented in the PDF itself or only in the text extraction.
8. Support status of the older image tags in the interview PDF; the claim that `docker secret` needs Swarm mode.
9. Steps in Exercises 1–5.

## Course-repo problems found (topic 03)

| # | Where | Problem | Evidence |
|---|---|---|---|
| 1 | `python-app/Dockerfile` | build fails twice: no `requirements.txt`; `apt install pip3` does not exist | Lab 6, exit 1 / apt exit 100 |
| 2 | `session6-7-docker/Dockerfile` | 0-byte stray file; `docker build` rejects it | Lab 9: "the Dockerfile cannot be empty" |
| 3 | `multi-stage-dockerfile/Dockerfile` | builder stage builds nothing; final image only 0.9 MB smaller | Lab 4A: 64,913,032 vs 63,999,186 bytes |
| 4 | all app images | run as root; no `.dockerignore`; no lockfile (`npm install` floats `^5.1.0`); `npm start` as PID 1 | Lab 3 (`uid=0`, `npm start` PID 1), Lab 8 |
| 5 | `nginx-web/Dockerfile` | `FROM nginx:latest` floating tag; `CMD` duplicates the base default | Lab 5: resolved nginx/1.31.6; base `cmd` identical |
| 6 | `docker-compose-app` | `web` serves the default nginx page, unrelated to `redis`; `depends_on` only orders start | Lab 7A; Lab 7C |
| 7 | `docker.md`, advance PDF §3, interview PDF Q28 | mass-delete one-liners hit every container/image/volume/network on the daemon | this machine had a `minikube` container + kicbase image (baseline) |
| 8 | `docker-basic-cmd.pdf` §2 | `docker run -d -p 8080 80` missing colon | Lab 3E: `Unable to find image '80:latest'` |
| 9 | PDFs | `docker-compose` v1 spelling and `version: '3.8'` (obsolete) | Lab 7B warning |
| 10 | `docker-advance-cmd.pdf` | tables with shifted rows (§4 lifecycle, §7 compose) | text extraction of pp. 2–3 |
| 11 | `docker-interview-qa.pdf` | unsourced statistics (Q1 "87%"), old tags, inconsistent example names (Q9) | text extraction |

## Five claims to verify yourself against the official docs

1. **`EXPOSE` does not publish a port** and `-P` publishes only exposed ports to random host ports: https://docs.docker.com/reference/dockerfile/#expose and the `run` reference.
2. **`docker stop` default grace period is 10 s** (I measured ~3 s on Docker Desktop): https://docs.docker.com/reference/cli/docker/container/stop/, then look at Docker Desktop's engine settings for an override.
3. **`depends_on` short syntax waits only for "running", not "ready"**: https://docs.docker.com/compose/how-tos/startup-order/.
4. **Docker Engine 29 defaults to the containerd image store on fresh installs** (and what `docker info` shows for each store): https://docs.docker.com/engine/storage/containerd/.
5. **Exit code 137 = 128+9 (SIGKILL), 143 = 128+15 (SIGTERM)** and that a container's PID 1 does not receive signals it has no handler for: `man 7 pid_namespaces` (https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) and your shell's manual.
