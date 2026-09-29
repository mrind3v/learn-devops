# 03-docker · 2. Course files, line by line

All paths are `course/session6-7-docker/… @ fe23e99`. I copied them to a scratch directory and built there; nothing under `course/` was modified. "Observed" means I ran it (Lab number in `03-labs.md`).

## 2.1 `node-app/`

`package.json` (identical in `multi-stage-dockerfile/`)
```json
{ "name": "docker-hello-world", "version": "1.0.0", "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^5.1.0" } }
```
- `scripts.start`: what `npm start` runs. Docker never runs it; the Dockerfile's `CMD ["npm","start"]` does.
- `"express": "^5.1.0"`: any 5.x ≥ 5.1.0. There is **no lockfile** in the folder, so `npm install` resolves the newest matching version at build time. Observed: `added 68 packages`.

`server.js`: an Express app answering `GET /` with `<h1>Hello World from Docker!</h1>`, listening on the hard-coded `PORT = 3000`. Express's `app.listen(PORT)` with no host argument accepts connections on all interfaces, which is what makes `-p 3000:3000` work. Observed (Lab 9): a listener bound to `127.0.0.1` *inside* the container behind `-p 8000:8000` gave `curl: (56) Recv failure: Connection reset by peer`, the same symptom as publishing the wrong port; the same listener bound to `0.0.0.0` answered.

`Dockerfile`
| Line | Meaning | If changed/omitted |
|---|---|---|
| `FROM node:24-alpine` | base: Node 24 on Alpine Linux. Observed layers: `NODE_VERSION=24.21.0`, `alpine-minirootfs-3.24.2` | `node` alone = `latest` tag, far bigger and unpinned |
| `WORKDIR /app` | creates `/app` (owned by root) and makes it the cwd | files land in `/` |
| `COPY package*.json ./` | copies only the dependency manifests first | if you `COPY . .` first, every code edit re-runs `npm install` (Lab 2 break: 6.8 s reinstall) |
| `RUN npm install` | resolves and installs into `/app/node_modules` at build time; adds a 15.1 MB layer | omitted → `Cannot find module 'express'` at runtime (not run) |
| `COPY . .` | copies the rest of the context (incl. `Dockerfile`, any local `node_modules`; no `.dockerignore`) | |
| `EXPOSE 3000` | documentation only; `-P` uses it | `-p 3000:3000` still works without it |
| `CMD ["npm", "start"]` | exec-form default command | see PID 1 below |

Observed in Lab 3: `docker exec … id` → `uid=0(root)`; `docker top` → `npm start` (PID 1 in container) with `node server.js` as its child; `docker stop` took 0.64 s and the exit code was **1**.
The same app with `CMD ["node","server.js"]` took 3.1 s and exited **137**, and with `--init` took 0.11 s and exited 143 (`01-concepts.md` §1.6).

## 2.2 `multi-stage-dockerfile/`

```dockerfile
FROM node:24-alpine AS builder        # stage 1, named "builder"
WORKDIR /app
COPY package*.json ./
RUN npm install                        # installs dev + prod deps
COPY . .                               # copies server.js etc.

FROM node:24-alpine AS production      # stage 2, starts from a clean image
WORKDIR /app
COPY --from=builder /app/package*.json ./
RUN npm install --omit=dev             # installs prod deps AGAIN, from scratch
COPY --from=builder /app/server.js ./  # only server.js crosses from stage 1
EXPOSE 3000
CMD ["npm", "start"]
```
- `AS builder`: names the stage so `COPY --from=builder` and `--target builder` can refer to it.
- `--omit=dev`: skip `devDependencies`. This project has none, so it changes nothing.
- **What it actually saves:** builder stage 64,913,032 B; final image 63,999,186 B; i.e. **0.9 MB** (Lab 4A). The `docker image ls` SIZE column showed 255 MB vs 249 MB because it counts shared layers differently; the byte counts come from `docker image inspect -f '{{.Size}}'`.
- Why: stage 1 does no build step (no `npm run build`, no compile), and stage 2 does the install anyway. The only artifact crossing stages is `server.js`, which was in the context to begin with. This is a correct *syntax* demo of multi-stage and a poor demo of its *purpose*. Lab 4B shows a build where it matters (241 MB → 176 kB).

## 2.3 `nginx-web/`

```dockerfile
FROM nginx:latest                                   # floating tag
COPY index.html /usr/share/nginx/html/index.html    # replaces the default page
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]                  # same as the base image's default
```
- `index.html` lands in nginx's default document root `/usr/share/nginx/html/`.
- `nginx -g "daemon off;"`: `-g` sets a global directive; `daemon off` keeps nginx in the foreground. A container lives as long as its main process; if nginx daemonized, PID 1 would exit and the container would stop.
- Observed (Lab 5): base image `entrypoint=["/docker-entrypoint.sh"]` and `cmd=["nginx","-g","daemon off;"]`. The course's `CMD` restates the default, harmless but redundant.
- `nginx:latest` resolved at my build time to `nginx/1.31.6` on Debian 13 (`trixie`), digest `sha256:abe47724e466…`. It will resolve to something else next month. Pin a version (and ideally a digest).
- Observed: `ps` is not installed in this image (`sh: 1: ps: not found`); don't assume debugging tools exist in slim images.

## 2.4 `python-app/`: does not build

`app.py` is one line: `print("Hello World from Docker!")`. It has no dependencies.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt update && apt install -y pip3 python3   # (a)
COPY requirements.txt .                          # (b)
RUN pip install -r requirements.txt              # (c)
COPY app.py .
CMD ["python", "app.py"]
```
Two independent failures, both observed (Lab 6):
- **(b)** `"/requirements.txt": not found`: the file is not in the folder.
- **(a)** `Error: Unable to locate package pip3` (exit code 100): there is no apt package named `pip3` (the Debian package is `python3-pip`). It is also unnecessary: `python:3.11-slim` already ships Python 3.11.16 and pip 24.0 (observed).
- BuildKit runs independent steps concurrently. The first build reported only (b) and cancelled (a), so fixing (b) alone just reveals (a). That is why you can fix one error and immediately hit another.
- `apt` in scripts also prints `WARNING: apt does not have a stable CLI interface`; use `apt-get`.

Minimal working Dockerfile (built and run, output `Hello World from Docker!`):
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```
Add `COPY requirements.txt .` + `RUN pip install --no-cache-dir -r requirements.txt` **only** when a real `requirements.txt` exists.

## 2.5 `docker-compose-app/docker-compose.yml`

```yaml
services:            # top-level key; each child is one container definition
  web:
    image: nginx:alpine
    ports:
      - "8080:80"    # host 8080 -> container 80
    depends_on:
      - redis        # start order only
  redis:
    image: redis:alpine
```
- No `version:` key → correct for current Compose (see `01-concepts.md` §1.10).
- The quotes around `"8080:80"`: I did not test the unquoted form or check the Compose docs on why quoting is recommended (**UNVERIFIED**); keep the quotes.
- `docker compose config` shows the normalized form: `depends_on: redis: condition: service_started, required: true`; the short syntax is expanded to `service_started`, i.e. "running", not "ready".
- Observed: `web` serves the **default nginx page** (`Welcome to nginx!`), not the course's `index.html`, and nothing connects `web` to `redis`; the file demonstrates syntax, networking and start order but no real interaction. From `web`, `getent hosts redis` → `172.18.0.2 redis`: Compose's default network gives you DNS by service name.
- Compose labeled the containers `com.docker.compose.project=d03compose` / `…service=web|redis`, which is how `docker compose down` finds what to remove.

## 2.6 `docker.md`

Two links (docker overview, GeeksforGeeks architecture) and cleanup one-liners.
- `docker stop $(docker ps -q)`: `-q` prints only IDs; the shell substitutes them as arguments. `docker rm $(docker ps -aq)`: `-a` includes stopped containers. `docker rmi -f $(docker images -q)`. `docker system prune -a`.
- **Danger, verified against this machine's state:** those commands act on every container and image on the daemon. Here that includes the `minikube` container and `kicbase` image. See `01-concepts.md` §1.11. None of them was run.
- Formatting: the "3." numbering starts mid-list and the file is otherwise unstructured; harmless.

## 2.7 `session6-7-docker/Dockerfile` (0 bytes)

An empty file at the top of the folder, apparently a stray leftover. Building a 0-byte Dockerfile (Lab 9) gives `ERROR: failed to build: failed to solve: the Dockerfile cannot be empty`.

## 2.8 The three PDFs

Read with `pdftotext`. Same author/series as the Linux PDFs.

`docker-basic-cmd.pdf` (5 pp), `docker-advance-cmd.pdf` (5 pp): command tables. Corrections found:
- **`docker run -d -p 8080 80 <image>`** (basic PDF, sec. 2) is missing the colon. Observed with `nginx:alpine`: `Unable to find image '80:latest' locally … pull access denied for 80`, exit 125. Docker read `80` as the *image name*. Correct: `-p 8080:80`.
- `docker-compose …` throughout is the v1 spelling; use `docker compose …` (v5.4.0 here; `docker-compose` still works on this machine only because it is a symlink to the plugin).
- `docker network rm $(docker network ls -q)` and `docker volume rm $(docker volume ls -q)` ("Bulk deletion", advance PDF sec. 3) would try to remove **every** network and volume, including minikube's; not run.
- Several advance-PDF tables have shifted or merged cells (sec. 4 "Container Lifecycle": descriptions no longer line up with commands; sec. 7 compose commands scrambled). Treat commands as a list of names and read each in `docker <cmd> --help`.
- Cleanup table entry `docker system prune -a` says "Remove all unused images (not just dangling ones)" as its whole description; the official page also lists containers, networks and build cache (§1.11).

`docker-interview-qa.pdf` (51 pp, 30 Q&A): a summary, not a lab source. Items to treat with care:
- Q1 "87% market share in containerization" and the size/startup table: unsourced.
- Compose examples use `version: '3.8'` (obsolete) and, in the extracted text, inconsistent indentation of `environment:`/`depends_on:` under `web:` (I only saw the text extraction; open the PDF to confirm the layout).
- Old base-image tags used as "good" examples: `nginx:1.21-alpine`, `python:3.9`, `node:14`, `node:16`, `openjdk:11-jre-slim`, `elasticsearch:7.14.0`. I did not check their support status; check before copying.
- Q15 item 7 uses `docker secret create` / `docker service create` for "Docker secrets"; those are Swarm commands, not plain `docker run`. **UNVERIFIED** against docs (not fetched).
- Q9 mixes `web` and `web1/web2` in one example (`docker exec web1 ping web2` after creating only `web`).
- Correct and consistent with the docs I fetched: `EXPOSE` is documentation only (Q10), `CMD` vs `ENTRYPOINT` interaction (Q7), HEALTHCHECK defaults (Q18: interval 30 s, timeout 30 s, start-period 0 s, retries 3), multi-stage `--target` (Q16), exit codes 137/143 (Q8).

## 2.9 Missing from the folder

No `.dockerignore`, no `package-lock.json`, no `requirements.txt`, no `HEALTHCHECK`, no `USER`. Lab 8 adds the non-root user; the rest are listed in the exercises.
