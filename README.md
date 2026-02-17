# Docker Lightest Images Pipeline

Automated pipeline that:
1. Pulls 10 candidate Docker images from Docker Hub
2. Inspects their sizes, sorts ascending, keeps the **5 lightest**
3. Writes a `docker_image_report.json` output file
4. Commits and pushes **both** the playbook and the report to Git via Jenkins

---

## Files

| File | Purpose |
|---|---|
| `pull_lightest_images.yml` | Ansible playbook — pull & rank images |
| `Jenkinsfile` | Jenkins pipeline — orchestrate + push to Git |
| `docker_image_report.json` | Auto-generated output (committed by Jenkins) |

---

## Prerequisites

### On the Ansible/Jenkins agent
```bash
# Ansible 2.9+
pip install ansible

# Docker SDK for Python (required by community.docker collection)
pip install docker

# Install the Ansible Docker collection
ansible-galaxy collection install community.docker
```

### Jenkins credentials to create
Go to **Manage Jenkins → Credentials → Global** and add:

| ID | Type | Used for |
|---|---|---|
| `git-credentials` | Username/Password | Push to Git repo |
| `dockerhub-creds` | Username/Password | Docker Hub login (optional) |

---

## Customise the candidate image list

Edit `candidate_images` in `playbook.yml`:

```yaml
candidate_images:
  - "alpine:latest"
  - "busybox:latest"
  - "your-custom-image:tag"
  # add as many as you like — the playbook always picks the 5 smallest
```

---

## Run locally (without Jenkins)

```bash
# Basic run
ansible-playbook playbook.yml

# Override output path
ansible-playbook playbook.yml -e "output_report=/tmp/report.json"

# With Docker Hub login
ansible-playbook playbooks.yml \
  -e "docker_username=myuser" \
  -e "docker_password=mypass"
```

---

## Jenkins setup

1. Create a new **Pipeline** job
2. Point **Pipeline script from SCM** → your Git repo
3. Set **Script Path** to `Jenkinsfile`
4. Set parameters (or accept defaults):
   - `GIT_REPO_URL` — your repo URL
   - `GIT_BRANCH` — branch to commit to (default: `main`)
   - `DOCKER_LOGIN` — enable if using private images
5. Save and **Build Now** or let the nightly cron trigger it

---

## Output report structure (`docker_image_report.json`)

```json
{
  "generated_at": "2026-02-17T00:00:00Z",
  "total_candidates": 10,
  "lightest_5": [
    { "name": "busybox:latest",    "size_bytes": 4279234,  "size_mb": 4.08 },
    { "name": "alpine:latest",     "size_bytes": 7804559,  "size_mb": 7.44 },
    { "name": "debian:slim",       "size_bytes": 31234567, "size_mb": 29.79 },
    { "name": "nginx:alpine",      "size_bytes": 44234567, "size_mb": 42.19 },
    { "name": "python:3.12-alpine","size_bytes": 57234567, "size_mb": 54.59 }
  ],
  "all_images_sorted": [ ... ]
}
```
