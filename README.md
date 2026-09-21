# settings
https://geba-trans.github.io/settings/

<!-- GEBA hand-off runbook -->

## GEBA operational hand-off

### Wat zijn de veelvoorkomende fouten en storingen, en hoe worden die normaal opgelost?

This repository contains configuration/bootstrap settings. Failures usually come from wrong environment values, missing secret, permissions or applying settings in the wrong order. Compare a change against the target environment before applying it.


<!-- GEBA git history bugfix review -->


#### Git history: bugfixes and incident-related changes
The following entries are taken from the available Git history. Where a commit references an issue or pull request, its title was looked up and included as the one-line error/change description.
- `d5025cc` (2025-03-25) — fix: update branding color variables in branding.css for consistency — **Description:** Commit description: fix: update branding color variables in branding.css for consistency
- `9d1ff67` (2025-03-24) — fix: define warning color variable in branding.css — **Description:** Commit description: fix: define warning color variable in branding.css

### Hoe en waar is de logging terug te vinden?

CI/CD output, deployment logs and the logs of the service consuming the settings; this repository has no standalone runtime log described.

### Hoe herstart of herstel je de applicatie?

There is no standalone application to start/stop. Apply settings through the consuming deployment, restart only affected services and validate configuration/health endpoints.

### Hoe worden updates doorgevoerd, en wat is er verder aan onderhoud nodig?

Review changes, protect secrets, test in staging and roll out atomically where possible. Maintain configuration ownership, secret rotation, audit history and rollback copies.

## Detailed Docker host recreation, start/stop and recovery

This section is the technical host-recreation procedure for this project. It is intentionally generic: the repository’s actual `Dockerfile`, `docker-compose*.yml`, `.env.example`, service names, ports, volumes and healthchecks are authoritative for this application.

### Scope and responsibility

This is technical documentation only. It does not describe the business process or provide functional acceptance criteria. Functional validation after a recreation must be performed by Jerry/functioneel beheer.

**Important backup responsibility:** GEBA Trans no longer manages application or infrastructure backups. **ZeroPlex manages the backups and must handle any data restore.** Do not independently restore databases, Docker volumes, uploads, queues or application data from an old file or volume. Contact ZeroPlex before any restore, replacement of a data volume, destructive migration or recovery from backup.

### 1. Decide whether a recreation is appropriate

A host recreation is destructive if it removes Docker volumes, bind-mounted data, local configuration or certificates. Before proceeding:

1. Confirm the affected environment and host: development, staging or production.
2. Record the incident, time, hostname, application version, current container status and error messages.
3. Confirm that the problem is a host/container problem and not an upstream API, database, DNS, certificate or credential problem.
4. Check whether the application is stateless. Treat every database, upload directory, queue, cache and named volume as persistent data until ZeroPlex confirms otherwise.
5. Stop and escalate before removing volumes or restoring data.
6. Obtain the correct repository branch, deployment approval, environment file and ZeroPlex recovery decision.

### 2. Linux host preparation

The following is a clean installation outline for a supported Ubuntu/Debian Linux host. Use the organization’s approved patching and access process in production.

```bash
# Check the host and remove conflicting legacy packages if approved
uname -a
cat /etc/os-release
sudo apt-get update
sudo apt-get remove -y docker.io docker-doc docker-compose podman-docker containerd runc || true

# Install prerequisites
sudo apt-get install -y ca-certificates curl git gnupg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository; choose the correct distribution for the host
printf 'deb [arch=%s signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu %s stable\n' \
  "$(dpkg --print-architecture)" "$(. /etc/os-release && echo "$VERSION_CODENAME")" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and verify the daemon
sudo systemctl enable --now docker
sudo systemctl is-active docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Optional non-root operation may be enabled only according to the host security policy:

```bash
sudo usermod -aG docker "$USER"
# Log out and in again before using Docker without sudo.
```

Do not expose Docker’s remote API to the network. Restrict SSH, Docker ports, application ports and database ports using the approved firewall and network policy. Check time synchronisation, DNS, disk space and available memory before deployment:

```bash
df -h
free -h
timedatectl
getent hosts <required-host>
sudo journalctl -u docker --since "1 hour ago"
```

### 3. Windows host preparation

For Windows, use Docker Desktop with the WSL 2 backend unless the organization has approved another runtime. Docker Desktop licensing must be checked for the organization and host type.

1. Install all approved Windows updates and reboot.
2. Enable virtualization in firmware and the Windows features **Virtual Machine Platform**, **Windows Subsystem for Linux**, and (where required) **Windows Hypervisor Platform**.
3. From an elevated PowerShell window, install/update WSL:

```powershell
wsl --install
wsl --update
wsl --set-default-version 2
wsl --status
```

4. Install Docker Desktop from the approved company software source, or with the approved package manager:

```powershell
winget install --id Docker.DockerDesktop
```

5. Start Docker Desktop and enable **Use the WSL 2 based engine**. Enable integration for the approved WSL distribution only.
6. Open a new PowerShell or WSL terminal and verify:

```powershell
docker version
docker compose version
docker run --rm hello-world
```

7. Keep source code in the WSL filesystem for Linux containers where possible, rather than a heavily shared Windows filesystem. Confirm that corporate VPN, DNS, proxy, certificate and firewall rules allow the required database and API connections.
8. Do not store production secrets in PowerShell history, `.env` files committed to Git, Docker labels or debug output.

### 4. Obtain the source and configuration

```bash
git clone <GEBA-Trans-repository-url>
cd <repository-directory>
git fetch --all --tags
git status
git branch -a
# Check out the approved release/branch; do not deploy an arbitrary feature branch.
git checkout <approved-branch-or-tag>
```

Review the repository before starting it:

```bash
find . -maxdepth 2 -iname 'docker-compose*.yml' -o -iname 'compose*.yml' -o -iname 'Dockerfile*'
grep -R "\${" -n --include='docker-compose*.yml' --include='compose*.yml' . || true
docker compose config --help
```

Use the project’s `.env.example` or documented environment template to create the environment file. Obtain secrets through the approved secret-management process. Never copy secrets from another environment without confirming host, database, tenant, certificate and endpoint values.

### 5. Validate the Compose configuration before starting

Run these checks from the directory containing the applicable Compose file:

```bash
docker compose -f docker-compose.<environment>.yml config
# Or, where the project uses the default file:
docker compose config
```

Before starting, verify:

- image names and tags are approved;
- build contexts and Dockerfiles exist;
- required environment variables are populated;
- host bind-mount paths exist and have correct permissions;
- published ports are available and not unintentionally public;
- database/API hostnames point to the correct environment;
- named volumes are understood and are not being replaced;
- healthchecks and restart policies are present or intentionally absent;
- TLS certificates and reverse-proxy routes match the environment;
- the application is not configured to write production data from a staging host.

### 6. Build and start the application

Use the project’s documented Compose file. Typical patterns are:

```bash
# Pull approved images when the project is image-based
docker compose -f docker-compose.<environment>.yml pull

# Build locally when the project contains a Dockerfile or build instructions
docker compose -f docker-compose.<environment>.yml build

# Start in the background
docker compose -f docker-compose.<environment>.yml up -d

# Check containers, health and published ports
docker compose -f docker-compose.<environment>.yml ps
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Some projects require a one-time setup, migration, seed, initialization or certificate step. Follow that project’s README and execute it only after confirming the target environment. A database migration is not a substitute for a backup; if data restoration is involved, stop and contact ZeroPlex.

### 7. Smoke test after startup

Perform technical checks before handing the application back:

```bash
# Follow startup logs without exposing them publicly
docker compose -f docker-compose.<environment>.yml logs --tail=200

# Follow one service during diagnosis
docker compose -f docker-compose.<environment>.yml logs -f <service>

# Test the documented health endpoint when available
curl -fsS https://<application-host>/health
```

Then verify, as technically applicable:

- all required containers are running and healthy;
- no restart loop is present;
- DNS and TLS resolve correctly;
- the application can reach its database and required APIs;
- queues/workers are connected;
- metrics and logs are being emitted;
- the reverse proxy routes to the expected container;
- no unexpected data migration, truncation or replay occurred.

Functional users and Jerry/functioneel beheer must perform the functional acceptance check.

### 8. Normal stop, start and restart

Use a graceful service operation first:

```bash
# Stop without removing containers or volumes
docker compose -f docker-compose.<environment>.yml stop

# Start existing containers again
docker compose -f docker-compose.<environment>.yml start

# Restart one service when possible
docker compose -f docker-compose.<environment>.yml restart <service>

# Stop and recreate containers while retaining named volumes
# Use only after checking the project’s dependencies.
docker compose -f docker-compose.<environment>.yml down
docker compose -f docker-compose.<environment>.yml up -d
```

Do **not** use `docker compose down -v`, `docker volume rm`, `docker system prune --volumes`, database drop commands or filesystem deletion as routine troubleshooting. Those actions can destroy data and require ZeroPlex involvement.

### 9. Updates and recreation

For an update:

1. Record the running image digest, Git commit, environment and current status.
2. Export/document the current application configuration and flow where the project requires it; do not export secrets into a public location.
3. Confirm the change and rollback plan.
4. Pull or build the approved version.
5. Validate Compose configuration.
6. Recreate only the necessary services.
7. Run technical smoke tests and have Jerry/functioneel beheer validate the function.
8. Monitor logs and health for an agreed observation period.

Typical update commands are:

```bash
git fetch --all --tags
git checkout <approved-release>
docker compose -f docker-compose.<environment>.yml config
docker compose -f docker-compose.<environment>.yml pull
docker compose -f docker-compose.<environment>.yml up -d --remove-orphans
docker compose -f docker-compose.<environment>.yml ps
docker compose -f docker-compose.<environment>.yml logs --tail=200
```

Do not add `--force-recreate`, `--build`, destructive migrations or volume removal automatically. Use them only when the application’s README and change procedure require them.

### 10. Recovery and data restore

If containers or the host fail, first preserve evidence: status output, logs, timestamps, image versions and configuration names. Recreate stateless containers only after checking mounts and dependencies.

For any of the following, stop and contact **ZeroPlex** before proceeding:

- restoring a database or Docker volume;
- restoring uploads, files, flow exports or queues;
- recovering after disk loss, host loss or ransomware;
- rolling back data after a failed migration;
- replacing a persistent volume;
- recovering secrets or certificates from backup.

**GEBA Trans no longer manages backups. ZeroPlex manages the backups and must execute or supervise data restoration.** Kai’s role is to provide technical information, validate the restored host/container configuration and perform technical smoke tests. Jerry/functioneel beheer validates the business result.

### 11. Troubleshooting evidence to capture

For an incident, capture the following without including secrets or personal data in an unsecured ticket:

```bash
date -Is
git rev-parse --short HEAD
docker version
docker compose version
docker compose -f docker-compose.<environment>.yml ps
docker compose -f docker-compose.<environment>.yml logs --since=30m --no-color
sudo journalctl -u docker --since "30 minutes ago" --no-pager
df -h
```

Also record the host, environment, service name, endpoint, request/job/order identifier, first error, last known good version and whether the issue is reproducible. Redact passwords, tokens, cookies, authorization headers, private keys and sensitive payloads.

### 12. Completion criteria

A technical recreation is complete only when:

- the approved code/image and environment are recorded;
- Compose configuration validates;
- containers remain healthy without restart loops;
- logs are available;
- health and dependency checks pass;
- backup/restore responsibility is confirmed with ZeroPlex where relevant;
- no destructive cleanup was performed without approval;
- Jerry/functioneel beheer has completed functional validation;
- the incident/change record contains the final version and any follow-up work.
