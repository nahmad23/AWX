# AWX on Ubuntu 24.04 — Ansible Deployment

A single Ansible playbook that bootstraps an Ubuntu 24.04 server and stands up
AWX end to end:

1. **Bootstrap** — base packages, hosts entry for the AWX hostname.
2. **k3s** — single-node Kubernetes (keeps the built-in Traefik ingress + `local-path` storage).
3. **AWX Operator** — deployed via kustomize.
4. **AWX custom resource** — the AWX instance itself.
5. **Persistent storage** — `local-path` PVCs for Postgres and the projects directory.
6. **Ingress + HTTPS** — exposed through Traefik with a TLS secret (self-signed by default).

## Requirements

- Ubuntu 24.04 target with **≥ 2 vCPU and 8 GB RAM recommended** (4 GB is the bare minimum and may OOM).
- Ansible on the machine you run this from (the server itself is fine).
- No extra Galaxy collections — uses `ansible.builtin` + the bundled `k3s kubectl`.

## Usage

```bash
cd ansible

# 1. Set the admin password (kept out of git)
cp group_vars/vault.yml.example group_vars/vault.yml
#    edit group_vars/vault.yml -> set awx_admin_password
ansible-vault encrypt group_vars/vault.yml

# 2. Review group_vars/all.yml (hostname, versions, storage sizes)

# 3. Deploy
ansible-playbook deploy-awx.yml -e @group_vars/vault.yml --ask-vault-pass
```

The playbook is **idempotent** and **resumable** — if a step fails, fix the
cause and re-run; it picks up where it left off.

When it finishes: **https://<awx_hostname>** (default `awx.unitedlex.global`),
login `admin`.

## Configuration (`group_vars/all.yml`)

| Variable | Default | Purpose |
|----------|---------|---------|
| `awx_hostname` | `awx.unitedlex.global` | FQDN AWX is served on (Ingress host). |
| `awx_operator_version` | `2.19.1` | AWX Operator release. |
| `storage_class` | `local-path` | k3s dynamic provisioner (ReadWriteOnce). |
| `postgres_storage_size` / `projects_storage_size` | `8Gi` | PVC sizes. |
| `awx_tls_cert_src` / `awx_tls_key_src` | `""` | Paths to your own cert/key; empty = self-signed. |
| `ingress_class_name` | `traefik` | k3s default ingress controller. |

## HTTPS / certificate

- **Default: self-signed** cert generated for `awx_hostname` — correct for an
  internal host with no public DNS. Browsers will warn; that's expected.
- **Bring your own:** set `awx_tls_cert_src` and `awx_tls_key_src` in
  `group_vars/all.yml` to PEM files and re-run.
- Let's Encrypt (cert-manager) is intentionally not used here because the
  hostname isn't publicly resolvable for ACME validation.

## Accessing AWX

`awx_hostname` must resolve to the server's IP from wherever you browse:

- The playbook adds a hosts entry **on the server itself**.
- For your workstation, add DNS or an `/etc/hosts` line:
  `192.168.27.162  awx.unitedlex.global`

## Notes / gotchas baked in

- **kube-rbac-proxy image** — operator 2.19.1 points at the retired
  `gcr.io/kubebuilder` registry; it's redirected to
  `quay.io/brancz/kube-rbac-proxy` via kustomize.
- **RWO access mode** — `local-path` only supports `ReadWriteOnce`, so the
  projects PVC access mode is overridden from AWX's `ReadWriteMany` default
  (otherwise the PVC stays `Pending`).
- **First run is slow** — image pulls + Postgres init + migrations mean the
  AWX web pod can take 5–15 minutes to go Ready.
