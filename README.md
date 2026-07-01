# plane-ansible

Ansible repo that configures a running EC2 instance to serve self-hosted
[Plane](https://plane.so). It mounts/verifies the `/data` volume, installs
Docker (if not already present), pulls SMTP credentials + the S3 bucket name
from SSM Parameter Store, renders Plane's `.env` + `docker-compose.yml`, brings
the stack up (bound to `127.0.0.1`), and puts **host nginx with Let's Encrypt
TLS** in front on 443.

This repo is **always manually triggered**. Nothing here (and nothing in
`plane-infra`) runs it automatically — no cloud-init, no systemd unit, no
cron. Every run, including the first on a fresh instance, is an operator
action.

## ⚠️ Precondition: DNS must resolve first

certbot issues the TLS cert via an HTTP-01 challenge, so **the GoDaddy A record
for `docs.joindevops.com` must already point at the instance's Elastic IP, and
port 80 must be open, before you run the playbook.** If DNS isn't ready, the
`nginx` role serves the site over plain HTTP and you simply re-run the playbook
once DNS propagates to flip it to HTTPS.

## Run it (local mode, primary)

SSH into the instance with PuTTY (the joindevops AMI uses username + password
auth — no key pair), clone this repo, run the playbook locally:

```bash
# PuTTY: Host = <elastic-ip>, Port = 22, then log in with the AMI user/password
git clone --branch main https://github.com/joindevopscloud/joindevops-plane-config.git /opt/provisioning
cd /opt/provisioning
ansible-galaxy collection install -r requirements.yml
sudo ansible-playbook -c local -i localhost, site.yml
```

`/opt/provisioning` is pre-owned by `ec2-user` (set by the infra `user_data`),
so the clone needs no sudo; the playbook itself is run with sudo.

## Remote mode (secondary)

`inventory/aws_ec2.yml` still exists for running from your own laptop over an
SSM connection, but note the instance is now SSH/password-based and the IAM
role no longer includes `AmazonSSMManagedInstanceCore`, so Session Manager /
`aws_ssm` connectivity is not guaranteed. Local mode above is the supported
path.

## Re-running / recovering from a partial failure

The whole role set is idempotent. If a run fails partway (bad SMTP creds, a
template typo, a Compose error), fix the cause and just run
`ansible-playbook` again on the same instance. No Terraform destroy/recreate
is needed.

## Secrets

- **SMTP username/password** and **S3 bucket name**: read from SSM Parameter
  Store (`/plane/*`) at runtime, `no_log: true`. Never committed.
- **Django `SECRET_KEY`** and **Postgres password**: generated once and stored
  under `/data/.plane-secrets/` on the persistent EBS volume, so they survive
  instance replacement (the Postgres password is baked into the DB on first
  init and must stay stable).

## Image versions

Images are **pinned to `v1.3.1`** (latest stable community release,
2026-05-14) in `group_vars/all.yml` — not the floating `stable` tag — so
re-runs are reproducible. The full `makeplane/*` community image inventory and
version history are tracked in memory (`project_plane_image_versions`). To
bump: change `plane_image_tag`, confirm the tag exists for every image, done.

## Topology (verified against Plane v1.3.1)

The compose was reconciled against the `makeplane/plane` **v1.3.1**
`docker-compose.yml`, `.env.example`, and `setup.sh`. Services:

| Service | Image | Notes |
|---------|-------|-------|
| `api` / `worker` / `beat-worker` / `migrator` | `plane-backend` | one image, four `docker-entrypoint-*.sh` commands |
| `web` | `plane-frontend` | |
| `space` | `plane-space` | |
| `admin` | `plane-admin` | God Mode / admin UI |
| `live` | `plane-live` | **required in v1.x** (live collaboration) |
| `proxy` | `plane-proxy` | Caddy; run HTTP-only, bound to `127.0.0.1:8080` |
| `plane-db` | `postgres:15.7-alpine` | bind-mounted to `/data/postgres` |
| `plane-redis` | `valkey/valkey:7.2.11-alpine` | ephemeral |
| `plane-mq` | `rabbitmq:3.13.6-management-alpine` | **required in v1.x**, ephemeral |

Dropped from upstream on purpose: **`plane-minio`** (we use external AWS S3).

### Resolved: SMTP is configured in God Mode, not env vars

v1.3.1's `.env.example` has **no** `EMAIL_*` variables and `setup.sh` injects
only `SECRET_KEY`. SMTP is configured at runtime in the **God Mode** admin UI
at `https://docs.joindevops.com/god-mode/`, using the credentials stored in
SSM (`/plane/smtp_username`, `/plane/smtp_password`). The playbook prints this
reminder at the end of a run. This retires the long-standing open item.

### S3 credentials via IMDS

The `api` container reads its S3 credentials from the EC2 instance role over
IMDSv2. Because containers are one network hop from the metadata endpoint,
`plane-infra` sets `http_put_response_hop_limit = 2` on the instance — without
it, uploads to S3 fail with credential errors.
