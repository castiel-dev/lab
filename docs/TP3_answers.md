Team Members:

- Ferhat Mohamed Tahar: https://github.com/ferhatmedtahar
- Naoum Ahmed: https://github.com/ahmedsweng
- Omar Chabira: https://github.com/omar-opt

---

# Part A

## Exercise 1

(a) `scoreboard:local` is **266MB**. `python:3.12-slim` is **180MB**. The application image is the same base plus the virtualenv (about 69MB in `docker history`) and the `data/` files.

(b) Two things that exist in the build stage and are not copied into the runtime image:

- The project tree used only to install the app: `requirements.txt`, `pyproject.toml`, `README.md` and `src/`. The runtime stage starts again from `python:3.12-slim` and copies just `/opt/venv` and `data/`.
- The temporary files of `pip install` (downloaded wheels and build directories). Those stay in the build-stage layers. `COPY --from=build` copies the installed virtualenv, not those layers.

(c) The raw project tree is not in the running container, so a process that can read the filesystem does not get the repository, the tests, or a local `.env` or database (`.dockerignore` also keeps `.git`, `.env`, `docs/`, `tests/` and `data/*.db` out of the build). The pip build leftovers are not shipped either, so the running image does not carry unused download and build artifacts that an attacker could reuse.

(d) `COPY requirements.txt` comes before `COPY src/` so the slow `pip install -r requirements.txt` layer stays cached when only the application code changes. The second build changed one line of `src/scoreboard/app.py`. `COPY requirements.txt` and `pip install -r requirements.txt` were `CACHED`. Docker ran `COPY src/` and `pip install --no-deps .` again. If the source had been copied first, that one-line change would have thrown away the dependency layer and repeated the long install.

### Evidence

Image sizes (same base digest the build used, `sha256:5e2bdb1b0d54…`):

```text
$ docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | grep -E "scoreboard|python"
scoreboard:local    266MB
python:3.12-slim    180MB
```

First build finished and tagged the image:

```text
$ docker build -t scoreboard:local .
[+] Building 225.4s (18/18) FINISHED
 => [build 5/8] RUN pip install --no-cache-dir -r requirements.txt    103.8s
 => [build 8/8] RUN pip install --no-cache-dir --no-deps .             10.5s
 => => naming to docker.io/library/scoreboard:local
 => => unpacking to docker.io/library/scoreboard:local
```

Second build, after `echo "# $(date +%s)" >> src/scoreboard/app.py`. The source was restored afterwards with `git checkout src/scoreboard/app.py`.

```text
$ docker build -t scoreboard:local .
[+] Building 37.9s (17/17) FINISHED
 => CACHED [build 2/8] WORKDIR /app
 => CACHED [build 3/8] RUN python -m venv /opt/venv
 => CACHED [build 4/8] COPY requirements.txt .
 => CACHED [build 5/8] RUN pip install --no-cache-dir -r requirements.txt
 => CACHED [build 6/8] COPY pyproject.toml README.md ./
 => [build 7/8] COPY src/ ./src/
 => [build 8/8] RUN pip install --no-cache-dir --no-deps .             23.8s
 => CACHED [runtime 2/5] RUN useradd --system --uid 10001 --no-create-home scoreboard
 => CACHED [runtime 3/5] WORKDIR /app
 => => naming to docker.io/library/scoreboard:local
```

---

# Part B

## Exercise 2

(a) The mount that makes the database writable is the named volume `-v sbdata:/app/data`. The application writes SQLite under `/app/data`. `--tmpfs /tmp` is also writable, but it is an empty in-memory directory and it is not where the database lives. The root filesystem stays read-only.

(b) `sbdata` is storage Docker keeps for this container. A bind mount of `$PWD/data` would write into the project folder on the host. The container runs as UID 10001, which often cannot write a directory owned by the host user, and it would be able to read whatever else is in that folder. The named volume keeps the database when the container is removed, without giving the container the host tree. `data/*.db` is already excluded from the image by `.dockerignore`, so the database belongs in the volume.

(c) Two other options, if an attacker can already run code in the container:

- `--cap-drop ALL` removes every Linux capability. The attacker cannot bind privileged ports, change file ownership, open raw sockets, or mount filesystems.
- `--security-opt no-new-privileges` stops a process from gaining privileges through a setuid or setgid binary. It cannot become root by executing a privileged file.

### Evidence

```text
$ MSYS_NO_PATHCONV=1 docker run -d --name sb -p 8000:8000 -e ADMIN_TOKEN=dev --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges --memory 256m --cpus 0.5 -v sbdata:/app/data scoreboard:local
818dc7c0ddb5a52f9d879d2c110c9f78ee0e30cd093353c6f08c2029d8749044

$ sleep 5 && ./tools/smoke.sh localhost:8000
smoke test against http://localhost:8000
  ok    liveness
  ok    readiness
  ok    version   dev
  ok    metrics
  ok    challenge list
  ok    register  smoke-13676
  ok    duplicate rejected
  ok    correct flag accepted
  ok    wrong flag rejected
  ok    team detail
  ok    scoreboard
  ok    admin refuses an unauthenticated request
all good

$ docker exec sb id
uid=10001(scoreboard) gid=999(scoreboard) groups=999(scoreboard)

$ docker exec sb sh -c 'touch /app/x || echo "root filesystem is read-only"'
touch: cannot touch '/app/x': Read-only file system
root filesystem is read-only
```

---

# Part C

## Exercise 3

(a) Stopping `sb` (uvicorn is PID 1) took **2.225s** real. Stopping `sb2` (a shell is PID 1) took **4.056s** real.

(b) `docker stop` sends SIGTERM to PID 1, then SIGKILL if that process is still alive when the grace period ends. In `sb`, PID 1 is uvicorn, because the Dockerfile `CMD` is in exec form. Uvicorn handles SIGTERM and exits, so the stop is short. In `sb2`, `--entrypoint sh` with `python -m scoreboard; echo exited` makes the shell PID 1. `cat /proc/1/cmdline` shows `sh`, then the python command. The shell does not forward SIGTERM to the child, so Docker cannot finish the stop until it kills the container. That is why `sb2` took longer.

(c) A rolling update replaces twenty replicas one after another. Each replica that ignores SIGTERM keeps its slot until the grace period ends (10 seconds by default). Twenty replicas then add on the order of a few extra minutes, with old and new versions running together and capacity stuck on pods that should already be gone. With uvicorn as PID 1, each replica leaves in about a second and the rollout stays short.

### Evidence

```text
$ time docker stop sb
sb

real    0m2.225s
user    0m0.324s
sys     0m0.463s

$ docker run -d --name sb2 --entrypoint sh scoreboard:local \
  -c 'python -m scoreboard; echo exited'
2e52b65d18cbfd1baf88fa43afe10b6c4b5f54802abf8d6d76dad0439ea877e0

$ sleep 5 && docker exec sb2 cat /proc/1/cmdline; echo
sh-cpython -m scoreboard; echo exited

$ time docker stop sb2
sb2

real    0m4.056s
user    0m0.121s
sys     0m0.291s

$ docker rm sb sb2
sb
sb2
```

The `cmdline` line is `sh` and `python -m scoreboard; echo exited` separated by null bytes, which the terminal prints as one line. PID 1 is the shell.

---

# Part D

## Exercise 4

(a) Trivy reports **2** findings, both **HIGH**, both in the Debian package **zlib1g** `1:1.3.dfsg+really1.3.1-1+b1`:

- `CVE-2026-76844`
- `CVE-2026-85091`

There are **0 CRITICAL** findings. Every Python package in the virtualenv is **0**. These are operating-system packages from the Debian base (`debian 13.7`), not Python libraries.

(b) In lab sheet 2, `pip-audit -r requirements.txt` only reads the Python dependencies. It never opens the base image, so it cannot see `zlib1g`. Trivy reads the saved image and scans both Debian packages and Python packages. The Python packages are clean here, which agrees with `pip-audit`. The two findings come from the operating system that `python:3.12-slim` ships, which the dependency audit does not cover.

(c) The scanner is pinned to `aquasec/trivy:0.75.0`. A `latest` tag moves, and a later scanner can change severity rules or detection, so the same image could pass one day and fail the next. The pin makes the result reproducible and records which program produced it.

### Evidence

```text
$ docker save scoreboard:local -o /tmp/scoreboard.tar
$ MSYS_NO_PATHCONV=1 docker run --rm \
    -v "C:/Users/raouf/AppData/Local/Temp/scoreboard.tar:/img.tar:ro" \
    aquasec/trivy:0.75.0 \
    image --input /img.tar --severity HIGH,CRITICAL --ignore-unfixed

Report Summary

Target                                                    Type         Vulnerabilities
/img.tar (debian 13.7)                                    debian       2
opt/venv/.../annotated_doc-0.0.5                          python-pkg   0
opt/venv/.../annotated_types-0.8.0                        python-pkg   0
opt/venv/.../anyio-4.16.0                                 python-pkg   0
opt/venv/.../certifi-2026.10.23                           python-pkg   0
opt/venv/.../charset_normalizer-3.5.2                     python-pkg   0
opt/venv/.../click-8.5.1                                  python-pkg   0
opt/venv/.../ctf_scoreboard-1.0.0                         python-pkg   0
opt/venv/.../fastapi-0.143.0                              python-pkg   0
opt/venv/.../h11-0.16.0                                   python-pkg   0
opt/venv/.../idna-3.20                                    python-pkg   0
opt/venv/.../opentelemetry_api-1.45.2                     python-pkg   0
opt/venv/.../pip-25.0.1                                   python-pkg   0
opt/venv/.../prometheus_client-0.26.0                     python-pkg   0
opt/venv/.../pydantic-2.14.3                              python-pkg   0
opt/venv/.../pydantic_core-2.50.4                         python-pkg   0
opt/venv/.../python_multipart-0.0.32                      python-pkg   0
opt/venv/.../pyyaml-6.0.3                                 python-pkg   0
opt/venv/.../requests-2.34.2                              python-pkg   0
opt/venv/.../sqlalchemy-2.1.4                             python-pkg   0
opt/venv/.../starlette-1.10.0                             python-pkg   0
opt/venv/.../typing_extensions-4.16.0                     python-pkg   0
opt/venv/.../typing_inspection-0.4.5                      python-pkg   0
opt/venv/.../urllib3-2.8.0                                python-pkg   0
opt/venv/.../uvicorn-0.54.0                               python-pkg   0
usr/local/lib/python3.12/site-packages/pip-25.0.1         python-pkg   0

/img.tar (debian 13.7)
======================
Total: 2 (HIGH: 2, CRITICAL: 0)

Library  Vulnerability   Severity  Status  Installed Version                 Fixed Version
zlib1g   CVE-2026-76844  HIGH      fixed   1:1.3.dfsg+really1.3.1-1+b1      1:1.3.dfsg+really1.3.1-1+deb13u1
zlib1g   CVE-2026-85091  HIGH      fixed   1:1.3.dfsg+really1.3.1-1+b1      1:1.3.dfsg+really1.3.1-1+deb13u1
```

---

# Part E

## Exercise 5

(a) The commit hash moves through these steps:

1. Git: the workflow runs on a push to `main`, and `github.sha` is that commit.
2. The `image` job passes it into the build: `build-args: BUILD_SHA=${{ github.sha }}`.
3. The Dockerfile accepts it with `ARG BUILD_SHA=dev` and stores it with `ENV BUILD_SHA=${BUILD_SHA}`.
4. The process reads that environment variable in `Settings.from_env` (`build_sha=os.getenv("BUILD_SHA", "dev")`).
5. `GET /version` returns it as `commit`.

The image tag is the same hash: `ghcr.io/castiel-dev/lab:<full commit sha>`. A local `docker build` without that argument leaves the default, which is why the smoke test printed `version dev`.

(b) If two different builds are both tagged `latest`, the name points at only one of them, and which one you get depends on when the machine last pulled. A node that already has `latest` keeps the old image, and a new node may pull the other one. A canary then cannot tell the two versions apart, and a rollback cannot ask for the previous `latest` because that name was overwritten. The commit SHA (or the image digest) names one immutable build.

### Evidence

The `image` job is skipped on a pull request (`if: github.event_name == 'push'`). It ran after the merge to `main`.

```text
$ gh run view 42498815391 --json conclusion,headSha,event,url
conclusion: success
event:      push
headSha:    6e58faa36cfc086b73fff61ae07c4dcb75113781
url:        https://github.com/castiel-dev/lab/actions/runs/42498815391

jobs:
  verify   pass   24s
  image    pass   38s
           tags: ghcr.io/castiel-dev/lab:6e58faa36cfc086b73fff61ae07c4dcb75113781
           digest: sha256:ed30664891e4c956afcb71656f305b769a4ea4d272484df4b06abfaefcd39b25
           build-arg BUILD_SHA=6e58faa36cfc086b73fff61ae07c4dcb75113781
```

An empty Docker config (no `ghcr.io` login) can still pull the image, so the package is public:

```text
$ docker --config /tmp/empty-docker pull \
    ghcr.io/castiel-dev/lab:6e58faa36cfc086b73fff61ae07c4dcb75113781
Digest: sha256:ed30664891e4c956afcb71656f305b769a4ea4d272484df4b06abfaefcd39b25
Status: Image is up to date for ghcr.io/castiel-dev/lab:6e58faa36cfc086b73fff61ae07c4dcb75113781
```

The running container reports that same commit:

```text
$ IMG=ghcr.io/castiel-dev/lab:6e58faa36cfc086b73fff61ae07c4dcb75113781
$ docker run -d --name pub -p 8000:8000 "$IMG"
afa6646526a2b9f77beec969b58cfd9039b5f6931d32a4182d67842a8cb3dee6

$ sleep 5 && curl -s localhost:8000/version; echo
{"version":"1.0.0","commit":"6e58faa36cfc086b73fff61ae07c4dcb75113781","built":"unknown","environment":"local","scoring_mode":"decay"}

$ git rev-parse HEAD
6e58faa36cfc086b73fff61ae07c4dcb75113781

$ docker rm -f pub
pub
```

`commit` in `/version` is the merge commit on `main`, the image tag, and `BUILD_SHA`.
