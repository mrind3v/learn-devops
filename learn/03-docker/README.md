# 03-docker

Covers `course/session6-7-docker/` @ `fe23e99` (Docker fundamentals: images, Dockerfiles, `run`, multi-stage, a first Compose file). Networking and volumes in depth are topic `04-docker-networking-volumes` (`course/session8-docker-networking-volume/`).

| File | Content |
|---|---|
| [`01-concepts.md`](01-concepts.md) | first principles: namespaces, cgroups, the Mac VM, architecture, layers, Dockerfile reference, PID 1 and signals, cache, multi-stage, Compose, cleanup safety |
| [`02-course-walkthrough.md`](02-course-walkthrough.md) | every course file line by line, plus the problems found (with evidence) |
| [`03-labs.md`](03-labs.md) | Labs 0–10: predict → run → observe → break → fix; answer keys collapsed; cleanup that was actually run |
| [`04-sources.md`](04-sources.md) | tool versions, URLs fetched, UNVERIFIED list, course-repo problems, 5 claims to double-check |

Suggested order: 01 → skim 02 → labs 1–10 (attempt the *Predict* step before opening any collapsed block).

Prerequisites: Docker Desktop running (`docker version` must print a server version); free ports 3000, 3001, 8000, 8080. **Before running any cleanup in `docker.md` or the PDFs, read `01-concepts.md` §1.11.**
