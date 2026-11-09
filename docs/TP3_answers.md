Team Members:

- Ferhat Mohamed Tahar: https://github.com/ferhatmedtahar
- Naoum Ahmed: https://github.com/ahmedsweng
- Omar Chabira: https://github.com/omar-opt

---

# Part A

## Exercise 1

(a) `scoreboard:local` is **266MB**. `python:3.12-slim` is **180MB**. The difference is the virtualenv (about 69MB) and the `data/` files.

(b) Two things in the build stage but not in the runtime image:

- The build-only project files: `requirements.txt`, `pyproject.toml`, `README.md` and `src/`.
- The temporary files from `pip install` (downloaded wheels and build directories).

The runtime stage starts again from `python:3.12-slim` and copies only `/opt/venv` and `data/`.

(c) Without the project files, a process that can read the container filesystem cannot see the repository, the tests, or a local `.env` or database. Without the pip build leftovers, the image carries no unused downloads or build artifacts that an attacker could reuse. `.dockerignore` also keeps `.git`, `.env`, `docs/`, `tests/` and `data/*.db` out of the build.

(d) To keep the slow dependency layer cached. The second build changed one line in `src/scoreboard/app.py`; `COPY requirements.txt` and `pip install -r requirements.txt` stayed `CACHED`, and only `COPY src/` and `pip install --no-deps .` ran again. If `src/` were copied first, the same one-line change would invalidate the dependency layer and repeat the long install.

---

# Part B

## Exercise 2

(a) The named volume `-v sbdata:/app/data`. The application writes SQLite under `/app/data`. `--tmpfs /tmp` is writable, but it is not where the database lives, and the root filesystem stays read-only.

(b) A named volume is managed by Docker and keeps the database when the container is removed, without exposing the host project folder. A bind mount of `$PWD/data` would write into the host tree and could fail because the container runs as UID 10001, which may not own that folder. `data/*.db` is already excluded from the image by `.dockerignore`.

(c) `--cap-drop ALL`: the attacker loses every Linux capability, so no privileged ports, `chown`, raw sockets or mounts. `--security-opt no-new-privileges`: the attacker cannot gain privileges through a setuid or setgid binary.

---

# Part C

## Exercise 3

(a) `sb` (uvicorn as PID 1): **2.225s**. `sb2` (shell as PID 1): **4.056s**.

(b) `docker stop` sends SIGTERM to PID 1 and sends SIGKILL after the grace period. In `sb`, the exec-form `CMD` makes uvicorn PID 1, and uvicorn handles SIGTERM and exits. In `sb2`, `sh` is PID 1; the shell does not forward SIGTERM to the `python` child, so Docker has to wait for the grace period and then kill the container.

(c) In a rolling update of twenty replicas, every replica that ignores SIGTERM holds its slot until the grace period ends (10s by default). That adds minutes to the rollout, keeps old and new versions running together, and leaves capacity stuck on pods that should already be gone. With uvicorn as PID 1, each replica leaves in about a second.

---

# Part D

## Exercise 4

(a) **2 findings**, both **HIGH**, both in the Debian package **zlib1g** `1:1.3.dfsg+really1.3.1-1+b1`: `CVE-2026-76844` and `CVE-2026-85091`. There are **0 CRITICAL** findings, and every Python package in the virtualenv is **0**.

(b) `pip-audit -r requirements.txt` only checks Python dependencies and never opens the base image. Trivy scans the saved image, including the Debian packages, so it sees the OS package `zlib1g`. The Python packages are clean, which agrees with `pip-audit`; the difference is the operating system layer that `python:3.12-slim` ships.

(c) A fixed version makes the scan reproducible. A `latest` tag can change detection or severity rules, so the same image could pass one day and fail the next.

---

# Part E

## Exercise 5

(a) The commit identifier travels like this:

1. The push to `main` gives `github.sha`.
2. The `image` job passes it as `build-args: BUILD_SHA=${{ github.sha }}`.
3. The Dockerfile stores it with `ARG BUILD_SHA=dev` and `ENV BUILD_SHA=${BUILD_SHA}`.
4. `Settings.from_env` reads `BUILD_SHA` from the environment.
5. `GET /version` returns it as `commit`.

The image tag is the same hash: `ghcr.io/castiel-dev/lab:<full-commit-sha>`. A local build without the argument keeps the default `dev`.

(b) Both builds would share the name `latest`, but the tag can point to only one of them, and which one a node runs depends on when it last pulled. One node can keep the old image while another pulls the new one, so a canary cannot tell the versions apart and a rollback cannot ask for the previous `latest` because the name was overwritten. The commit SHA or image digest names one immutable build.

---

# What to hand in

Submit the link to the final commit on `main`:

`https://github.com/castiel-dev/lab/tree/<full-commit-sha>`

On that commit:

- `Dockerfile` and `.dockerignore` are on `main`.
- `docs/TP3_answers.md` contains the answers to Exercises 1–5.
- The `image` job is green for that commit.
- `ghcr.io/castiel-dev/lab:<full-commit-sha>` is public and its `/version` returns the same hash.
