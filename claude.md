# plane-ansible — Build Spec (v4)

## Purpose
Ansible repo that configures the EC2 instance: mounts/verifies the data volume,
installs Docker (if not already present from `user_data`), pulls SMTP credentials +
S3 bucket name from SSM Parameter Store, renders Plane's `.env`, and brings up the
Docker Compose stack.

## Change from v3: this repo is now always manually triggered
In v3, `plane-infra`'s `user_data` cloned this repo and ran it automatically at boot.
As of v4, `user_data` only installs prerequisites (git, ansible, docker) and mounts
the data volume — it does **not** clone this repo or invoke `ansible-playbook`. Every
run of this playbook, including the very first one on a freshly created instance, is
now a deliberate manual action:

```
aws ssm start-session --target <instance-id> --region us-east-1
sudo su -
git clone --branch <branch> <ansible_repo_url> /opt/provisioning
cd /opt/provisioning
ansible-playbook -c local -i localhost, site.yml
```

This means "local mode" (as described in prior spec versions) is still the correct
term for how the playbook connects (`ansible_connection: local`, run directly on the
box) — what's changed is only *who/what invokes it*: an operator by hand, not
`cloud-init`/`user_data` automatically. If the playbook run fails partway through
(bad SMTP creds, a typo in a template, a Docker Compose syntax error), the fix is
simply re-running `ansible-playbook` again on the same already-up instance — no
destroy/recreate cycle needed at the Terraform layer.

## Scope
- `site.yml` runnable in two modes:
  1. **Local mode** (primary, always manually invoked): connect via SSM Session
     Manager, clone this repo, run `ansible-playbook -c local -i localhost,
     site.yml` by hand
  2. **Remote mode** (secondary, for re-running from an operator's own machine
     instead of SSH-ing/SSM-ing in first): dynamic AWS inventory targeting
     instances tagged `Name = plane-app`, connected via `amazon.aws.aws_ssm`
     (never SSH, since no key pair exists)
- Idempotent role set: safe to run repeatedly against the same instance, whether
  it's a first-time configuration or a retry after a failure
- Mount verification for `/data` (already mounted by `user_data`, re-asserted here
  rather than assumed)
- Install Docker + Compose plugin if not already present (belt-and-suspenders —
  `user_data` should have already installed these, but this role doesn't assume
  that succeeded)
- Fetch SMTP username/password and S3 bucket name from SSM (`/plane/*` path)
- Render `.env` for Plane's Docker Compose stack (SMTP magic-code auth, S3 file
  storage, Postgres on `/data/postgres`, ephemeral Redis)
- Install and configure the CloudWatch agent
- Bring up the Plane Docker Compose stack, with a health check before reporting
  success

## Explicit non-goals
- No Google OAuth configuration anywhere
- No EC2/EBS/IAM/S3/Route53 resource creation
- No start/stop scheduling logic
- No secret values committed anywhere in this repo
- No SSH-based connection anywhere, including remote mode
- This repo does not get invoked automatically by anything — there is no
  `cloud-init`, systemd unit, or cron job anywhere in this repo or in
  `plane-infra` that triggers a run. Every execution is operator-initiated.

## Directory structure to generate

```
plane-ansible/
├── ansible.cfg
├── site.yml
├── inventory/
│   ├── local.yml
│   └── aws_ec2.yml
├── group_vars/
│   └── all.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   ├── docker/
│   │   └── tasks/main.yml
│   ├── data-mount/
│   │   └── tasks/main.yml
│   ├── cloudwatch-agent/
│   │   ├── tasks/main.yml
│   │   └── templates/cw-agent-config.json.j2
│   └── plane-bootstrap/
│       ├── tasks/main.yml
│       └── templates/
│           ├── env.j2
│           └── docker-compose.yml.j2
└── requirements.yml
```

## requirements.yml
- `amazon.aws` (for `ssm_parameter_info`, the `aws_ec2` dynamic inventory
  plugin, and the `aws_ssm` connection plugin used in remote mode)
- `community.docker` (for `docker_compose_v2`)
- `community.general`
- Note in the README: remote mode requires the `session-manager-plugin`
  binary installed on the operator's local machine (one-time setup, not
  automated by this repo)

## site.yml
```yaml
---
- hosts: all
  become: true
  roles:
    - common
    - docker
    - data-mount
    - cloudwatch-agent
    - plane-bootstrap
```
Unchanged in structure from v3 — this playbook doesn't know or care whether
it's being run by an operator who just SSM'd in, or remotely via dynamic
inventory. Same roles, same order, both cases.

## inventory/local.yml
```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```
This is what an operator uses after connecting via
`aws ssm start-session` and cloning the repo directly on the instance.

## inventory/aws_ec2.yml — remote mode, SSM connection
```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Name: plane-app
  instance-state-name: running
keyed_groups:
  - key: tags.Name
    prefix: tag
compose:
  ansible_connection: "'amazon.aws.aws_ssm'"
  ansible_aws_ssm_region: "'us-east-1'"
```
Used only if the operator prefers running Ansible from their own laptop
against the live instance rather than logging in first — functionally
equivalent to the local-mode flow, just triggered from a different machine.
Both paths are manual; neither is automatic.

## Role details
(Unchanged from v3 — see below for the abbreviated restatement since the only
structural change in this version is *how the playbook gets invoked*, not
what it does once running.)

### roles/common
Base packages, ensure `amazon-ssm-agent` is active, unattended security
upgrades, `dnf`-based (RHEL 9), UTC timezone.

### roles/docker
Verify/install Docker Engine + Compose plugin via the RHEL Docker CE repo.
Since `user_data` should already have done this, this role's tasks should be
written to detect the already-installed state cleanly (no errors on a
second, redundant install attempt) rather than assuming a bare OS.

### roles/data-mount
Idempotently verify `/data` is mounted (already done by `user_data`, but
re-checked here); create `/data/postgres` with correct ownership for the
Postgres container image in use.

### roles/cloudwatch-agent
Install/configure via the RHEL RPM package, CPU/memory/disk metrics, Docker
log collection.

### roles/plane-bootstrap
Create `/opt/plane`; fetch `/plane/smtp_username`, `/plane/smtp_password`,
`/plane/s3_bucket_name` from SSM with `with_decryption: true`; mark
secret-handling tasks `no_log: true`; render `.env` and
`docker-compose.yml.j2`; bring up the stack via `docker_compose_v2`; poll for
a healthy response before reporting success. Verify Plane's exact SMTP
environment variable names (or God Mode API path) against current
self-hosting docs at implementation time — same open item carried over from
prior versions.

### group_vars/all.yml
Non-secret config: Plane image tag/version, port mappings, S3 region, log
retention.

## Validation checklist for Claude Code to self-check after generating
- [ ] Nothing in this repo, and nothing referenced by `plane-infra`, causes
      this playbook to run automatically — confirm there's no systemd unit,
      cron entry, or `user_data` invocation anywhere in either repo
- [ ] `site.yml` runs correctly against both `inventory/local.yml` (after a
      manual SSM login + clone) and `inventory/aws_ec2.yml` (remote mode)
- [ ] No SSH connection configuration anywhere — `local` or
      `amazon.aws.aws_ssm` only
- [ ] `roles/docker` and `roles/cloudwatch-agent` handle the case where
      Docker is already installed (from `user_data`) without erroring
- [ ] The full playbook run is safe to re-run repeatedly on the same
      instance after a partial failure, with no leftover broken state from
      the failed attempt blocking a clean retry
- [ ] No Google OAuth references anywhere
- [ ] SMTP password is fetched from SSM at runtime only
- [ ] Redis has no persistent volume; Postgres data is bind-mounted to
      `/data/postgres`
