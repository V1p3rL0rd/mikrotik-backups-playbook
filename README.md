# Ansible MikroTik Backup 💾

**English** | [Русский](README.ru.md)

Idempotent, secure backup of MikroTik RouterOS configurations (`/export`) with Ansible.

```
systemd timer ──► ansible-playbook ──network_cli (SSH, known_hosts verified)──► RouterOS (user=backup, policy ssh,read)
                      │  ansible-vault: credentials
                      ├─► backups/<identity>/current.rsc          (0600, rewritten only on real config change)
                      ├─► backups/<identity>/<identity>_<ts>.rsc  (archived on change, pruned after N days)
                      └─► node_exporter textfile *.prom ──► Prometheus ──► Alertmanager
```

## Features ✨

- **Native Ansible network connection** — `community.routeros` + `ansible.netcommon.network_cli`. No `sshpass`, no password in process list, SSH host keys are verified.
- **Secrets in Ansible Vault** — no credentials in inventory.
- **Idempotent** — the timestamp line of the export header is normalized, so `current.rsc` changes (and a timestamped archive is created) only when the configuration really changes. Works great with `git` on top of `backups/`.
- **Validation** — truncated/empty exports fail the host instead of being saved silently.
- **Retention** — archives older than `backup_retention_days` are pruned; `current.rsc` is always kept.
- **Observability** — optional node_exporter textfile metrics + Prometheus alert rules (`alerts.yml`).
- **Scheduling** — systemd service/timer units with hardening (`systemd/`).

## Layout

```
ansible.cfg
requirements.yml
inventory/hosts.yml
group_vars/mikrotik/vars.yml
group_vars/mikrotik/vault.yml.example   # copy to vault.yml and encrypt
mt-backup.yml
alerts.yml                              # Prometheus rules
systemd/mikrotik-backup.{service,timer}
```

## Requirements 🛠️

- Ansible ≥ 2.15 (ansible-core)
- Collections: `ansible-galaxy collection install -r requirements.yml`
- SSH library: `pip install paramiko` (or `ansible-pylibssh`)

## RouterOS: dedicated least-privilege user

```routeros
/user group add name=backup policy=ssh,read,!local,!telnet,!ftp,!reboot,!write,!policy,!test,!winbox,!password,!web,!sniff,!sensitive,!api,!romon,!rest-api
/user add name=backup group=backup address=<ANSIBLE_HOST_IP>/32 password="<STRONG_PASSWORD>"
# Preferred: key auth
# /user ssh-keys import public-key-file=backup.pub user=backup
```

If you set `backup_show_sensitive: true`, add the `sensitive` policy to the group **and** store `backups/` encrypted.

## Getting started 🚀

```bash
git clone https://github.com/V1p3rL0rd/mikrotik-backups-playbook.git
cd mikrotik-backups-playbook
ansible-galaxy collection install -r requirements.yml
pip install paramiko

# 1. Edit inventory/hosts.yml with your routers
# 2. Credentials
cp group_vars/mikrotik/vault.yml.example group_vars/mikrotik/vault.yml
$EDITOR group_vars/mikrotik/vault.yml
ansible-vault encrypt group_vars/mikrotik/vault.yml

# 3. Trust host keys (verify fingerprints against `/ip ssh print` on each router!)
ssh-keyscan 192.0.2.1 192.0.2.2 >> ~/.ssh/known_hosts
ssh-keyscan -p 2222 192.0.2.3 >> ~/.ssh/known_hosts

# 4. Run
ansible-playbook mt-backup.yml --ask-vault-pass
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `backup_root` | `{{ playbook_dir }}/backups` | Where backups are stored on the control node |
| `backup_retention_days` | `90` | Archives older than this are deleted |
| `backup_show_sensitive` | `false` | Add `show-sensitive` to `/export` (RouterOS 7) |
| `backup_textfile_dir` | `""` | node_exporter textfile collector dir; empty = disabled |

## Verification ✅

```bash
ansible-playbook mt-backup.yml --ask-vault-pass        # 1st run: changed
ansible-playbook mt-backup.yml --ask-vault-pass        # 2nd run: changed=0
stat -c '%a %n' backups/*/current.rsc                  # expect 600
```

Test restore periodically on a lab/CHR instance:
`/system reset-configuration no-defaults=yes run-after-reset=current.rsc`

## Scheduling & monitoring 📈

The units expect the repository in `/opt/mikrotik-backup`, run as the `ansible` user (its `~/.ssh/known_hosts` must contain the router keys):

```bash
sudo git clone https://github.com/V1p3rL0rd/mikrotik-backups-playbook.git /opt/mikrotik-backup
sudo install -d -m 0700 -o ansible -g ansible /opt/mikrotik-backup/backups
sudo install -d -m 0750 -o root -g ansible /etc/mikrotik-backup
sudo install -m 0400 -o ansible /dev/stdin /etc/mikrotik-backup/vault_pass   # paste vault password, then Ctrl-D
sudo cp /opt/mikrotik-backup/systemd/mikrotik-backup.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now mikrotik-backup.timer
sudo systemctl start mikrotik-backup.service && journalctl -u mikrotik-backup.service -n 30
```

Set `backup_textfile_dir` and load `alerts.yml` into Prometheus:
- `MikrotikBackupStale` — no successful backup for > 26h
- `MikrotikBackupSizeDrop` — export size dropped > 50% vs 7-day max

## Caveats ⚠️

- RouterOS 7 hides secrets in `/export` by default (WireGuard keys, IPsec PSK, PPP secrets) — a restore from `.rsc` will lack them unless `backup_show_sensitive: true`.
- RouterOS 6 includes secrets by default — keep `backups/` at `0700`/`0600` and encrypted off-site.
- `.rsc` never contains user passwords or certificate private keys. For full disaster recovery also take a binary `/system backup save` (requires `ftp`/`write` policies — run it as a separate, less frequent job).

## License

MIT — see [LICENSE](LICENSE).
