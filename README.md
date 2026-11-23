# Lab: Self-Hosted GitLab CE + Docker Runner

**Duration:** 3 hours
**Level:** Beginner to intermediate
**Format:** Hands-on, individual or pairs

In this lab you deploy your own GitLab server with Docker Compose, attach a Docker-executor runner that uses the host's Docker engine, and build CI pipelines that run jobs in containers.

---

## Learning objectives

By the end of this lab you will be able to:

1. Deploy GitLab CE with Docker Compose using persistent volumes.
2. Create and register a GitLab Runner with the Docker executor.
3. Explain how the runner, GitLab and job containers talk to each other.
4. Write a `.gitlab-ci.yml` with stages, variables, artifacts and rules.
5. Troubleshoot the most common runner problems.

## Schedule

| Time | Part | Topic |
|------|------|-------|
| 0:00 - 0:15 | 0 | Introduction and architecture |
| 0:15 - 0:50 | 1 | Deploy GitLab with Docker Compose |
| 0:50 - 1:00 | | Break |
| 1:00 - 1:35 | 2 | Add and register the Docker runner |
| 1:35 - 1:55 | 3 | First pipeline: "Hello from Docker" |
| 1:55 - 2:05 | | Break |
| 2:05 - 2:40 | 4 | Exercises: stages, variables, artifacts, rules |
| 2:40 - 2:55 | 5 | Troubleshooting challenge |
| 2:55 - 3:00 | 6 | Cleanup and wrap-up |

---

## Prerequisites

- A Linux host or VM (or Docker Desktop with WSL2), with **8 GB RAM minimum** (GitLab is heavy) and 20 GB free disk.
- Docker Engine 24+ and the Docker Compose plugin (`docker compose version`).
- A web browser and a terminal.
- Free ports: `8080`, `8443`, `2222`.

Check your setup:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

---

## Part 0 - Architecture (15 min)

```
 Host (Docker Engine)
 ├── gitlab-ce       (GitLab server, web on :8080)
 ├── gitlab-runner   (talks to GitLab, mounts /var/run/docker.sock)
 └── job containers  (started by the runner, one per CI job, then removed)

 All of them share the Docker network: gitlab-lab-net
```

Key ideas:

- The runner **does not run jobs itself**. It asks the host's Docker to start a new container for each job.
- This is the "Docker socket binding" approach: job containers are *siblings* of the runner, not children.
- Job containers must reach GitLab by name to clone the repo, so they must join the same Docker network.

**Discussion question:** What can a pipeline do if it has access to `docker.sock`? (Answer in the security section at the end.)

---

## Part 1 - Deploy GitLab (35 min)

### 1.1 Create the project folder

```bash
mkdir gitlab-lab && cd gitlab-lab
```

### 1.2 Create `.env`

```env
GITLAB_VERSION=latest
GITLAB_HOSTNAME=gitlab
GITLAB_EXTERNAL_URL=http://localhost:8080
GITLAB_HTTP_PORT=8080
GITLAB_HTTPS_PORT=8443
GITLAB_SSH_PORT=2222
```

> Pin a real version tag (for example `17.5.2-ce.0`) instead of `latest` for a reproducible lab.

### 1.3 Create `docker-compose.yml`

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:${GITLAB_VERSION}
    container_name: gitlab-ce
    restart: always
    hostname: ${GITLAB_HOSTNAME}
    shm_size: 256m
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url '${GITLAB_EXTERNAL_URL}'
        gitlab_rails['gitlab_shell_ssh_port'] = ${GITLAB_SSH_PORT}
    ports:
      - "${GITLAB_HTTP_PORT}:${GITLAB_HTTP_PORT}"
      - "${GITLAB_HTTPS_PORT}:${GITLAB_HTTPS_PORT}"
      - "${GITLAB_SSH_PORT}:22"
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab

networks:
  default:
    name: gitlab-lab-net
```

The explicit network name (`gitlab-lab-net`) matters later: it lets us tell job containers which network to join.

### 1.4 Start GitLab

```bash
docker compose up -d gitlab
docker compose logs -f gitlab
```

First start takes **3 to 5 minutes**. Wait until the status is healthy:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### 1.5 First login

Get the initial root password (valid for 24 hours):

```bash
docker compose exec gitlab cat /etc/gitlab/initial_root_password
```

Open <http://localhost:8080>, log in as `root`, then change the password.

### Checkpoint 1

- [ ] GitLab UI opens and you are logged in as `root`.
- [ ] You can name the three volumes and what each one stores.

### 1.6 Create the lab project

In the UI: **New project → Create blank project**, name it `hello-ci`, visibility Private, and **untick** "Initialize with a README" only if you plan to push from the terminal. Otherwise leave it ticked and edit files in the web editor.

---

## Part 2 - Add and register the Docker runner (35 min)

### 2.1 Add the runner service

Add this under `services:` in `docker-compose.yml`:

```yaml
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    depends_on:
      - gitlab
    volumes:
      - ./gitlab-runner/config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
```

Start it:

```bash
docker compose up -d gitlab-runner
```

### 2.2 Create the runner in the GitLab UI

Registration tokens are deprecated. Use a **runner authentication token**.

1. Open your project `hello-ci` → **Settings → CI/CD → Runners → New project runner**.
   (For all projects: **Admin Area → CI/CD → Runners → New instance runner**.)
2. Tags: `docker`. Tick **Run untagged jobs** if you want.
3. Click **Create runner** and copy the token. It starts with `glrt-` and is shown **once**.

### 2.3 Register the runner

```bash
docker compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab:8080" \
  --clone-url "http://gitlab:8080" \
  --token "glrt-XXXXXXXXXXXX" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --docker-network-mode "gitlab-lab-net" \
  --docker-pull-policy "if-not-present" \
  --description "docker-host-runner"
```

What each important flag does:

| Flag | Purpose |
|------|---------|
| `--url` | How the **runner** reaches GitLab (container name on the Compose network, with port). |
| `--clone-url` | URL that **job containers** use to clone the repo. Without it they would try `localhost:8080` and fail. |
| `--docker-network-mode` | Puts job containers on `gitlab-lab-net` so the name `gitlab` resolves. |
| `--docker-image` | Default image when a job does not set `image:`. |

### 2.4 Inspect the generated config

```bash
cat gitlab-runner/config/config.toml
```

Find the `[[runners]]` block and identify `url`, `clone_url`, `executor`, and `network_mode`.

### 2.5 Verify

```bash
docker compose exec gitlab-runner gitlab-runner verify
```

In the UI, the runner should show a green **online** dot.

### Checkpoint 2

- [ ] The runner appears online in the UI.
- [ ] You can explain the difference between `url` and `clone_url`.

---

## Part 3 - First pipeline (20 min)

Create `.gitlab-ci.yml` in the root of `hello-ci`:

```yaml
stages:
  - test

hello_docker:
  stage: test
  image: alpine:3.20
  tags:
    - docker
  script:
    - echo "Hello from the Docker runner!"
    - cat /etc/os-release
```

Commit it. Go to **Build → Pipelines**, open the job, and read the log.

While the job runs, watch the containers on the host in a second terminal:

```bash
watch -n 1 docker ps
```

You should see a short-lived container appear and disappear. That is the job container.

### Checkpoint 3

- [ ] Pipeline is green and the log shows `Hello from the Docker runner!`.
- [ ] You saw the job container on the host.

**Question:** Which image ran your script, and where did it come from?

---

## Part 4 - Exercises (35 min)

Do the exercises in order. Each one extends the same `.gitlab-ci.yml`.

### Exercise 1 - Stages and variables

Create a three-stage pipeline: `build`, `test`, `deploy`. Use a global variable and print it in every job.

```yaml
stages: [build, test, deploy]

variables:
  GREETING: "Hello from the lab"

default:
  tags: [docker]
  image: alpine:3.20

build_job:
  stage: build
  script:
    - echo "$GREETING - build"

test_job:
  stage: test
  script:
    - echo "$GREETING - test"

deploy_job:
  stage: deploy
  script:
    - echo "$GREETING - deploy"
```

### Exercise 2 - Artifacts between jobs

Make `build_job` create a file and `test_job` read it.

```yaml
build_job:
  stage: build
  script:
    - echo "built at $(date)" > build.txt
  artifacts:
    paths: [build.txt]
    expire_in: 1 hour

test_job:
  stage: test
  script:
    - cat build.txt
```

Open the finished job in the UI and download the artifact.

### Exercise 3 - Use a different image

Add a job that uses `python:3.12-slim` and prints the Python version.

```yaml
python_job:
  stage: test
  image: python:3.12-slim
  script:
    - python --version
```

### Exercise 4 - Rules

Run the deploy job only on the default branch, and a new job only on merge requests.

```yaml
deploy_job:
  stage: deploy
  script:
    - echo "deploying"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

mr_only_job:
  stage: test
  script:
    - echo "running in a merge request"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Create a branch, push a change, and observe which jobs run.

### Exercise 5 (bonus) - Build an image in a job

Docker-in-Docker needs a privileged runner. Mounting the socket is the simpler option for this lab. Add this to the runner's `config.toml` under `[runners.docker]`:

```toml
volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
```

Then restart the runner (`docker compose restart gitlab-runner`) and use:

```yaml
docker_build:
  stage: build
  image: docker:27-cli
  script:
    - docker version
    - docker run --rm alpine:3.20 echo "docker inside a job"
```

### Checkpoint 4

- [ ] Artifacts pass from build to test.
- [ ] You can explain which `rules:` condition made each job run or skip.

---

## Part 5 - Troubleshooting challenge (15 min)

Your instructor (or your partner) breaks one thing. Find and fix it using logs only.

Useful commands:

```bash
docker compose ps
docker compose logs --tail=100 gitlab-runner
docker compose exec gitlab-runner gitlab-runner verify
docker network inspect gitlab-lab-net
```

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Job stuck in **pending** | Tag mismatch, or runner offline | Match the `tags:` with the runner tags, or enable "Run untagged jobs". |
| `Could not resolve host: gitlab` while cloning | Missing `network_mode` or `clone_url` | Set both in `config.toml`, then restart the runner. |
| Runner shows **offline** | Wrong `url`, or GitLab not ready | Check `docker compose logs gitlab-runner` and the `url` value. |
| `permission denied` on `docker.sock` | Socket permissions on the host | Check `ls -l /var/run/docker.sock` on the host. |
| `pull access denied` or image not found | Typo in `image:` or no internet | Check the image name and the host's network access. |
| GitLab UI returns 502 | GitLab is still starting | Wait a few minutes and watch `docker compose logs -f gitlab`. |

### Instructor break ideas

1. Change `network_mode` to a non-existent network.
2. Change the job `tags:` to `dockr`.
3. Set `clone_url` to `http://localhost:8080`.
4. Stop the runner container.

---

## Part 6 - Cleanup and wrap-up (5 min)

Stop everything but keep data:

```bash
docker compose down
```

Remove everything, including GitLab data:

```bash
docker compose down
sudo rm -rf gitlab gitlab-runner
```

### Security note

Mounting `/var/run/docker.sock` gives the runner **root-equivalent access to the host**. Any pipeline author can start privileged containers, read host files, or stop other containers. This is fine for a private lab. For production, consider rootless Docker, a dedicated build VM, or the Docker Machine / autoscaling executors, and restrict who can run pipelines on the runner.

---

## Final self-check

1. Why does the runner need `clone_url`?
2. What is the difference between `--url` and `--clone-url`?
3. Where is the runner configuration stored?
4. What happens to a job container after the job finishes?
5. Why is mounting `docker.sock` risky?

## Repository layout (final)

```
gitlab-lab/
├── .env
├── docker-compose.yml
├── gitlab/              # created by GitLab (config, logs, data)
└── gitlab-runner/
    └── config/config.toml
```

Add a `.gitignore` containing `gitlab/`, `gitlab-runner/` and `.env` before you push this lab to GitHub, so that tokens and server data are never committed.

## References

- GitLab Docker install: <https://docs.gitlab.com/ee/install/docker/>
- GitLab Runner Docker executor: <https://docs.gitlab.com/runner/executors/docker.html>
- CI/CD YAML reference: <https://docs.gitlab.com/ee/ci/yaml/>
