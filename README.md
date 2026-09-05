# Forgejo on Synology via Ansible

Idempotent Ansible role and playbook to deploy [Forgejo](https://forgejo.org/) on a Synology NAS with Container Manager (Docker Compose v2). Designed so the tracked tree can be published without your homelab identity or secrets.

## What it converges

- Install directories and `docker-compose.yml` under a configurable path (default `/volume1/docker/forgejo`)
- Forgejo container with SQLite, healthcheck, and host ports for HTTP + Git SSH
- Instance config via env (`DOMAIN`, `ROOT_URL`, `SSH_PORT`, `INSTALL_LOCK`, registration policy)
- Optional administrator user via `gitea admin user create` (idempotent)

Dual `GITEA__*` / `FORGEJO__*` environment keys keep tag `10` and newer images aligned.

## Layout

```text
.
├── LICENSE
├── README.md
├── ansible.cfg
├── site.yml
├── inventory.ini.example
├── requirements.txt              # ansible-core
├── collections/requirements.yml  # community.docker
├── group_vars/all/
│   ├── vars.yml.example
│   └── vault.yml.example
└── roles/forgejo/
    ├── defaults/main.yml
    ├── meta/main.yml
    ├── tasks/{main,preflight,deploy,admin}.yml
    └── templates/docker-compose.yml.j2
```

Local files (gitignored after you copy the examples): `inventory.ini`, `group_vars/all/vars.yml`, `group_vars/all/vault.yml`.

## Prerequisites

**Controller**

```bash
uv python install 3.12
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

**Synology NAS**

- Container Manager installed and running
- SSH enabled for your deploy user
- Python 3.9+ from Package Center (Ansible target interpreter)
- SSH host key accepted once from the controller (`ssh user@nas`)

## Configure a private site

```bash
cp inventory.ini.example inventory.ini
cp group_vars/all/vars.yml.example group_vars/all/vars.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
```

Edit `vars.yml`:

| Variable | Purpose |
|---|---|
| `ansible_host` / `ansible_user` | NAS connection |
| `ansible_python_interpreter` | Usually `/var/packages/Python3.9/target/usr/bin/python3` |
| `forgejo_domain` | Hostname used in clone URLs and `ROOT_URL` |
| `forgejo_web_port` / `forgejo_ssh_port` | Host publish ports (SSH often `2222` because DSM owns `22`) |
| `forgejo_ssh_listen_port` | In-container SSH listen port (default `2222`; must be `>1023` when `forgejo_user_uid` is non-root) |
| `forgejo_user_uid` / `forgejo_user_gid` | Data volume ownership (often `1026` / `100`) |
| `forgejo_admin_username` / `forgejo_admin_email` | Admin created when install lock is on |

Edit `vault.yml` and set `vault_forgejo_admin_password`. Encrypt if you want:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Prefer leaving `vault.yml` untracked (default `.gitignore`). Only commit an encrypted vault if you also verify CI rejects plaintext (`$ANSIBLE_VAULT` header).

## Deploy

Become is required for Docker on stock DSM (`/var/run/docker.sock` is `root:root` `0660`). Scope sudo to the Docker CLI rather than `NOPASSWD: ALL`:

```text
your_ssh_user ALL=(ALL) NOPASSWD: /usr/local/bin/docker
```

```bash
# First deploy (vault + become prompts)
ansible-playbook site.yml --ask-vault-pass -K

# Dry run (safe on a blank NAS: skips template/compose until the install path exists)
ansible-playbook site.yml --check --diff --ask-vault-pass -K
```

If `vault.yml` is not encrypted, omit `--ask-vault-pass`.

## Verify

1. Open `http://<forgejo_domain>:<forgejo_web_port>/` — with `forgejo_install_lock: true` the web installer is skipped.
2. Sign in with the admin created by the playbook.
3. Git SSH listen check:

```bash
ssh -p 2222 -o BatchMode=yes git@<forgejo_domain>
# Expect: Permission denied (publickey).
```

## Idempotency and upgrades

- Re-run with unchanged config should report no meaningful drift (`pull: missing`, `recreate: auto`, admin create treats “already exists” as ok).
- Upgrade by bumping `forgejo_version` (prefer a specific tag, not a floating major if you need bit-for-bit re-runs) and re-running the playbook.
- First-run `--check` prints that directories would be created and skips compose until the path exists on a real run. After the path exists, `--check` exercises Docker in check mode.

## Synology notes

1. **Python:** DSM system Python is often 3.8; set `ansible_python_interpreter` to Package Center Python 3.9+.
2. **Docker socket:** use `become` (and preferably a scoped sudoers rule for `/usr/local/bin/docker`).
3. **Timezone:** DSM has `/etc/localtime` but not `/etc/timezone` — only mount localtime.
4. **Data dir:** the role creates `{{ forgejo_install_path }}/data` before Compose starts.

## Reset a broken first install

If you previously used the web installer and need a clean SQLite DB:

```bash
ssh <ansible_user>@<ansible_host>
sudo rm -rf /volume1/docker/forgejo/data/*
sudo /usr/local/bin/docker restart forgejo
```

Then re-run the playbook so install lock and admin creation apply again.
