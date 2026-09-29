# 03-docker · 3. Labs

Format per lab: **Predict** (write answers down first) → **Run** → **Observe** → **Break** → **Fix**. My real output and the answer keys are in collapsed blocks; open them after you try.

Environment used for the recorded output: macOS 27.2 arm64, Docker Desktop, Engine 29.7.2, Compose v5.4.0, buildx v0.36.1-desktop.1 (details in `04-sources.md`). Image tags and package versions resolve at run time, so your numbers (sizes, nginx/node versions, timings) will differ; the *behavior* should not.

## Ground rules (read once)

- Your daemon is shared with everything else you run. On my machine it also held a `minikube` container and its image. **Every resource in these labs carries `--label lab=03-docker` or a `d03-` name, and cleanup filters on that.** Never use `$(docker ps -aq)` or `docker system prune` for lab cleanup.
- Ports used: 3000, 3001, 8000, 8080. Check they are free: `lsof -nP -iTCP:3000 -iTCP:8080 -sTCP:LISTEN` should print nothing.
- Run from the repo root. The scratch copy goes in `.tmp/` (gitignored). `course/` is never modified.

## Lab 0 · Setup and baseline

```bash
mkdir -p .tmp/lab03
cp -R course/session6-7-docker/{node-app,multi-stage-dockerfile,nginx-web,python-app,docker-compose-app} .tmp/lab03/
export LAB=$PWD/.tmp/lab03
# baseline to compare against at the end
docker images --format '{{.Repository}}:{{.Tag}}' | sort > $LAB/baseline-images.txt
docker ps -a --format '{{.Names}}' > $LAB/baseline-containers.txt
docker network ls --format '{{.Name}}' | sort > $LAB/baseline-networks.txt
docker volume ls -q | sort > $LAB/baseline-volumes.txt
docker version --format 'client {{.Client.Version}} / engine {{.Server.Version}}'
```
If the last command prints an error about `docker.sock`, Docker Desktop is not running (start it and wait for the whale icon to settle).

<details><summary>My output</summary>

```
client 29.7.2 / engine 29.7.2
```
Baseline on my machine: containers `minikube`; networks `bridge host minikube none`; 1 volume; 6 image entries (`ubuntu`, `python-web`, `hey-cicd`, minikube `kicbase`).
</details>

---

## Lab 1 · A container is a process with a private view (namespaces + cgroups)

Reading: `01-concepts.md` §1.2.

**Predict**
1. Inside `alpine`, what PID does the first process have, and how many processes does `ps` list?
2. With `--pid=host`, what will `ps` show on a Mac, and why?
3. What number will `/sys/fs/cgroup/memory.max` hold for `--memory 64m`? For `cpu.max` with `--cpus 0.5`?
4. Give a container `--memory 32m --memory-swap 32m` and make it allocate ~100 MB. What exit code, and what does `docker inspect` say?

**Run**
```bash
docker run --rm --label lab=03-docker alpine:3.22 sh -c 'echo "my pid: $$"; ps; echo "hostname: $(hostname)"; ls -l /proc/self/ns'
docker run --rm --label lab=03-docker --pid=host alpine:3.22 sh -c 'ps | head -6; echo ...; echo "processes visible: $(ps | wc -l)"'
docker run --rm --label lab=03-docker alpine:3.22 hostname
docker run --rm --label lab=03-docker --uts=host alpine:3.22 hostname
docker run --rm --label lab=03-docker --memory 64m --cpus 0.5 alpine:3.22 sh -c 'echo memory.max=$(cat /sys/fs/cgroup/memory.max); echo cpu.max=$(cat /sys/fs/cgroup/cpu.max); cat /proc/self/cgroup'
docker run --rm --label lab=03-docker alpine:3.22 sh -c 'echo memory.max=$(cat /sys/fs/cgroup/memory.max)'
```

**Break**: exceed the memory limit.
```bash
docker run --label lab=03-docker --name d03-oom --memory 32m --memory-swap 32m alpine:3.22 \
  sh -c 'x=$(head -c 100000000 /dev/zero | tr "\0" a); echo survived'
echo "exit code: $?"
docker inspect -f 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}' d03-oom
docker rm d03-oom
```

**Fix**: raise the limit (`--memory 256m`) and rerun; it prints `survived`. (I did not run the fixed variant, **UNVERIFIED**, but it is the same command with a larger limit.)

<details><summary>My output</summary>

```
my pid: 1
PID   USER     TIME  COMMAND
    1 root      0:00 sh -c echo "my pid: $$"; ps; ...
    7 root      0:00 ps
hostname: 29668b605273
lrwxrwxrwx 1 root root 0 Sep 29 18:50 cgroup -> cgroup:[4026532558]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 ipc -> ipc:[4026532556]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 mnt -> mnt:[4026532554]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 net -> net:[4026532559]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 pid -> pid:[4026532557]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 pid_for_children -> pid:[4026532557]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 time -> time:[4026532687]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 time_for_children -> time:[4026532687]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Sep 29 18:50 uts -> uts:[4026532555]
---
PID   USER     TIME  COMMAND
    1 root      0:00 /initd
    2 root      0:00 [kthreadd]
    3 root      0:00 [pool_workqueue_]
    4 root      0:00 [kworker/R-rcu_g]
    5 root      0:00 [kworker/R-sync_]
...
processes visible: 166
---
04e337728356
docker-desktop
---
memory.max=67108864
cpu.max=50000 100000
0::/
memory.max=max
---
(OOM run)  exit code 137
ExitCode=137 OOMKilled=true
```
</details>

<details><summary>Answer key</summary>

1. PID 1; two processes (`sh` and `ps`). PID namespace.
2. The Linux VM's processes (`/initd`, kernel threads, 166 of them). macOS has no namespaces; Docker Desktop's VM is the "host" for `--pid=host`. Also `--uts=host` printed `docker-desktop`.
3. `67108864` (64 MiB in bytes). `50000 100000` = 50 ms CPU time per 100 ms period. With no `--memory`: `max`.
4. `137` = 128 + 9 (SIGKILL by the kernel OOM killer); `OOMKilled=true`. `--memory-swap` equal to `--memory` means no swap headroom (flag semantics not verified in docs here: **UNVERIFIED**).
</details>

---

## Lab 2 · Layers and the build cache (`node-app`)

Reading: `01-concepts.md` §1.4, §1.7. Course file: `node-app/Dockerfile`.

**Predict**: for each of the four builds below, which of the five steps print `CACHED`?
(a) first build; (b) rebuild, nothing changed; (c) change the text in `server.js`; (d) bump `"version"` in `package.json`.

**Run**
```bash
cd $LAB/node-app
docker build --progress=plain -t d03-node:v1 --label lab=03-docker .
docker history d03-node:v1
docker image inspect d03-node:v1 -f 'layers={{len .RootFS.Layers}} os/arch={{.Os}}/{{.Architecture}} size={{.Size}}'
docker build --progress=plain -t d03-node:v1 --label lab=03-docker .                       # (b)
sed -i '' 's/Hello World from Docker!/Hello v2/' server.js
docker build --progress=plain -t d03-node:v2 --label lab=03-docker .                       # (c)
sed -i '' 's/"version": "1.0.0"/"version": "1.0.1"/' package.json
docker build --progress=plain -t d03-node:v3 --label lab=03-docker .                       # (d)
cd - >/dev/null
```
(`sed -i ''` is the macOS form.)

**Break**: put `COPY . .` before `RUN npm install`.
```bash
mkdir -p $LAB/node-bad && cp $LAB/node-app/{package.json,server.js} $LAB/node-bad/
cat > $LAB/node-bad/Dockerfile <<'DF'
FROM node:24-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
DF
docker build --progress=plain -t d03-bad:1 --label lab=03-docker $LAB/node-bad 2>&1 | grep -E '^#[0-9]+ \[[0-9]/[0-9]\]|CACHED|added'
sed -i '' 's/Hello v2/Hello bad2/' $LAB/node-bad/server.js      # edit ONE line of source
docker build --progress=plain -t d03-bad:2 --label lab=03-docker $LAB/node-bad 2>&1 | grep -E '^#[0-9]+ \[[0-9]/[0-9]\]|CACHED|added'
```
Gotcha I hit: my first attempt used a `sed` pattern that no longer matched (the file already said "Hello v2"), so nothing changed and everything was `CACHED`. Confirm the edit happened (`grep Hello $LAB/node-bad/server.js`) before believing a cache result.

**Fix**: restore the course order (package manifests → install → source), which is what `node-app/Dockerfile` already does.

**Restore the pristine scratch copy before Lab 3** (Lab 2 edited it; the copy in `course/` was never touched):
```bash
cp course/session6-7-docker/node-app/{server.js,package.json} $LAB/node-app/
diff -r course/session6-7-docker/node-app $LAB/node-app && echo identical
```

<details><summary>My output</summary>

`docker history d03-node:v1` (top layers):
```
IMAGE          CREATED                  CREATED BY                                      SIZE
6cc518ccc599   Less than a second ago   CMD ["npm" "start"]                             0B
<missing>      Less than a second ago   EXPOSE [3000/tcp]                               0B
<missing>      Less than a second ago   COPY . . # buildkit                             16.4kB
<missing>      Less than a second ago   RUN /bin/sh -c npm install # buildkit           15.1MB
<missing>      3 seconds ago            COPY package*.json ./ # buildkit                12.3kB
<missing>      3 seconds ago            WORKDIR /app                                    8.19kB
<missing>      11 days ago              ... (node:24-alpine base layers: 161MB node, 9.31MB alpine rootfs, ...)
layers=8 os/arch=linux/arm64 size=64913165
```
(b) rebuild unchanged: `[2/5] WORKDIR CACHED`, `[3/5] COPY package*.json CACHED`, `[4/5] RUN npm install CACHED`, `[5/5] COPY . . CACHED`.
(c) `server.js` edited: `[2/5]`, `[3/5]`, `[4/5]` `CACHED`; `[5/5] COPY . .` re-ran (`DONE 0.0s`).
(d) `package.json` edited: `[2/5]` `CACHED`; `[3/5] COPY package*.json` re-ran; `[4/5] RUN npm install` re-ran (`added 68 packages, and audited 69 packages in 3s`); `[5/5]` re-ran.

Break, first build: `[3/4] COPY . .`, `[4/4] RUN npm install` → `added 68 packages`. After editing one line of `server.js`:
```
#6 [2/4] WORKDIR /app        CACHED
#7 [3/4] COPY . .
#8 [4/4] RUN npm install
#8 6.824 added 68 packages, and audited 69 packages in 7s
```
</details>

<details><summary>Answer key</summary>

(a) none; (b) all four non-`FROM` steps; (c) `WORKDIR`, `COPY package*.json`, `RUN npm install`; (d) only `WORKDIR`. Rule: a changed layer invalidates every layer after it. In the broken order, `COPY . .` changes whenever any file changes, so `npm install` (the slow step) reruns on every edit.
</details>

---

## Lab 3 · `docker run`: publishing ports, PID 1, and `docker stop`

Reading: `01-concepts.md` §1.6, §1.9. Course file: `node-app/`.

**Predict**
1. Run the image with no `-p`. Can `curl localhost:3000` reach it? What does `docker port` print?
2. With `-p 3001:80` (wrong container port), what does `curl localhost:3001` say?
3. Who is PID 1 in the container, and which user runs it?
4. `docker stop` on `CMD ["npm","start"]` vs `CMD ["node","server.js"]` vs the latter with `--init`: which is slow, and what are the exit codes?

**Run**
```bash
docker build -q -t d03-node:course --label lab=03-docker $LAB/node-app
mkdir -p $LAB/node-exec && cp $LAB/node-app/{package.json,server.js} $LAB/node-exec/
sed 's/CMD \["npm", "start"\]/CMD ["node", "server.js"]/' $LAB/node-app/Dockerfile > $LAB/node-exec/Dockerfile
docker build -q -t d03-node:exec --label lab=03-docker $LAB/node-exec

# A. EXPOSE without -p
docker run -d --label lab=03-docker --name d03-nopub d03-node:course
sleep 2; curl -sS -m 3 http://localhost:3000/; docker port d03-nopub; docker rm -f d03-nopub

# B. published
docker run -d --label lab=03-docker --name d03-web -p 3000:3000 d03-node:course
sleep 2; curl -sS -m 3 http://localhost:3000/; echo; docker port d03-web; docker logs d03-web
docker exec d03-web id; docker exec d03-web ps; docker top d03-web

# D. wrong container port
docker run -d --label lab=03-docker --name d03-wrong -p 3001:80 d03-node:course
sleep 2; curl -sS -m 3 http://localhost:3001/; docker rm -f d03-wrong

# E. the typo from docker-basic-cmd.pdf
docker run -d --label lab=03-docker --name d03-typo -p 8080 80 nginx:alpine; echo "exit $?"

# F. -P and loopback binding
docker run -d --label lab=03-docker --name d03-P -P d03-node:course; docker port d03-P; docker rm -f d03-P
docker run -d --label lab=03-docker --name d03-lo -p 127.0.0.1:3001:3000 d03-node:course; docker port d03-lo; docker rm -f d03-lo

# G. stop behavior
time docker stop d03-web; docker inspect -f 'ExitCode={{.State.ExitCode}}' d03-web; docker rm d03-web
docker run -d --label lab=03-docker --name d03-exec d03-node:exec
sleep 2; time docker stop d03-exec; docker inspect -f 'ExitCode={{.State.ExitCode}}' d03-exec; docker rm d03-exec
docker run -d --label lab=03-docker --name d03-init --init d03-node:exec
sleep 2; time docker stop d03-init; docker inspect -f 'ExitCode={{.State.ExitCode}}' d03-init; docker rm d03-init
```

**Break / Fix**: A, D and E are the breaks; fix each with `-p 3000:3000`, `-p 3001:3000`, `-p 8080:80`. For G the fix for slow, non-graceful stops is exec-form `CMD` plus `--init` (or a SIGTERM handler in the app).

<details><summary>My output</summary>

```
# A
curl: (7) Failed to connect to localhost port 3000 after 0 ms: Couldn't connect to server
(docker port: empty)
# B
<h1>Hello World from Docker!</h1>
3000/tcp -> 0.0.0.0:3000
3000/tcp -> [::]:3000
> docker-hello-world@1.0.0 start
> node server.js
Server running on port 3000
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
PID   USER     TIME  COMMAND
    1 root      0:00 npm start
   18 root      0:00 {MainThread} node server.js
   31 root      0:00 ps
UID   PID   PPID  C  STIME TTY TIME     CMD          (docker top, host-VM PIDs)
root  1606  1583  2  18:51 ?   00:00:00 npm start
root  1638  1606  2  18:51 ?   00:00:00 node server.js
# D
curl: (56) Recv failure: Connection reset by peer
# E
Unable to find image '80:latest' locally
docker: Error response from daemon: pull access denied for 80, repository does not exist or may require 'docker login'
exit 125
# F
3000/tcp -> 0.0.0.0:55000
3000/tcp -> 127.0.0.1:3001
# G
npm start:        real 0.636s   ExitCode=1
node (exec form): real 3.133s   ExitCode=137
node + --init:    real 0.11s    ExitCode=143
```
Repeat of the exec-form run: 3.11 s and 3.09 s, both 137; `--stop-timeout 1` gave 1.12 s. Container `Config.StopTimeout` printed `<nil>`.
</details>

<details><summary>Answer key</summary>

1. No: the daemon blocks unpublished ports; `docker port` prints nothing. `EXPOSE` is documentation.
2. `Connection reset by peer`: the port mapping exists, nothing listens on container port 80. (Same symptom if the app listens only on 127.0.0.1 inside the container, Lab 9.)
3. PID 1 is `npm start`, with `node server.js` as its child; user is `root` (uid 0) because the Dockerfile has no `USER`.
4. Exec-form `node` as PID 1 has no SIGTERM handler, so the kernel does not deliver SIGTERM; Docker waits the grace period and sends SIGKILL → 137. `--init` runs a small init that forwards signals → the app exits on SIGTERM → 143. `npm start` exited fast with 1 (npm relayed the signal; I did not investigate why the status is 1, **UNVERIFIED**). The docs say the default grace period is 10 s; I measured about 3 s on Docker Desktop and did not find the setting responsible, **UNVERIFIED**.
</details>

---

## Lab 4 · Multi-stage: what the course shows vs what it is for

Reading: `01-concepts.md` §1.8.

**Predict**: (1) How much smaller is the course's final image than its `builder` stage? (2) For a C program compiled in stage 1 and copied to `FROM scratch`, how big is the final image? Can you `docker exec` a shell into it?

**Run**
```bash
docker build -q -t d03-ms:builder --target builder --label lab=03-docker $LAB/multi-stage-dockerfile
docker build -q -t d03-ms:prod --label lab=03-docker $LAB/multi-stage-dockerfile
docker image inspect -f '{{.RepoTags}} {{.Size}}' d03-ms:builder d03-ms:prod

mkdir -p $LAB/cbuild && cd $LAB/cbuild
cat > hello.c <<'C'
#include <stdio.h>
int main(void){ puts("hello from a static binary"); return 0; }
C
cat > Dockerfile.single <<'DF'
FROM alpine:3.22
RUN apk add --no-cache gcc musl-dev
WORKDIR /src
COPY hello.c .
RUN gcc -static -o hello hello.c
CMD ["./hello"]
DF
cat > Dockerfile.multi <<'DF'
FROM alpine:3.22 AS builder
RUN apk add --no-cache gcc musl-dev
WORKDIR /src
COPY hello.c .
RUN gcc -static -o hello hello.c

FROM scratch
COPY --from=builder /src/hello /hello
CMD ["/hello"]
DF
docker build -q -f Dockerfile.single -t d03-c:single --label lab=03-docker .
docker build -q -f Dockerfile.multi  -t d03-c:multi  --label lab=03-docker .
docker image ls --format 'table {{.Repository}}:{{.Tag}}\t{{.Size}}' | grep -E 'REPOSITORY|d03-c'
docker run --rm --label lab=03-docker d03-c:single
docker run --rm --label lab=03-docker d03-c:multi
cd - >/dev/null
```

**Break**: `docker run --rm --entrypoint /bin/sh d03-c:multi` (no shell in `scratch`).
**Fix / trade-off**: use a small base with a shell (`alpine`) as the final stage when you need to debug, or `docker build --target builder` to get a debug image.

<details><summary>My output</summary>

```
d03-ms:builder  content=64913032
d03-ms:prod     content=63999186         (0.91 MB smaller; docker image ls SIZE: 255MB vs 249MB)
d03-c:multi     176kB
d03-c:single    241MB
hello from a static binary   (both)
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: exec: "/bin/sh": stat /bin/sh: no such file or directory     [exit 127]
```
</details>

<details><summary>Answer key</summary>

1. About 0.9 MB. The course's stage 1 builds nothing; stage 2 reinstalls dependencies. 2. 176 kB vs 241 MB; no, `scratch` contains only the copied binary. The value of multi-stage is dropping compilers/build caches from the runtime image.
</details>

---

## Lab 5 · `nginx-web`: the writable layer is disposable

**Predict**: after you edit `index.html` inside a running container and then `docker rm -f` it, does a new container from the same image show your edit?

**Run**
```bash
docker build -t d03-nginx:course --label lab=03-docker $LAB/nginx-web
docker image inspect d03-nginx:course -f 'entrypoint={{json .Config.Entrypoint}} cmd={{json .Config.Cmd}} exposed={{json .Config.ExposedPorts}}'
docker run -d --label lab=03-docker --name d03-nginx -p 8080:80 d03-nginx:course
sleep 1; curl -sS -m 3 http://localhost:8080/
docker exec d03-nginx nginx -v
docker exec d03-nginx sh -c 'echo edited-in-container > /usr/share/nginx/html/index.html'; curl -sS -m 3 http://localhost:8080/
docker rm -f d03-nginx
docker run -d --label lab=03-docker --name d03-nginx -p 8080:80 d03-nginx:course
sleep 1; curl -sS -m 3 http://localhost:8080/
docker rm -f d03-nginx
```
**Break**: the edit is lost (that is the point). **Fix**: to persist, build it into the image (change `index.html`, rebuild) or mount a volume/bind mount (topic 04).

<details><summary>My output</summary>

```
entrypoint=["/docker-entrypoint.sh"] cmd=["nginx","-g","daemon off;"] exposed={"80/tcp":{}}
<!DOCTYPE html> ... <h1>Hello World from Nginx + Docker!</h1> ...
nginx version: nginx/1.31.6
edited-in-container
(after rm -f)        curl: (7) Failed to connect to localhost port 8080 ...
(new container)      <h1>Hello World from Nginx + Docker!</h1>
```
`docker exec … ps` → `sh: 1: ps: not found`; `/etc/os-release` → Debian GNU/Linux 13 (trixie).
</details>

<details><summary>Answer key</summary>

No. The edit went into the container's writable layer, which is deleted with the container; the image is unchanged.
</details>

---

## Lab 6 · `python-app`: read a build error, fix it

**Predict**: what are the errors when you build the course Dockerfile as-is? Are there more than one?

**Run**
```bash
docker build --progress=plain -t d03-py:course --label lab=03-docker $LAB/python-app
```
Then create an empty `requirements.txt` in a copy, rebuild, and read the *next* error:
```bash
mkdir -p $LAB/py-step2 && cp $LAB/python-app/{Dockerfile,app.py} $LAB/py-step2/ && : > $LAB/py-step2/requirements.txt
docker build --progress=plain -t d03-py:step2 --label lab=03-docker $LAB/py-step2 2>&1 | grep -E 'Error|ERROR|Unable'
```
**Fix**
```bash
mkdir -p $LAB/py-fixed && cp $LAB/python-app/app.py $LAB/py-fixed/
printf 'FROM python:3.11-slim\nWORKDIR /app\nCOPY app.py .\nCMD ["python", "app.py"]\n' > $LAB/py-fixed/Dockerfile
docker build -q -t d03-py:fixed --label lab=03-docker $LAB/py-fixed
docker run --rm --label lab=03-docker d03-py:fixed
docker run --rm --label lab=03-docker d03-py:fixed sh -c 'python --version; pip --version; id -u'
```
<details><summary>My output</summary>

First build: `ERROR: failed to calculate checksum of ref …: "/requirements.txt": not found` (`Dockerfile:7`).
Second build: `Error: Unable to locate package pip3`, `ERROR: process "/bin/sh -c apt update && apt install -y pip3 python3" did not complete successfully: exit code: 100`.
Fixed: `Hello World from Docker!`; `Python 3.11.16`, `pip 24.0 from /usr/local/lib/python3.11/site-packages/pip (python 3.11)`, uid `0`; image 213 MB.
</details>

<details><summary>Answer key</summary>

Two independent errors: missing `requirements.txt` (COPY) and a nonexistent apt package `pip3` (`python3-pip` is the Debian name, and unnecessary on the `python` image). BuildKit ran steps concurrently, so the first run showed only the COPY error and cancelled the `apt` step.
</details>

---

## Lab 7 · Compose: syntax, `version:`, and what `depends_on` guarantees

Reading: `01-concepts.md` §1.10.

**Predict**: (1) With the course's compose file, what does `curl localhost:8080` show? (2) In a stack where `client` runs `nc -z slow 9000` once at startup and `slow` opens port 9000 after 6 s, does short-syntax `depends_on` make the client succeed? What about `condition: service_healthy`?

**Run A: the course file**
```bash
cd $LAB/docker-compose-app
docker compose -p d03compose config
docker compose -p d03compose up -d --quiet-pull
docker compose -p d03compose ps
curl -sS -m 3 http://localhost:8080/ | head -4
docker compose -p d03compose exec web getent hosts redis
docker network ls --filter name=d03compose --format '{{.Name}} {{.Driver}}'
docker-compose version
docker compose -p d03compose down
cd - >/dev/null
```
**Run B: obsolete `version:`**
```bash
mkdir -p $LAB/dep
{ printf 'version: "3.8"\n'; cat $LAB/docker-compose-app/docker-compose.yml; } > $LAB/dep/compose.versioned.yml
docker compose -p d03ver -f $LAB/dep/compose.versioned.yml config --quiet
```
**Run C: start order vs readiness**
```bash
cat > $LAB/dep/compose.short.yml <<'Y'
services:
  slow:
    image: alpine:3.22
    labels: {lab: "03-docker"}
    command: sh -c "sleep 6; nc -lk -p 9000 -e /bin/cat"
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 9000"]
      interval: 1s
      retries: 20
  client:
    image: alpine:3.22
    labels: {lab: "03-docker"}
    command: sh -c "if nc -z -w 2 slow 9000; then echo CLIENT-RESULT=connected; else echo CLIENT-RESULT=refused; fi"
    depends_on:
      - slow
Y
sed 's/^      - slow$/      slow:\n        condition: service_healthy/' $LAB/dep/compose.short.yml > $LAB/dep/compose.healthy.yml
cd $LAB/dep
for f in compose.short.yml compose.healthy.yml; do
  echo "=== $f"
  docker compose -p d03dep -f $f up --quiet-pull --abort-on-container-exit --exit-code-from client 2>&1 | grep -E 'CLIENT-RESULT|Healthy|Waiting'
  docker compose -p d03dep -f $f down -v
done
cd - >/dev/null
```
**Break**: forget `--abort-on-container-exit --exit-code-from client` and `up` (foreground) never returns because `slow` runs forever; that hung my first attempt. Ctrl-C, then `docker compose -p d03dep down -v`.

<details><summary>My output</summary>

A (`config`): `redis: image: redis:alpine`; `web: depends_on: redis: condition: service_started, required: true`, `image: nginx:alpine`, ports `target: 80 published: "8080"`. After `up`: both containers `Up`; `curl` → the default page (`<title>Welcome to nginx!</title>`); `getent hosts redis` → `172.18.0.2        redis  redis`; network `d03compose_default bridge`; `docker-compose version` → `Docker Compose version v5.4.0`.

B: ``level=warning msg=".../compose.versioned.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"``

C:
```
--- short syntax          client-1  | CLIENT-RESULT=refused
--- service_healthy       Container d03dep-slow-1 Waiting / Healthy
                          client-1  | CLIENT-RESULT=connected
```
</details>

<details><summary>Answer key</summary>

1. The stock nginx welcome page (the course's `index.html` is not used by this compose file), and nothing talks to redis. 2. Short syntax orders *start*, not readiness → `refused`; `service_healthy` waits for the healthcheck → `connected`. (This is the same race the session 8 demo has with MySQL.)
</details>

---

## Lab 8 · Run as non-root (and why order matters)

**Predict**: put `USER node` before `RUN npm install` in `node-app`'s Dockerfile. Does the build succeed?

**Run (break)**
```bash
mkdir -p $LAB/node-user && cp $LAB/node-app/{package.json,server.js} $LAB/node-user/
cat > $LAB/node-user/Dockerfile <<'DF'
FROM node:24-alpine
WORKDIR /app
COPY --chown=node:node package*.json ./
USER node
RUN npm install --omit=dev
COPY --chown=node:node . .
EXPOSE 3000
CMD ["node", "server.js"]
DF
docker build --progress=plain --no-cache -t d03-node:user --label lab=03-docker $LAB/node-user 2>&1 | grep -E 'EACCES|errno|path'
```
**Fix**: install as root, copy the source owned by `node`, drop privileges last.
```bash
cat > $LAB/node-user/Dockerfile <<'DF'
FROM node:24-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --chown=node:node . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
DF
docker build -q -t d03-node:user --label lab=03-docker $LAB/node-user
docker run -d --label lab=03-docker --name d03-user --init -p 3000:3000 d03-node:user
sleep 2; curl -sS -m 3 http://localhost:3000/; echo
docker exec d03-user id
docker exec d03-user sh -c 'touch /app/x || true; ls -ld /app /app/node_modules; touch /etc/x || true'
docker rm -f d03-user
```
<details><summary>My output</summary>

Break:
```
npm error code EACCES
npm error syscall mkdir
npm error path /app/node_modules
npm error errno -13
npm error Error: EACCES: permission denied, mkdir '/app/node_modules'
```
(build exit code 243 without `-q` filtering.) Fixed:
```
<h1>Hello World from Docker!</h1>
uid=1000(node) gid=1000(node) groups=1000(node)
touch: /app/x: Permission denied
drwxr-xr-x    1 root     root          4096 Sep 29 19:02 /app
drwxr-xr-x   67 root     root          4096 Sep 29 19:02 /app/node_modules
touch: /etc/x: Permission denied
```
</details>

<details><summary>Answer key</summary>

No: `WORKDIR /app` created `/app` owned by root, so `node` cannot create `node_modules` there (EACCES). Installing as root and then `USER node` leaves code and dependencies root-owned and read-only to the running app, which is the safer default. `COPY --chown` sets ownership of the copied files.
</details>

---

## Lab 9 · Two small edge cases

```bash
mkdir -p $LAB/emptydf && : > $LAB/emptydf/Dockerfile
docker build $LAB/emptydf; echo "exit $?"                 # like course/session6-7-docker/Dockerfile (0 bytes)

# app listening only on loopback inside the container, published with -p
docker run -d --label lab=03-docker --name d03-loop -p 8000:8000 alpine:3.22 sh -c 'while true; do echo hi | nc -l -s 127.0.0.1 -p 8000; done'
sleep 2; curl -sS -m 3 http://localhost:8000/; echo "curl exit $?"
docker exec d03-loop sh -c 'netstat -ltn | grep 8000'; docker rm -f d03-loop
docker run -d --label lab=03-docker --name d03-all -p 8000:8000 alpine:3.22 sh -c 'while true; do echo hi | nc -l -s 0.0.0.0 -p 8000; done'
sleep 2; curl -sS -m 3 http://localhost:8000/; echo "curl exit $?"; docker rm -f d03-all
```
<details><summary>My output</summary>

```
ERROR: failed to build: failed to solve: the Dockerfile cannot be empty
curl: (56) Recv failure: Connection reset by peer        [curl exit 56]
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN
curl: (1) Received HTTP/0.9 when not allowed             [curl exit 1]
```
The last line is curl rejecting the raw `hi` reply (no HTTP headers): the connection itself worked when bound to `0.0.0.0`. Answer: a service bound to loopback inside the container is unreachable through published ports; bind to `0.0.0.0`.
</details>

---

## Lab 10 · Cleanup (I ran this)

```bash
# containers/networks/compose projects that belong to the labs
docker ps -a -q --filter label=lab=03-docker | xargs docker rm -f       # BSD xargs: does nothing on empty input
for p in d03compose d03dep d03ver; do docker compose -p $p down -v --remove-orphans; done
docker network ls --format '{{.Name}}' | grep '^d03'                    # expect no output
# images: only what is new compared with the baseline from Lab 0
docker images --format '{{.Repository}}:{{.Tag}}' | sort > $LAB/now-images.txt
comm -13 $LAB/baseline-images.txt $LAB/now-images.txt | tee $LAB/to-remove.txt   # READ THIS LIST
xargs docker rmi < $LAB/to-remove.txt
# verify
diff <(sort $LAB/baseline-images.txt) <(docker images --format '{{.Repository}}:{{.Tag}}' | sort) && echo images: same as baseline
diff $LAB/baseline-containers.txt <(docker ps -a --format '{{.Names}}') && echo containers: same
rm -rf "${LAB:?LAB is not set}"      # scratch dir under .tmp/ only
git -C course status --short
```
Not done on purpose: `docker builder prune`. The build cache is shared with your other projects; if you want the ~1 GB back, `docker builder prune` (asks for confirmation) is your call.

<details><summary>My result</summary>

To-remove list: `alpine:3.22, d03-bad:1, d03-bad:2, d03-bad:3, d03-c:multi, d03-c:single, d03-ms:builder, d03-ms:prod, d03-ms:single, d03-nginx:course, d03-node:course, d03-node:exec, d03-node:user, d03-node:v1, d03-node:v2, d03-node:v3, d03-py:fixed, nginx:alpine, redis:alpine`. After removal: image list, container list, network list and volume list identical to the Lab 0 baseline (`minikube` untouched). `git -C course status --short` printed ` M .DS_Store` only: that file was already modified before I started (Finder metadata, 10244 → 10244 bytes); no other file under `course/` changed.
</details>

---

## Exercises (not run; UNVERIFIED until you do them)

1. Add a `.dockerignore` (`node_modules`, `.git`, `*.md`) to `node-app`, run `npm install` on your Mac first, and compare the build-context size (`transferring context:` line) with and without it.
2. Pin `node:24-alpine` to a digest (`docker buildx imagetools inspect node:24-alpine` shows it) and rebuild.
3. Add `HEALTHCHECK --interval=5s CMD wget -qO- http://localhost:3000/ || exit 1` and watch `docker ps` go `starting` → `healthy`; then stop the app process and watch it go `unhealthy`.
4. Replace `npm install` with `npm ci` (needs a `package-lock.json`: generate one with `npm install --package-lock-only`).
5. Write a real multi-stage build for a Node project that has a build step (e.g. `npm run build` producing `dist/`), and measure the difference.
