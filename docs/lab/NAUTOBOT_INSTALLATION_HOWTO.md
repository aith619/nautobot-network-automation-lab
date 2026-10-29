# Nautobot 3.2.6 — Ubuntu Docker Lab Installation & Troubleshooting HOWTO

**Lab:** Nautobot Network Automation Lab  
**Platform:** Ubuntu VM running on VMware  
**Application:** Nautobot 3.2.6  
**Source project:** `nautobot/nautobot-docker-compose`  
**Personal GitHub repository:** https://github.com/aith619/nautobot-network-automation-lab

## 1. What Are We Building?

We are building a Nautobot server that can store information about network devices, interfaces, IP addresses, VLANs, and other network infrastructure.

Think of Nautobot as a digital filing cabinet for a network.

Our lab uses several pieces:

| Component | What it does |
|---|---|
| Ubuntu | The operating system hosting our lab |
| Docker | Runs applications in isolated containers |
| Docker Compose | Starts multiple related containers together |
| Nautobot | The network source-of-truth application |
| PostgreSQL | Stores Nautobot's data |
| Redis | Supports caching and background tasks |
| Celery Worker | Processes background jobs |
| Celery Beat | Schedules recurring background work |
| Git | Tracks changes to our lab files |
| GitHub | Stores and shares our project |

**Important:** This is a development and learning environment, not a production deployment.

---

## 2. Our Lab Environment

The following details were observed during installation.

| Setting | Lab value |
|---|---|
| Working directory | `~/network-automation/nautobot-docker-compose` |
| Docker Compose directory | `environments/` |
| Python virtual environment | `.venv` |
| Host Python requirement | `>=3.10,<3.15` |
| Configured Python version | `3.12` |
| Nautobot requirement | `3.2.6` |
| VM IP address during setup | `192.168.159.129` |
| Nautobot web port | `8080` |
| Personal GitHub remote | `origin` |
| Official Nautobot remote | `upstream` |

The IP address is specific to this VMware lab. Another VM may have a different address.

To discover the VM's IP addresses:

```bash
hostname -I
```

---

## 3. Python Environment and Invoke

The Nautobot Docker Compose project uses Python-based development tasks.

We worked inside an activated virtual environment, indicated by:

```text
(.venv)
```

A virtual environment separates the Python packages used by this project from other Python projects.

### Problem: Missing TOML module

We encountered:

```text
ModuleNotFoundError: No module named 'toml'
```

This meant the Python interpreter could not find a library required by the project's `tasks.py` file.

### Fix executed

```bash
python -m pip install invoke toml
```

### Verification executed

```bash
python -c "import invoke, toml; print('Invoke and TOML imports: PASS')"
```

Result:

```text
Invoke and TOML imports: PASS
```

We then listed the available development tasks:

```bash
python -m invoke --list
```

Tasks included:

- `build`
- `start`
- `stop`
- `restart`
- `createsuperuser`
- `db-export`
- `db-import`
- `migrate`
- `nb-shell`

### Why this mattered

Before running the Docker build commands, we needed the Python task runner to work.

A missing Python dependency can stop the setup process before Docker even begins building the application.

### Nonblocking warnings

We also saw warnings about invalid escape sequences such as `\$` inside `tasks.py`.

These were Python syntax warnings, not the cause of our Docker startup failure.

---

## 4. Docker Permissions

### Problem

When checking Docker, we encountered:

```text
permission denied while trying to connect to the Docker API
at unix:///var/run/docker.sock
```

Docker uses a Unix socket to communicate with its background daemon.

The Ubuntu user initially did not have permission to access that socket.

### Explanation

Imagine Docker as a workshop with a locked door.

The Docker daemon is the workshop. The Unix socket is its door. The Linux user needs permission to enter.

This is a host operating-system permission issue, not a Nautobot problem.

### Recovery guidance

For an authorized local administrator, a common setup is:

```bash
sudo usermod -aG docker "$USER"
```

The user must then start a new login session before group membership normally takes effect.

Verify with:

```bash
docker info
```

**Security note:** Membership in the Docker group grants powerful host-level privileges. Only trusted users should receive it.

The original successful permission-change command was not preserved in the available screenshots, so the command above is documented as recovery guidance rather than a confirmed historical command.

---

## 5. Building the Nautobot Docker Image

Once Docker access worked, we built the Nautobot image:

```bash
python -m invoke build
```

The build downloaded base images, installed Python packages, copied application files, and created a local Docker image.

Our build completed successfully:

```text
nautobot Built
```

### What is a Docker image?

An image is like a prepared lunchbox.

It contains the software and dependencies needed to run the application.

A container is what happens when Docker opens that lunchbox and runs its contents.

### Important distinction

A successful Docker image build does not guarantee that the application will start successfully.

We learned this during the next stage.

---

## 6. Docker Compose Credentials

### Problem: Invalid Redis template

During startup, Docker Compose reported an invalid template involving a Redis URL:

```text
Invalid template: "redis://..."
```

The problem involved Docker Compose interpreting dollar signs inside an environment-file value.

Compose uses `$` for variable substitution.

### Why this happened

A password or URL containing a dollar sign may be interpreted as a Compose variable rather than literal text.

This can break configuration parsing before the containers start.

### What we learned

- Shell variables and Compose variables are not always interpreted the same way.
- Passwords containing special characters need careful handling.
- `env_file` and Compose's own variable interpolation serve different purposes.
- We must never publish the real `creds.env` file to GitHub.

### Safe practice

Keep real credentials in:

```text
environments/creds.env
```

Do not include real passwords in documentation or committed examples.

Use placeholders such as:

```text
NAUTOBOT_REDIS_PASSWORD=REPLACE_WITH_PRIVATE_PASSWORD
```

The exact successful Redis credential edit is not fully preserved in the screenshots, so it should not be reconstructed as though it were an executed command.

---

## 7. Containers Started, Then Exited

After Docker downloaded the required images, we saw containers being created for:

- PostgreSQL
- Redis
- Nautobot
- Celery Worker
- Celery Beat

Initially, some services exited with status code `1`.

To inspect the problem, we used:

```bash
docker ps -a
```

And examined logs with commands such as:

```bash
docker logs --tail 60 nautobot_docker_compose-nautobot-1
```

### What does `Exited (1)` mean?

It means the container's main process stopped with an error.

Docker itself may be working perfectly while the application inside a container is failing.

This distinction is important during troubleshooting.

---

## 8. Python `charset-normalizer` Failure

### Problem

The Nautobot and Celery logs showed:

```text
AttributeError: module 'charset_normalizer.md'
has no attribute 'CharInfo'
```

The traceback passed through Python's `requests` package into `charset_normalizer`.

This prevented Nautobot from starting.

### Why it mattered

Nautobot depends on Python packages, which may depend on additional packages.

In this case, importing `requests` failed because its character-encoding dependency failed to initialize.

### Investigation executed

We inspected package versions in the Docker image.

Observed versions included:

```text
charset-normalizer 3.4.4
requests 2.34.2
```

We also tested importing the module directly.

The import failed with the same `CharInfo` error.

### First repair attempt

We tried reinstalling `charset-normalizer` inside a temporary container.

The attempt failed because the container was not running as root:

```text
PermissionError: [Errno 13] Permission denied:
'/usr/local/bin/normalizer'
```

### Why that happened

The container's default user did not have permission to replace system-level Python package files.

### Successful temporary test

We reran the repair using:

```bash
docker run --rm \
  --user 0 \
  --entrypoint /bin/sh \
  docker.io/yourrepo/nautobot-docker-compose:local \
  -c '
    python -m pip uninstall -y charset-normalizer &&
    python -m pip install --no-cache-dir \
      --no-binary=charset-normalizer charset-normalizer==3.4.4 &&
    python -c "import requests; print(\"SUCCESS: requests imports correctly!\")"
  '
```

Result:

```text
SUCCESS: requests imports correctly!
```

### Why this fixed the test

The package was reinstalled from its source distribution rather than the prebuilt binary package.

This avoided the failing compiled component in our environment.

### Important lesson

A temporary container disappears after it exits.

Therefore, repairing a package inside a temporary container is useful for testing, but it is not a permanent solution.

We needed to create a corrected Docker image.

---

## 9. Building the Permanent Fixed Image

We created a custom Dockerfile:

```text
Dockerfile.charset-fix
```

The Dockerfile built on top of the original Nautobot image and installed the corrected dependency.

The resulting image was tagged:

```text
nautobot-fixed:local
```

### Verification executed

We tested the corrected image by importing the dependencies:

```bash
docker run --rm \
  --entrypoint python \
  nautobot-fixed:local \
  -c "import requests; import nautobot; print('PASS: Nautobot dependencies load')"
```

Result:

```text
PASS: Nautobot dependencies load
```

### Why this was better

A Dockerfile documents the fix as part of the image-building process.

This makes the change repeatable.

Instead of repairing a running container manually every time, we can rebuild the corrected image.

The exact Dockerfile is included separately in this Git repository.

---

## 10. Using the Fixed Image in Docker Compose

The original Compose configuration referenced the original Nautobot image.

We needed Nautobot and its background workers to use our corrected image instead.

We created:

```text
environments/docker-compose.fixed.yml
```

The override specified:

```yaml
services:
  nautobot:
    image: nautobot-fixed:local
    build: !reset null

  celery_worker:
    image: nautobot-fixed:local

  celery_beat:
    image: nautobot-fixed:local
```

### Why all three services?

Nautobot, Celery Worker, and Celery Beat run Python code from the Nautobot application environment.

If only the main web container used the fixed image, background services could still crash with the same dependency error.

### Why `build: !reset null`?

The base Compose configuration contained image-building instructions.

The override was intended to use our already-built corrected image rather than rebuild the original image.

This configuration is specific to the Compose version and files used in our lab.

---

## 11. PostgreSQL Compatibility Failure

### Problem

After fixing the Python dependency, Nautobot encountered a database compatibility error:

```text
django.db.utils.NotSupportedError:
PostgreSQL 14 or later is required (found 13.23).
```

### Explanation

Nautobot 3.2.6 requires a supported PostgreSQL version.

Our database was running PostgreSQL 13.23, which was too old.

Think of this as trying to open a new application with an outdated database engine.

The application can connect, but it refuses to operate against an unsupported database version.

### What we changed

We modified:

```text
environments/docker-compose.postgres.yml
```

to use a supported PostgreSQL image.

### Important database warning

Changing a PostgreSQL image tag does not automatically upgrade an existing database's files.

An old PostgreSQL 13 data directory cannot simply be treated as a PostgreSQL 14+ data directory.

For this disposable learning lab, we stopped the stack and removed its old database volume.

**Never delete a database volume containing important data without a verified backup and migration plan.**

### Commands executed during the lab reset

We stopped the Compose stack:

```bash
docker compose down
```

We listed matching database volumes:

```bash
docker volume ls --filter name=postgres_data
```

Two volumes were visible because earlier Compose commands had used different project names.

We removed the old lab database volume:

```bash
docker volume rm environments_postgres_data
```

Then restarted the stack using the intended configuration.

### What we learned

Docker volumes are persistent storage.

Deleting a container does not necessarily delete its data.

Deleting a database volume can permanently erase the database stored there.

---

## 12. Duplicate Docker Compose Projects

During troubleshooting, we encountered two sets of container names:

```text
nautobot_docker_compose-...
environments-...
```

### Why this happened

Docker Compose uses a project name to group related containers, networks, and volumes.

Running Compose from a different directory or with a different project name can create a second stack.

### Why this matters

Two stacks can cause confusion:

- One may contain the old database.
- Another may contain a fresh database.
- Both may attempt to use the same host port.
- Troubleshooting the wrong stack can waste time.

### Useful commands

```bash
docker compose ls -a
```

```bash
docker ps -a
```

```bash
docker volume ls
```

### Lesson

Use one consistent Compose project and file combination.

Do not delete a volume just because its name looks unfamiliar.

First identify which stack owns it and whether it contains important data.

---

## 13. Starting the Working Stack

From the `environments` directory, we used:

```bash
docker compose \
  -f docker-compose.postgres.yml \
  -f docker-compose.base.yml \
  -f docker-compose.local.yml \
  -f docker-compose.fixed.yml \
  up -d --no-build
```

### What these flags mean

| Option | Explanation |
|---|---|
| `-f` | Selects a Compose configuration file |
| Multiple `-f` options | Combines the selected configurations |
| `up` | Creates and starts services |
| `-d` | Runs services in the background |
| `--no-build` | Uses existing images without rebuilding |

### Successful startup

The terminal reported:

```text
Container environments-nautobot-1 Healthy
Container environments-celery-worker-1 Started
Container environments-celery-beat-1 Started
```

We verified the containers using:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

The Nautobot container published port 8080:

```text
0.0.0.0:8080->8080/tcp
```

This means Docker forwards connections from port 8080 on the Ubuntu VM to port 8080 inside the Nautobot container.

---

## 14. Finding the VM IP Address

We used:

```bash
hostname -I
```

The VM returned several addresses, including:

```text
192.168.159.129
172.17.0.1
172.18.0.1
172.19.0.1
```

### Which address should we use?

The `192.168.159.129` address belonged to the VMware lab network.

The `172.x.x.x` addresses were associated with Docker networks.

For browser access from our lab computer, we used:

```text
http://192.168.159.129:8080
```

### Important lesson

Not every IP address shown by `hostname -I` is appropriate for browser access.

Choose an address reachable from the computer running the browser.

---

## 15. Troubleshooting Browser Connection Failures

### Initial problem

The browser displayed:

```text
ERR_CONNECTION_REFUSED
```

Even though the container appeared to be running.

### Host-side test

We executed:

```bash
curl -v --max-time 5 http://127.0.0.1:8080/
```

The connection was established, but then reset:

```text
Recv failure: Connection reset by peer
```

### Container-side test

We also tested whether Nautobot was listening inside its container.

The test returned:

```text
127.0.0.1:8080 FAILED: [Errno 111] Connection refused
0.0.0.0:8080 FAILED: [Errno 111] Connection refused
```

### What this told us

Docker had published the port, but the application was not yet accepting connections on that port.

A healthy-looking container and a published Docker port do not, by themselves, prove the web application is listening.

### Log investigation

The Nautobot logs eventually showed:

```text
Nautobot 3.2.6 initialized!
```

and:

```text
Starting development server at http://0.0.0.0:8080/
```

The application subsequently became reachable.

### Why `0.0.0.0` matters

Listening on `127.0.0.1` inside a container restricts the application to the container's loopback interface.

Listening on `0.0.0.0` allows it to accept connections through its container network interfaces.

That is necessary for normal Docker port publishing.

---

## 16. Nautobot Login and Admin Account

After the application became reachable, the browser displayed Nautobot's login page.

Our first login attempt using the username:

```text
admin
```

failed.

### Error displayed

```text
Please enter a correct username and password.
```

We investigated whether the admin user existed in the database.

### Database check executed

```bash
docker exec environments-nautobot-1 \
  nautobot-server shell --command \
  "from django.contrib.auth import get_user_model; U=get_user_model(); print('Admin exists:', U.objects.filter(username='admin').exists())"
```

Result:

```text
Admin exists: False
```

### Why this mattered

The login was not necessarily failing because we typed the wrong password.

The database did not yet contain the `admin` account.

### Creating an administrator

The standard interactive Nautobot management command is:

```bash
docker exec -it environments-nautobot-1 \
  nautobot-server createsuperuser
```

Choose a username, email address, and strong private password when prompted.

Do not store that password in this document or commit it to Git.

The exact successful account-creation command was not captured in the available screenshots, but the final dashboard confirmed that we successfully logged in as `admin`.

### Final result

The Nautobot dashboard loaded successfully.

We could see sections including:

- Devices
- IPAM
- Organization
- Circuits
- Virtualization
- Cloud
- Jobs
- Apps

**The installation was successful.**

---

## 17. Understanding the Empty Nautobot Dashboard

Our new Nautobot instance displayed zero devices, locations, prefixes, and other objects.

This was expected.

We had installed the application but had not yet populated its network inventory.

Think of this as installing a new database application and opening an empty database.

The software is ready; the network data comes next.

Future exercises will populate Nautobot with devices, interfaces, IP addresses, VLANs, and automation-related data.

---

## 18. Git and GitHub Setup

We created our personal repository:

https://github.com/aith619/nautobot-network-automation-lab

The original cloned repository initially used:

```text
origin -> https://github.com/nautobot/nautobot-docker-compose.git
```

We wanted our own repository to become the default push destination while preserving the original project as a reference.

### Commands executed

```bash
git remote rename origin upstream
git remote rename mylab origin
```

### Final configuration

```text
origin   https://github.com/aith619/nautobot-network-automation-lab.git
upstream https://github.com/nautobot/nautobot-docker-compose.git
```

### What the names mean

`origin` is our personal GitHub repository.

`upstream` is the original Nautobot repository.

This lets us publish our own changes without confusing our repository with the original project.

---

## 19. GitHub Personal Access Token Troubleshooting

Our first push failed with:

```text
refusing to allow a Personal Access Token to create or update
workflow `.github/workflows/ci.yml`
```

### Why this happened

The original Nautobot repository contains GitHub Actions workflow files.

Our fine-grained GitHub token had permission to write repository contents, but it initially lacked the required workflow permission.

### Fix

In GitHub's personal access token settings, we confirmed access to:

```text
aith619/nautobot-network-automation-lab
```

We then updated the token permissions to allow the necessary repository and workflow operations.

### Successful push

We retried:

```bash
git push -u origin main
```

Result:

```text
[new branch] main -> main
branch 'main' set up to track 'origin/main'.
```

This confirmed that the GitHub upload succeeded.

**Never paste a personal access token into a HOWTO, source file, or Git commit.**

---

## 20. Files Created or Modified During Troubleshooting

These files are part of the actual lab:

| File | Purpose |
|---|---|
| `Dockerfile.charset-fix` | Builds an image with the Python dependency repair |
| `environments/docker-compose.fixed.yml` | Overrides services to use the fixed image |
| `environments/docker-compose.postgres.yml` | Configures the PostgreSQL service |
| `docs/lab/NAUTOBOT_INSTALLATION_HOWTO.md` | Documents installation and troubleshooting |

The first three files were part of our Docker troubleshooting work.

This Markdown document explains how we got the lab running.

---

## 21. Useful Troubleshooting Commands

### List running containers

```bash
docker ps
```

### List all containers, including stopped ones

```bash
docker ps -a
```

### Check container logs

```bash
docker logs --tail 60 environments-nautobot-1
```

### Find the Ubuntu VM IP address

```bash
hostname -I
```

### Test the local Nautobot web port

```bash
curl -v --max-time 5 http://127.0.0.1:8080/
```

### List Docker volumes

```bash
docker volume ls
```

### List Docker Compose projects

```bash
docker compose ls -a
```

### Show Git changes

```bash
git status --short
```

### Show GitHub repository destinations

```bash
git remote -v
```

---

## 22. Security Rules for This Lab

1. Never commit `creds.env` or files containing real passwords.
2. Never commit GitHub personal access tokens.
3. Never publish private API tokens or Nautobot secret keys.
4. Never delete a Docker database volume without understanding what it contains.
5. Do not expose the development server directly to the public internet.
6. Review changes before every Git commit.
7. Keep the GitHub repository private until its contents have been reviewed for sensitive information.

---

## 23. What We Learned

This installation demonstrated several important engineering principles.

**A successful build is not the same as a working application.** Docker can build an image successfully even when Python dependencies fail at runtime.

**Containers depend on other services.** Nautobot requires a compatible database and working supporting services.

**Logs are essential.** The real problem was often visible in the traceback, not in the container status.

**Persistence matters.** Docker volumes can outlive containers and can contain important database information.

**Version compatibility matters.** Nautobot 3.2.6 could not use the PostgreSQL 13.23 database we initially had.

**Reproducibility matters.** Building a fixed image through a Dockerfile is more reliable than manually repairing a temporary container.

**GitHub permissions matter.** A token that can write code may still lack permission to update GitHub Actions workflows.

**Documentation matters.** Every failure and correction is an opportunity to make the next installation easier.

---

## 24. Final Lab Status

At the end of this installation:

- Nautobot 3.2.6 was running.
- The Nautobot web interface was reachable on port 8080.
- The admin login succeeded.
- PostgreSQL and Redis were operational.
- The corrected Nautobot image was being used.
- Our GitHub repository was configured and the initial push succeeded.

**Next phase:** Populate Nautobot with our network lab infrastructure and begin network automation exercises.

---

## Documentation Notes

This guide is based on the previously prepared HOWTO and the installation screenshots and terminal outputs shared during the lab.

Commands and results confirmed in the conversation are documented as executed. Where an exact historical command was not available, the guide provides an explicitly identified standard recovery procedure rather than inventing a transcript.

Real passwords, personal access tokens, database credentials, and private keys have intentionally been excluded.

**Project:** Nautobot Network Automation Lab  
**Maintainer:** aith619  
**GitHub:** https://github.com/aith619/nautobot-network-automation-lab
