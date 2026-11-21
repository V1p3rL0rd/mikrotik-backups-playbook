# Ansible MikroTik Backup 💾

[English](README.md) | **Русский**

Идемпотентное и безопасное резервное копирование конфигурации MikroTik RouterOS (`/export`) с помощью Ansible.

```
systemd timer ──► ansible-playbook ──network_cli (SSH, known_hosts verified)──► RouterOS (user=backup, policy ssh,read)
                      │  ansible-vault: credentials
                      ├─► backups/<identity>/current.rsc          (0600, rewritten only on real config change)
                      ├─► backups/<identity>/<identity>_<ts>.rsc  (archived on change, pruned after N days)
                      └─► node_exporter textfile *.prom ──► Prometheus ──► Alertmanager
```

## Возможности ✨

- **Штатное сетевое подключение Ansible** — `community.routeros` + `ansible.netcommon.network_cli`. Без `sshpass`, пароль не виден в списке процессов, ключи хостов SSH проверяются.
- **Секреты в Ansible Vault** — в inventory нет учётных данных.
- **Идемпотентность** — строка с датой в заголовке экспорта нормализуется, поэтому `current.rsc` меняется (и создаётся архивная копия с меткой времени) только при реальном изменении конфигурации. Удобно хранить `backups/` в `git`.
- **Проверка** — обрезанный или пустой экспорт не сохраняется молча, а приводит к ошибке для этого хоста.
- **Срок хранения** — архивы старше `backup_retention_days` удаляются; `current.rsc` сохраняется всегда.
- **Наблюдаемость** — необязательные метрики для textfile collector node_exporter и правила оповещений Prometheus (`alerts.yml`).
- **Расписание** — юниты systemd service/timer с харденингом (`systemd/`).

## Структура

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

## Требования 🛠️

- Ansible ≥ 2.15 (ansible-core)
- Коллекции: `ansible-galaxy collection install -r requirements.yml`
- Библиотека SSH: `pip install paramiko` (или `ansible-pylibssh`)

## RouterOS: отдельный пользователь с минимальными правами

```routeros
/user group add name=backup policy=ssh,read,!local,!telnet,!ftp,!reboot,!write,!policy,!test,!winbox,!password,!web,!sniff,!sensitive,!api,!romon,!rest-api
/user add name=backup group=backup address=<ANSIBLE_HOST_IP>/32 password="<STRONG_PASSWORD>"
# Preferred: key auth
# /user ssh-keys import public-key-file=backup.pub user=backup
```

Если включаете `backup_show_sensitive: true`, добавьте группе политику `sensitive` **и** храните `backups/` в зашифрованном виде.

## Быстрый старт 🚀

```bash
git clone https://github.com/V1p3rL0rd/mikrotik-backups-playbook.git
cd mikrotik-backups-playbook
ansible-galaxy collection install -r requirements.yml
pip install paramiko

# 1. Укажите свои роутеры в inventory/hosts.yml
# 2. Учётные данные
cp group_vars/mikrotik/vault.yml.example group_vars/mikrotik/vault.yml
$EDITOR group_vars/mikrotik/vault.yml
ansible-vault encrypt group_vars/mikrotik/vault.yml

# 3. Добавьте ключи хостов в доверенные (сверьте отпечатки с `/ip ssh print` на каждом роутере!)
ssh-keyscan 192.0.2.1 192.0.2.2 >> ~/.ssh/known_hosts
ssh-keyscan -p 2222 192.0.2.3 >> ~/.ssh/known_hosts

# 4. Запуск
ansible-playbook mt-backup.yml --ask-vault-pass
```

## Переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `backup_root` | `{{ playbook_dir }}/backups` | Где хранятся резервные копии на управляющем узле |
| `backup_retention_days` | `90` | Архивы старше этого срока удаляются |
| `backup_show_sensitive` | `false` | Добавляет `show-sensitive` к `/export` (RouterOS 7) |
| `backup_textfile_dir` | `""` | Каталог textfile collector node_exporter; пусто — отключено |

## Проверка ✅

```bash
ansible-playbook mt-backup.yml --ask-vault-pass        # 1-й запуск: changed
ansible-playbook mt-backup.yml --ask-vault-pass        # 2-й запуск: changed=0
stat -c '%a %n' backups/*/current.rsc                  # ожидается 600
```

Периодически проверяйте восстановление на тестовом стенде или экземпляре CHR:
`/system reset-configuration no-defaults=yes run-after-reset=current.rsc`

## Расписание и мониторинг 📈

Юниты ожидают репозиторий в `/opt/mikrotik-backup` и запускаются от пользователя `ansible` (в его `~/.ssh/known_hosts` должны быть ключи роутеров):

```bash
sudo git clone https://github.com/V1p3rL0rd/mikrotik-backups-playbook.git /opt/mikrotik-backup
sudo install -d -m 0700 -o ansible -g ansible /opt/mikrotik-backup/backups
sudo install -d -m 0750 -o root -g ansible /etc/mikrotik-backup
sudo install -m 0400 -o ansible /dev/stdin /etc/mikrotik-backup/vault_pass   # вставьте пароль vault, затем Ctrl-D
sudo cp /opt/mikrotik-backup/systemd/mikrotik-backup.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now mikrotik-backup.timer
sudo systemctl start mikrotik-backup.service && journalctl -u mikrotik-backup.service -n 30
```

Задайте `backup_textfile_dir` и загрузите `alerts.yml` в Prometheus:
- `MikrotikBackupStale` — нет успешной резервной копии более 26 часов
- `MikrotikBackupSizeDrop` — размер экспорта упал более чем на 50% относительно максимума за 7 дней

## Ограничения ⚠️

- RouterOS 7 по умолчанию скрывает секреты в `/export` (ключи WireGuard, PSK IPsec, секреты PPP) — при восстановлении из `.rsc` их не будет, если не включить `backup_show_sensitive: true`.
- RouterOS 6 по умолчанию включает секреты в экспорт — держите права `backups/` на `0700`/`0600` и храните внешние копии в зашифрованном виде.
- `.rsc` никогда не содержит паролей пользователей и закрытых ключей сертификатов. Для полного аварийного восстановления делайте также бинарную копию `/system backup save` (нужны политики `ftp`/`write` — запускайте её отдельной, более редкой задачей).

## Лицензия

MIT — см. [LICENSE](LICENSE).
