# Конфигурация и резервные копии

## Что хранить в Git

В Git хранится только обезличенный пример [VLESS + TCP + REALITY inbound](../examples/vless-reality-inbound.example.json). Все значения в угловых скобках — заполнители. Этот фрагмент показывает структуру и не является готовым файлом для запуска. На рабочем сервере конфигурацией управляет 3x-ui; его база `/etc/x-ui/x-ui.db` содержит учётные данные клиентов и параметры входов.

На сервере генерировать UUID и REALITY private key через 3x-ui/Xray. Не переносить рабочие UUID, private key, short ID, ссылки подписки или QR в шаблон, issues, коммиты и логи CI. Публичная часть ключа и клиентский профиль тоже не нужны в этом репозитории.

`.gitignore` игнорирует конфиги, JSON/YAML/TXT-файлы, базы, архивы и клиентские экспорты по умолчанию; примеры с суффиксом `.example` разрешены. Перед коммитом проверять `git status --ignored`, `git diff --cached` и список добавленных файлов. Игнорирование не защищает от `git add -f`.

## Резервная копия панели

База работает в режиме SQLite WAL, поэтому копировать только `x-ui.db` при работающей панели нельзя: последние изменения могут находиться в WAL-файле. Для простой согласованной копии остановить панель на время архивации. Это ненадолго прервёт VPN и веб-панель.

Команды ниже выполняются на VDS в Bash; требуются `tar`, `gzip`, `gnupg` и свободное место. `gnupg` устанавливается на этапе подготовки. Архив и шифрованная копия используют **одно и то же фактическое имя с timestamp**, без ручного переименования. Парольную фразу GPG вводить интерактивно; не передавать аргументом команды.

```bash
sudo bash -euc '
  set -o pipefail
  umask 077
  install -d -o root -g root -m 700 /root/x-ui-backups
  archive="/root/x-ui-backups/x-ui-$(date +%F-%H%M%S).tar.gz"
  systemctl is-active --quiet x-ui
  trap "systemctl start x-ui" EXIT
  systemctl stop x-ui
  tar --numeric-owner -czf "$archive" -C / etc/x-ui
  systemctl start x-ui
  systemctl is-active --quiet x-ui
  trap - EXIT
  gpg --symmetric --cipher-algo AES256 --output "$archive.gpg" "$archive"
  gpg --decrypt --output /dev/null "$archive.gpg"
  printf "Encrypted backup verified: %s\n" "$archive.gpg"
  printf "Unencrypted source to remove after transfer: %s\n" "$archive"
'
```

Команда прекращает работу при ошибке. Если архивация не удалась после остановки, EXIT trap пытается запустить панель; отдельно проверить `sudo systemctl status x-ui --no-pager` и клиент. При ошибке запуска не считать backup завершённым. Шифрование выполняется после запуска панели, чтобы не держать VPN выключенным во время ввода пароля.

Хранить зашифрованную копию и парольную фразу отдельно. Перенести **выведенный `.tar.gz.gpg`** в приватное хранилище вне VDS и Git, проверить его расшифровку в месте хранения. Для root-доступа с локальной машины: `scp root@<SERVER_IP>:/root/x-ui-backups/<ACTUAL_ARCHIVE>.tar.gz.gpg <PRIVATE_BACKUP_DIR>/`; при отдельном sudo-администраторе сначала безопасно подготовить доступный ему файл вне публичных каталогов. Заполнители заменить локально.

После проверки удалённой копии удалить незашифрованный оригинал: `sudo rm -- /root/x-ui-backups/<ACTUAL_ARCHIVE>.tar.gz`. Не удалять по wildcard и не полагаться на `shred` на SSD/виртуальном диске. Архив содержит базу панели, клиентские UUID и другие секреты.

## Восстановление панели

Восстанавливать на сервере с совместимой версией 3x-ui. Проверить доступ по SSH до начала работ и иметь копию текущего `/etc/x-ui` вне целевой директории.

```sh
sudo install -d -o root -g root -m 700 /root/x-ui-restore
sudo gpg --output /root/x-ui-restore/panel.tar.gz --decrypt /secure/path/<ACTUAL_ARCHIVE>.tar.gz.gpg
sudo chmod 600 /root/x-ui-restore/panel.tar.gz
sudo tar -tzf /root/x-ui-restore/panel.tar.gz
```

Проверить успешную расшифровку и список архива **до остановки**; архив должен содержать ожидаемый `etc/x-ui`, без посторонних путей. Использовать доверенную копию. При любой ошибке остановиться; следующие команды выполнять по одной, проверяя результат. Переменная `previous_dir` ниже сохраняет точное имя каталога для отката; весь этап выполнять в одном Bash-сеансе.

```sh
sudo systemctl stop x-ui
previous_dir="/etc/x-ui.before-restore-$(date +%F-%H%M%S)"
sudo mv /etc/x-ui "$previous_dir"
printf "Previous panel state: %s\n" "$previous_dir"
sudo tar --numeric-owner -xzf /root/x-ui-restore/panel.tar.gz -C /
sudo systemctl start x-ui
sudo systemctl status x-ui --no-pager
```

После восстановления проверить вход в панель, слушающие порты и подключение тестового клиентского устройства. База из архива заменяет текущее состояние панели и возвращает значения на момент создания копии. Если распаковка/запуск не удались, остановить `x-ui`, переименовать частично восстановленный `/etc/x-ui` в отдельный каталог, вернуть **точный** сохранённый `/etc/x-ui.before-restore-<TIMESTAMP>` на `/etc/x-ui`, запустить службу и проверить доступ. Не выбирать каталог по wildcard автоматически. После успешной проверки удалить расшифрованный архив из `/root/x-ui-restore/`; копия зашифрованного оригинала должна сохраниться.

## Границы этой копии

Архив `/etc/x-ui` сохраняет состояние панели, но не весь сервер. Для полного disaster recovery отдельно защищённо резервировать `/etc/caddy/`, хранилище сертификатов Caddy (для пакетной службы обычно `/var/lib/caddy/.local/share/caddy/`, фактическое расположение проверить), `/etc/ssh/`, `/etc/ufw/`, `/etc/default/ufw`, `/etc/fail2ban/`, APT security-update настройки, unit и drop-ins `x-ui`, включая `/etc/systemd/system/x-ui.service.d/10-xui-sandbox.conf`. Зафиксировать версии и сохранить доверенные установочные артефакты 3x-ui/Xray/Caddy; для текущего VDS это 3.10.2 / 26.7.28 / 2.11.7 (2026-10-26). Сохранить DNS и правила firewall провайдера в приватном журнале. Содержимое override и полный сценарий восстановления всего VDS в публичном репозитории отсутствуют; восстановление этой копии в ходе сверки README не выполнялось. Не собирать эти файлы в публичном репозитории: они могут содержать домены, пути панели, приватные TLS-ключи и правила доступа.
