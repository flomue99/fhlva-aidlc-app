# Windows + Podman: Troubleshooting Log

Problems hit while setting up the Beer Tracker stack on Windows 11 with Podman (WSL backend), Docker Compose and minikube — what went wrong, why, and how it was fixed.

**Environment:** Windows 11 Home 23H2 · Podman 6.1.4 (WSL machine) · WSL 2.2.4 (initially) · Docker Desktop also installed · minikube v1.38.1

---

## 1. Podman machine never reaches "running"

**Symptom** (Podman Desktop → Podman Machine → Logs):

```
your 131072x1 screen size is bogus. expect trouble
API forwarding for Docker API clients is not available ...
    win-sshproxy.exe failed to start, exit code: 0
Error: machine did not transition into running state: ssh error: machine not in running state
```

**Diagnosis**

- `wsl -l -v` showed the distro `podman-machine-default` as *Running*, yet `podman machine inspect` reported `"State": "stopped"`.
- Inside the distro, PID 1 was WSL's plain `init`, not systemd → `sshd` never started → Podman (which talks to the machine over SSH) never saw it come up.
- `/etc/wsl.conf` contained only `[user] default=user` — the `[boot] systemd=true` block that `podman machine init` normally writes was missing, i.e. provisioning had aborted halfway.

**Learnings**

- `your 131072x1 screen size is bogus` is harmless WSL noise — ignore it.
- "machine not in running state" usually means *sshd inside the machine is not reachable*, not that WSL is down. Check PID 1 and `/etc/wsl.conf` inside the distro.

## 2. `podman machine init` fails with `Wsl/Service/E_UNEXPECTED`

**Symptom** (re-creating the machine):

```
Catastrophic failure
Error code: Wsl/Service/E_UNEXPECTED
Error: the WSL bootstrap script failed: command wsl -u root -d podman-machine-default /root/bootstrap failed: exit status 0xffffffff
```

**Diagnosis**

- Other distros (`wsl -d Debian -- echo ok`) started fine; only the Podman distro crashed WSL on start — even for `echo ok`.
- Podman 6.1 ships a Fedora 44 machine image with a very recent systemd. WSL 2.2.4 (mid-2024) is too old to boot it. This was also the root cause of problem 1: the bootstrap died before it could finish configuring the machine.

**Fix**

```powershell
podman machine rm -f podman-machine-default
wsl --update          # may need an elevated terminal; afterwards wsl --version should be >= 2.5
wsl --shutdown
podman machine init --rootful
podman machine start
```

**Learning:** keep WSL up to date together with Podman. A new Podman version brings a new machine OS image, which may need a newer WSL.

## 3. Moving Podman's storage (images/containers) off `C:`

**Question:** can Podman put images and containers on another drive?

**Answer:** not with a Podman flag. On Windows everything lives inside the machine's WSL virtual disk (`ext4.vhdx`) under `%USERPROFILE%\.local\share\containers\podman\machine\wsl\`. Move that folder and you move everything.

**Fix used — directory junction before `init`:**

```powershell
podman machine rm -f podman-machine-default        # folder must not be in use
New-Item -ItemType Directory -Force D:\podman\wsl
robocopy "$env:USERPROFILE\.local\share\containers\podman\machine\wsl" D:\podman\wsl /E /MOVE   # keeps the cached image
New-Item -ItemType Junction -Path "$env:USERPROFILE\.local\share\containers\podman\machine\wsl" -Target D:\podman\wsl
podman machine init --rootful
podman machine start
```

**Pitfall:** `New-Item -ItemType Junction` fails with *"cannot be removed because it is not empty"* if the folder still exists. Remove the machine and move/delete the folder first.

**Alternative for an existing machine** (WSL ≥ 2.3): `wsl --manage podman-machine-default --move D:\podman\wsl`.

**Don't:** point Podman's `graphroot` at `/mnt/d/...`. The Windows↔Linux file share is slow and breaks overlay storage.

## 4. Docker Hub: `toomanyrequests` (pull rate limit)

**Symptom**

```
reading manifest ... in docker.io/library/maven: toomanyrequests: You have reached your unauthenticated pull rate limit.
```

**Cause:** anonymous Docker Hub pulls are limited per IP. A fresh machine pulls every base image at once.

**Fix options**

1. Log in (a free account gets a higher limit):
   ```powershell
   podman login docker.io
   ```
   If your Docker Hub account uses **Sign in with Google**, it has no password. Create a **Personal Access Token** (hub.docker.com → Account settings → Personal access tokens, read-only is enough) and use your Docker Hub *username* + the token.
2. Use Google's public Docker Hub mirror (`mirror.gcr.io`) via `/etc/containers/registries.conf` inside the machine.
3. Wait — the limit resets over a rolling 6-hour window.

## 5. Maven Central: `429 Too Many Requests` during image build

**Symptom** (`podman-compose up -d` / `docker compose up -d`):

```
Non-resolvable parent POM ... spring-boot-starter-parent:pom:3.3.5 ...
from/to central (https://repo.maven.apache.org/maven2): status code: 429, reason phrase: Too Many Requests (429)
Error: building at STEP "RUN mvn dependency:go-offline -q": exit status 1
```

**Cause:** the four Spring Boot services build **in parallel**, and each Dockerfile runs `mvn dependency:go-offline` with an empty Maven cache. That's four complete dependency downloads at once from one IP, so Maven Central throttles it. The limit is per IP, so building outside the devcontainer doesn't avoid it.

**Fix used:** build one service at a time and retry after a short wait.

```powershell
$env:COMPOSE_PARALLEL_LIMIT=1; docker compose up -d --build
```

or per service:

```powershell
foreach ($s in 'auth-service','beer-tracking-service','statistics-service','health-notification-service') { podman compose build $s }
podman compose up -d
```

**Possible long-term improvements (not implemented yet):**

- BuildKit cache mount in the Dockerfiles (`RUN --mount=type=cache,target=/root/.m2 mvn ...`), so dependencies are downloaded once across services and rebuilds.
- A `settings.xml` with a Maven Central mirror (e.g. `https://maven-central.storage-download.googleapis.com/maven2`).

## 6. Docker Desktop and Podman installed side by side

**Observations**

- `podman compose` picked up **Docker Desktop's** `docker-compose.exe` as its provider, not `podman-compose`. It still works against Podman's API pipe, but the two can behave differently.
- `docker context ls` showed `desktop-linux` (Docker Desktop) as the active context. Switch the Docker CLI to Podman:

  ```powershell
  docker context use podman-machine-default
  ```

**Learning:** the Docker CLI context and the kubectl context are **independent**. "minikube" showing up as a context is the *kubectl* context, not the container engine.

## 7. minikube still tries to use Docker Desktop

**Symptom**

```
* Using the docker driver based on existing profile
X Exiting due to PROVIDER_DOCKER_VERSION_EXIT_1: ... failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
```

**Cause:** the existing minikube profile was created with the `docker` driver; minikube remembers the driver per profile.

**Fix**

```powershell
minikube delete                       # deletes the old cluster and everything deployed in it
minikube config set driver podman
minikube start --driver=podman --container-runtime=containerd
```

Needs a **rootful** Podman machine that is running. The Podman driver on Windows is still marked experimental; Podman Desktop's Kind cluster is a fallback.

## 8. Running `./scripts/deploy-minikube.sh` from PowerShell

**Symptom:** Windows asks *"How do you want to open this file?"*

**Cause:** PowerShell can't execute `.sh` files.

**Fix:** run it with Git Bash, using the **full path**:

```powershell
& "C:\Program Files\Git\bin\bash.exe" ./scripts/deploy-minikube.sh
```

**Pitfall:** plain `bash` in PowerShell resolves to `C:\Windows\System32\bash.exe`, i.e. **WSL's Debian**, which doesn't have the Windows `minikube`, `kubectl` and `helm`. Git Bash uses the Windows tools.

**Pre-flight checklist**

- `minikube status` → host/kubelet/apiserver `Running`
- `.env` exists with `POSTGRES_PASSWORD`, `INTERNAL_API_KEY`, `AUTH_RSA_KEY_PATH`
- RSA key exists, otherwise: `openssl genrsa -out secrets/auth-rs256-private-key.pem 2048`
- The script builds the images inside minikube (`minikube image build`), so the Maven Central 429 from problem 5 can come back. The script is idempotent, so just re-run it.

---

## Key takeaways

1. **Update WSL** (`wsl --update`) whenever Podman is updated. An outdated WSL was the root cause of the machine never starting.
2. "Machine not in running state" means sshd/systemd isn't up inside the machine. Check inside the distro, not just `wsl -l -v`.
3. Decide where Podman's storage lives **before** `podman machine init` (junction trick).
4. Rate limits come from both registries — Docker Hub (log in / mirror) and Maven Central (build sequentially / cache / mirror).
5. With Docker Desktop and Podman side by side, check which engine every tool is actually using: `docker context`, `kubectl config current-context`, minikube profile driver.
6. Run repo `.sh` scripts on Windows with Git Bash's full path, not `bash`.
