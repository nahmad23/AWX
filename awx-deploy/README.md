# AWX Deployment + Ping Job

Automation to **deploy AWX** on an Ubuntu server, register **10 managed servers**
in an AWX inventory, and create/launch a **ping job** against them.

> **Why a package instead of a live deploy?** Your AWX server
> (`192.168.27.162`) is on a private LAN that the Claude Code cloud sandbox
> cannot reach. Run these playbooks yourself from a machine that *can* reach
> the server (or from the server itself).

## What gets deployed

- **AWX** on Ubuntu via **k3s** (single-node Kubernetes) + the **AWX Operator**
  (the upstream-supported install method).
- Web UI at `http://192.168.27.162:30080` (login `admin`).
- An AWX **inventory** ("UnitedLex Servers") containing all 10 hosts.
- A **machine credential**, a **project** (this git repo), and a
  **job template** ("Ping All Servers") that runs `ansible.builtin.ping`.

## Prerequisites

- A control machine with Ansible (`pip install ansible`) and SSH access to
  `192.168.27.162`.
- Collections: `ansible-galaxy collection install -r requirements.yml`

## Setup

```bash
cd awx-deploy

# 1. Provide secrets (kept out of git)
cp group_vars/secrets.yml.example group_vars/secrets.yml
#   edit group_vars/secrets.yml -> set passwords / SSH key
ansible-vault encrypt group_vars/secrets.yml

# 2. Install AWX on the Ubuntu server (takes ~10-15 min)
ansible-playbook playbooks/install-awx.yml \
    -e @group_vars/secrets.yml --ask-vault-pass

# 3. Configure AWX + launch the ping job
ansible-playbook playbooks/configure-awx.yml \
    -e @group_vars/secrets.yml --ask-vault-pass
```

## Files

| Path | Purpose |
|------|---------|
| `inventory/awx-host.ini` | The Ubuntu server that hosts AWX (`192.168.27.162`). |
| `inventory/managed-hosts.yml` | The 10 servers AWX manages. |
| `group_vars/all.yml` | Non-secret config (versions, names, host list). |
| `group_vars/secrets.yml.example` | Template for passwords/keys (copy → encrypt). |
| `playbooks/install-awx.yml` | Installs k3s + AWX Operator + AWX. |
| `playbooks/ping.yml` | The playbook the ping job runs. |
| `playbooks/configure-awx.yml` | Creates inventory/credential/project/job template & launches it. |

## Security notes

- **No credentials are committed.** `secrets.yml` is git-ignored; supply
  passwords via `ansible-vault`.
- The weak default root password used during setup should be **rotated**.
- For production, put a TLS reverse proxy in front of the AWX NodePort.

## Notes

- `server3` was corrected to `server3.unitedlex.global` (the original list
  was missing the dot).
- The ping job only succeeds once the 10 target hosts are reachable by SSH
  from the AWX server with the credential you configure.
