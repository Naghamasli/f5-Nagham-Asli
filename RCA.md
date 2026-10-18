# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
The proxy image failed during apt-get install.

Error: dpkg: error processing package nginx-common (--configure): end of file on stdin at conffile prompt.
### How I diagnosed it
Ran ./scripts/run-tests.sh and inspected the complete Docker build output.

Checked proxy/Dockerfile and noticed that the custom Nginx configuration was copied before installing Nginx.
### Root cause
The Dockerfile copied nginx.conf to /etc/nginx/nginx.conf before the Nginx package installation.

During installation, dpkg detected an existing configuration file and requested interactive input, which was unavailable during the Docker build.
### Fix
Reordered the Dockerfile instructions to install Nginx before copying the custom configuration.

Verified using docker compose build --no-cache proxy, which completed successfully.
---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
/nginx-health returned HTTP 200, but /api/info returned HTTP 502 Bad Gateway.

Nginx logged connect() failed (111: Connection refused) with upstream http://127.0.0.1:8080/api/info.
### How I diagnosed it
Examined the proxy logs and inspected proxy/nginx.conf.

Found that the upstream was configured as 127.0.0.1:8080.

Checked the Docker Compose service configuration.
### Root cause
Each Docker container has its own network namespace.

127.0.0.1 inside the proxy container refers to the proxy itself, not the backend container.
### Fix
Changed the upstream address to backend:8080, using Docker Compose's internal DNS.

Verified that the proxy resolved the backend address and could successfully forward /api/info requests once the backend was ready.
---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
Integration tests still returned HTTP 502 during cold startup.

The same tests passed when executed against already-running containers.
### How I diagnosed it
Started services using docker compose up -d --build backend proxy.

Checked backend logs and confirmed it was listening on port 8080.

Verified connectivity using docker compose exec proxy curl -i --max-time 5 http://backend:8080/health.

Ran docker compose run --rm test-runner while the services were already running. All tests passed.
### Root cause
A startup race condition.

The short form of depends_on controls container startup order but does not guarantee application readiness.

The test runner could start sending requests before the backend was ready.
### Fix, and why I chose it over the alternatives
Added a backend health check using Python's standard library and the /health endpoint.

Added a proxy health check using /nginx-health.

Updated Compose dependencies to use condition: service_healthy.

Chose health checks instead of a fixed sleep because they verify actual service readiness rather than assuming a specific startup time.

Verified with docker compose down followed by ./scripts/run-tests.sh. Both integration tests passed, and the test runner exited with code 0.
---

## Defect 4: CI pipeline fails

### Symptom / error output
The challenge identified a failing GitHub Actions workflow.

The workflow executes ./scripts/run-tests.sh directly, but the script initially lacked executable permission.
### How I diagnosed it
Inspected .github/workflows/ci.yml and scripts/run-tests.sh.

Compared bash ./scripts/run-tests.sh, which worked locally, with the direct execution used by CI.

Checked permissions using ls -l scripts/run-tests.sh.

The script initially had permissions -rw-r--r--.
### Root cause
The missing executable bit prevented direct script execution. This is a likely cause of the CI failure, pending confirmation from the GitHub Actions logs.
### Fix
Executed chmod +x scripts/run-tests.sh.

Used git update-index --chmod=+x scripts/run-tests.sh to preserve executable permission in Git.

Confirmed Git mode 100755.

Successfully ran ./scripts/run-tests.sh directly.

GitHub Actions verification is still pending.
---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

1. Reduce proxy image size

Original image: 422.46 MiB

Final image: 96.59 MiB

Reduction: 325.87 MiB (77.14%)

Changes:

Replaced ubuntu:24.04 with nginx:stable-alpine.

Removed unnecessary packages including Git, Vim, Htop, and Curl.

Replaced the Curl-based proxy health check with BusyBox wget.

Removed the unused default Nginx site configuration.

Verified that the optimized image builds and passes the integration tests.

2. Run the proxy as non-root

Used the existing nginx user (UID 101) from the Alpine image.

Added USER nginx to the Dockerfile.

Updated ownership of /var/cache/nginx to allow Nginx to use its temporary directories.

Configured the PID file at /tmp/nginx.pid, a writable location.

Verified the image's default user with docker run --rm --entrypoint id f5-nagham-asli-proxy:latest.

Confirmed that both integration tests still passed.

3. Additional security and operational improvements

Added server_tokens off to prevent disclosure of the Nginx version in HTTP response headers.

Verified that the response returned Server: nginx without a version number.

Removed the unused default site configuration.

Retained stdout/stderr logging for container observability.

Reduced unnecessary installed software by adopting the minimal Nginx image
>

---

## Anything you would do differently with more time
Add Dockerfile linting and image vulnerability scanning to CI.

Pin and regularly update the base image to improve build reproducibility.

Evaluate additional container hardening, such as dropping unnecessary Linux capabilities and using a read-only root filesystem.

Add repeated clean-start testing to further validate reliability.
