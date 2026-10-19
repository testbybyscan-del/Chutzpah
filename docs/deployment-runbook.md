# Пошаговое развёртывание VPN на новом VDS

Runbook описывает чистое развёртывание Ubuntu 24.04 LTS с 3x-ui/Xray, VLESS + TCP + REALITY и Caddy перед веб-панелью. Он не предназначен для повторного запуска на уже настроенном сервере: команды установки и сетевые изменения могут прервать доступ или перезаписать конфигурацию.

Это публичная документация. Все имена, порты и значения в командах ниже — примеры. Не заменять заполнители реальными секретами в Git. Настройки рабочего сервера хранить в защищённом месте вне репозитория.

## 0. Подготовка и переменные

До начала работ получить:

- новый VDS с консолью или rescue-доступом у провайдера;
- публичный IPv4 и, если используется, корректно настроенный IPv6;
- домен, DNS-запись `A` на IPv4 и корректную `AAAA` на IPv6 (или отсутствие `AAAA`, если IPv6 не настроен);
- стабильный административный IP для ограничения панели, если он есть;
- отдельную SSH-пару ключей администратора и защищённое место для хранения приватного ключа;
- согласованное окно простоя для возможной перезагрузки.

Зафиксировать в локальном приватном журнале, не в этом репозитории:

```text
SERVER_IP=<публичный адрес VDS>
PANEL_DOMAIN=<домен панели>
SSH_PORT=22
XUI_PORT=2053                 # внутренний порт панели; наружу не публиковать
PANEL_HTTPS_PORT=8443         # пример внешнего порта Caddy
PANEL_BASE_PATH=/<случайный длинный путь>/
VPN_PORT=443
REALITY_TARGET=<проверенный TLS target>:443
REALITY_SNI=<имя из выбранного target>
ADMIN_SOURCE_IP=<административный адрес, если он статичен>
```

`PANEL_BASE_PATH`, пароль панели, Basic Auth пароль, UUID клиентов, REALITY private key и short ID являются секретами или элементами доступа. Не помещать их в shell history, репозиторий, тикеты, CI-логи или публичные сообщения.

## 1. Проверить сеть и DNS

В панели провайдера разрешить входящие TCP/22, TCP/80, TCP/443 и TCP/8443. Ограничить TCP/22 и TCP/8443 административными IP, если это возможно. На первом этапе не открывать панельный backend-порт 2053. Порт 80 будет нужен Caddy для HTTP-01, а TCP/443 — VPN-входу; Caddy будет слушать HTTPS панели на отдельном порту.

После создания DNS-записи проверить её с локальной машины:

```sh
getent ahostsv4 <DOMAIN>
getent ahostsv6 <DOMAIN>
```

Убедиться, что адреса ведут на этот VDS. Если AAAA-запись указывает на неработающий IPv6, исправить или удалить её до выпуска сертификата.

## 2. Первый вход и отдельный администратор

Войти через консоль провайдера или первоначальную учётную запись образа. Немедленно обновить ОС:

```sh
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y ca-certificates curl gnupg openssh-server sudo ufw fail2ban unattended-upgrades
sudo systemctl enable --now ssh
```

Создать отдельного пользователя-администратора, если образ не создал его:

```sh
sudo adduser <ADMIN_USER>
sudo usermod -aG sudo <ADMIN_USER>
```

На рабочем компьютере создать отдельный ключ, если подходящего ключа нет:

```sh
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/<VDS_KEY_NAME>
ssh-copy-id -i ~/.ssh/<VDS_KEY_NAME>.pub <ADMIN_USER>@<SERVER_IP>
```

Проверить вход новым ключом во втором терминале, выполнить `sudo -v` и убедиться, что доступна консоль провайдера. Не отключать парольный вход, пока ключевой вход не проверен.

После успешной проверки можно отключить удалённый вход root и пароли. Создать ранний drop-in, чтобы он прочитался до возможных cloud-init настроек:

```sh
sudo tee /etc/ssh/sshd_config.d/00-local-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
EOF
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
sudo systemctl reload ssh.service
```

Оставить исходную SSH-сессию открытой и во втором терминале заново подключиться новым пользователем. При проблеме использовать консоль провайдера; не закрывать единственную рабочую SSH-сессию до подтверждения повторного входа. Рекомендации Ubuntu по OpenSSH и ключам — в [официальном руководстве Ubuntu](https://ubuntu.com/server/docs/how-to/security/openssh-server/).

## 3. Включить firewall и обновления безопасности

Добавить только необходимые порты до включения UFW. Порядок команд важен: сначала правило SSH, затем `ufw enable`.

```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8443/tcp
sudo ufw enable
sudo ufw status verbose
```

Если административный IP статичен, вместо открытого всем TCP/8443 использовать правило `sudo ufw allow from <ADMIN_SOURCE_IP>/32 to any port 8443 proto tcp`. Для IPv6 настроить соответствующий адрес и проверить, что IPv6-фильтрация включена в `/etc/default/ufw`. Сверить правила UFW с firewall провайдера.

Не добавлять правило для TCP/2053: панель должна быть доступна только локально или через Caddy. Не открывать резервные VPN-порты «на всякий случай».

Включить и проверить автоматические security updates:

```sh
sudo dpkg-reconfigure -plow unattended-upgrades
systemctl list-timers 'apt-daily*'
systemctl is-enabled apt-daily.timer apt-daily-upgrade.timer
```

Проверить настройки автоматической перезагрузки в `/etc/apt/apt.conf.d/50unattended-upgrades`. Для VDS, где VPN должен оставаться доступным, согласовать reboot policy и окно простоя. Официальные справки: [firewall/UFW Ubuntu](https://ubuntu.com/server/docs/firewalls/) и [automatic updates Ubuntu](https://ubuntu.com/server/docs/how-to/software/automatic-updates/).

## 4. Установить 3x-ui

Установщик 3x-ui выполняется с правами root и меняет систему. Сверить актуальные требования и способ установки с [официальной инструкцией установки 3x-ui](https://github.com/MHSanaei/3x-ui/wiki/Installation). Для контроля не запускать загруженный код вслепую через конвейер: сохранить его, проверить источник и содержимое, записать версию/контрольную сумму в приватный журнал, затем запустить:

```sh
sudo install -d -o root -g root -m 700 /root/install-sources
sudo curl -fL --proto '=https' --tlsv1.2 \
  https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh \
  -o /root/install-sources/3x-ui-install.sh
sudo less /root/install-sources/3x-ui-install.sh
sudo sha256sum /root/install-sources/3x-ui-install.sh
sudo bash /root/install-sources/3x-ui-install.sh
```

Если в официальном проекте доступен зафиксированный release installer или подписанный checksum, использовать его и записать точный release/tag. Установка из `master` выше удобна, но сама ссылка подвижная и не обеспечивает воспроизводимость версии.

Во время установки задать случайные учётные данные панели, непредсказуемый web base path и внутренний порт. Сохранить установочные данные только в локальное хранилище секретов. Файл `/etc/x-ui/install-result.env` может содержать пароль, API token и URL; держать его с правами root-only. Не публиковать его содержимое.

Убедиться, что служба запущена, а внешний firewall по-прежнему не разрешает внутренний порт панели:

```sh
sudo systemctl enable --now x-ui
sudo systemctl status x-ui --no-pager
sudo ss -lntup
sudo ufw status numbered
```

Перед первой настройкой панели подключаться к backend через SSH-туннель, не открывая его в Интернет:

```sh
ssh -L 2053:127.0.0.1:2053 <ADMIN_USER>@<SERVER_IP>
```

Затем открыть `http://127.0.0.1:2053/<PANEL_BASE_PATH>/` локально. Если панель настроена на другой bind address/port, заменить параметры туннеля на фактические. После входа проверить/изменить учётные данные, web base path и порт; включить MFA, если эта версия панели её поддерживает.

Путь и нестандартный порт затрудняют автоматический перебор, но не заменяют пароль, TLS, firewall и ограничение источников доступа.

## 5. Установить Caddy

Для Ubuntu использовать официальный пакет Caddy из его репозитория. Команды ниже приведены из [официальной инструкции установки Caddy](https://caddyserver.com/docs/install):

```sh
sudo apt install --yes debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list >/dev/null
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install --yes caddy
sudo systemctl enable --now caddy
```

## 6. Опубликовать панель только через Caddy

До настройки reverse proxy убедиться, что в 3x-ui заданы:

- backend-порт, например 2053;
- web base path с длинным случайным значением, например `/RANDOM_LONG_PATH/`;
- loopback bind `127.0.0.1`, если настройка поддерживается установленной версией;
- извне закрыт backend-порт 2053 как в UFW, так и у провайдера.

Сгенерировать Basic Auth hash интерактивно, чтобы пароль не попал в историю команд:

```sh
sudo caddy hash-password --algorithm bcrypt
```

Команда без `--plaintext` читает пароль из stdin; при интерактивном терминале ввод не отображается. Вставить полученный hash, а не пароль, в защищённый Caddyfile. Пример структуры — заменить все заглавные заполнители до проверки:

```caddyfile
panel.example.net:8443 {
    # Xray occupies public TCP/443; use ACME HTTP-01 through public TCP/80.
    tls {
        issuer acme {
            disable_tlsalpn_challenge
        }
    }

    handle /RANDOM_LONG_PATH/* {
        basic_auth {
            vpnadmin <BCRYPT_HASH_FROM_CADDY>
        }
        reverse_proxy 127.0.0.1:2053
    }

    respond "Not found" 404
}
```

В этом варианте панель доступна по HTTPS на TCP/8443. `disable_tlsalpn_challenge` исключает ACME challenge на занятом Xray порту 443; Caddy получает и продлевает сертификат через HTTP-01. Поэтому DNS должен указывать на VDS, а TCP/80 должен быть доступен из Интернета и разрешён в firewall провайдера и UFW. Если выпуск не прошёл, проверить DNS, порт 80 и журнал Caddy. См. [Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https) и [опции ACME в Caddyfile](https://caddyserver.com/docs/caddyfile/directives/tls).

Отредактировать файл командой `sudoedit /etc/caddy/Caddyfile`; не включать резервную копию с реальными значениями в Git.

3x-ui использует WebSocket для live updates; Caddy обрабатывает upgrade автоматически. Сохраняя тот же web base path в панели и Caddy, проксировать запросы к loopback backend. Дополнительные сведения — [официальный раздел 3x-ui о reverse proxy](https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/en/operations/reverse-proxy.mdx).

Проверить конфиг до reload:

```sh
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
sudo systemctl status caddy --no-pager
sudo journalctl -u caddy -n 100 --no-pager
```

Проверить с внешней машины: URL панели отдаёт TLS-сертификат, неавторизованный запрос получает 401 или 404, вход с двумя факторами/паролем открывает панель, а `http://<SERVER_IP>:2053` недоступен снаружи. Не включать в Caddy доступ к `/sub/`, пока не решено, что подписки нужны и как будут защищаться ссылки подписки.

## 7. Создать VLESS + TCP + REALITY inbound

В 3x-ui создать inbound и записать его параметры в локальный менеджер секретов:

1. Протокол: VLESS.
2. Транспорт: TCP (в актуальной конфигурации Xray может называться RAW).
3. Публичный порт: TCP/443.
4. Flow: `xtls-rprx-vision`, только если клиент поддерживает Vision.
5. Security: REALITY.
6. Сгенерировать уникальные client UUID, server private key и короткий hex short ID на сервере/в панели. Не повторно использовать значения на другом inbound.
7. Выбрать совместимый REALITY target и SNI, проверить его доступность и соответствие текущей документации Xray. Не выбирать адреса, превращающие сервер в открытый форвардер к CDN или чужому сервису. REALITY пересылает не прошедший проверку трафик к target; учитывать это при выборе.
8. Оставить ограничение минимальной версии клиента пустым, если нет установленной необходимости; выставлять `minClientVer=0.0.0` только после теста совместимости и документировать причину.
9. Не создавать дополнительный старый inbound/порт. Для перехода существующих клиентов отдельный временный резервный inbound оформлять как временное исключение с датой удаления.

Схема полей есть в [обезличенном inbound-примере](../examples/vless-reality-inbound.example.json). Официальные поля и формат REALITY проверять в [документации Xray](https://xtls.github.io/en/config/transports/reality.html). Настройки в 3x-ui генерируются панелью; не копировать пример как полный `config.json`.

На ранее настроенном сервере Xray закреплён на 26.7.28 после тайм-аутов REALITY handshake в Hiddify 4.1.1 на более новой версии. Это результат проверки конкретной пары сервера/клиента, а не универсальное требование. На новом VDS не считать автоматическое обновление совместимым без клиентской проверки; для уже работающего сервера не обновлять Xray автоматически.

Создать отдельный клиентский профиль на каждое устройство/пользователя, указать осмысленную метку, лимит/срок при необходимости. Экспортировать ссылку или QR напрямую из панели, передать через защищённый канал и после передачи проверить подключение на конечном устройстве. Ссылка и QR являются паролем доступа.

## 8. Установить Fail2ban и проверить обновления

Убедиться, что Fail2ban установлен и включён. Для jail SSH применять локальный файл `/etc/fail2ban/jail.d/sshd.local`, не править пакетный `jail.conf`:

```ini
[sshd]
enabled = true
backend = systemd
maxretry = 5
findtime = 10m
bantime = 1h
```

После сохранения:

```sh
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Если текущая версия 3x-ui создаёт отдельный jail, проверить его эффективную конфигурацию и firewall backend по [официальной документации 3x-ui](https://github.com/MHSanaei/3x-ui/wiki/Configuration). Не считать бан работающим, пока jail не виден в `fail2ban-client status` и блокировка не применяется выбранным backend.

## 9. Проверка перед передачей в эксплуатацию

На сервере:

```sh
sudo systemctl is-active x-ui caddy fail2ban
sudo systemctl is-enabled x-ui caddy fail2ban apt-daily.timer apt-daily-upgrade.timer
sudo ss -lntup
sudo ufw status numbered
sudo fail2ban-client status
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '
sudo test -e /var/run/reboot-required && cat /var/run/reboot-required.pkgs || true
```

Ожидаемая схема listeners: SSH/22, HTTP/80 для Caddy/ACME, Xray/443 и Caddy/8443. Backend-порт панели не должен быть разрешён извне. Отдельной службы `xray.service` может не быть: Xray может запускаться дочерним процессом `x-ui`.

С внешней машины проверить только транспортные порты:

```sh
nc -vz -w 5 <SERVER_IP> 22
nc -vz -w 5 <SERVER_IP> 80
nc -vz -w 5 <SERVER_IP> 443
nc -vz -w 5 <SERVER_IP> 8443
nc -vz -w 5 <SERVER_IP> 2053  # ожидается недоступность
```

TCP-проверка не доказывает работоспособность REALITY. Финальная проверка — импорт профиля в поддерживаемый клиент, подключение и реальный трафик. Проверить отдельно Wi-Fi и мобильную сеть. Зафиксировать фактические версии, дату, результат проверки и контрольную сумму installer без сохранения секретов.

## 10. Резервная копия, обслуживание и откат

Сразу после успешной проверки сделать согласованную копию `/etc/x-ui`, зашифровать её и перенести за пределы VDS. Полная процедура SQLite/WAL, восстановления и ограничения копии описана в [руководстве по конфигурации и backup](configuration-and-backups.md). Не сохранять backup в этом репозитории.

Перед обновлением:

1. сохранить backup и записать текущие версии 3x-ui/Xray/Caddy;
2. прочитать release notes и проверить совместимость клиента;
3. менять одну существенную часть за раз;
4. иметь доступ к provider console и SSH rollback path;
5. после обновления повторить проверки разделов 9 и клиентский тест.

При ошибке:

- **Панель недоступна, VPN работает:** проверить `systemctl status caddy x-ui`, `journalctl -u caddy -u x-ui`, DNS/сертификат и Caddyfile; вернуть последнюю рабочую копию Caddyfile, выполнить `caddy validate`, затем reload.
- **VPN не подключается после изменения inbound:** вернуть ранее сохранённый вариант inbound из backup/панели, проверить правильность UUID, flow, SNI, private/public keys и short ID; затем проверить `ss -lntup` и клиентский профиль.
- **Firewall отрезал SSH:** использовать provider console/rescue, восстановить правило SSH до изменения остальных правил. Не пытаться исправлять firewall через недоступный SSH.
- **Обновление Xray сломало клиент:** откатить Xray к последней проверенной версии поддерживаемым способом панели/релиза и восстановить проверенную конфигурацию. Не менять одновременно версию Xray, transport и клиентский профиль.
- **Восстановление базы:** следовать разделу восстановления из [backup guide](configuration-and-backups.md); не заменять живую SQLite DB отдельным файлом без учёта WAL.

После успешного переноса всех клиентов временный старый inbound удалить, его порт закрыть в UFW и firewall провайдера, затем повторить внешний порт-скан.

## Официальные источники

- [Ubuntu: OpenSSH server и ключи](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [Ubuntu: UFW/firewall](https://ubuntu.com/server/docs/firewalls/)
- [Ubuntu: unattended upgrades](https://ubuntu.com/server/docs/how-to/software/automatic-updates/)
- [3x-ui: установка](https://github.com/MHSanaei/3x-ui/wiki/Installation)
- [3x-ui: reverse proxy](https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/en/operations/reverse-proxy.mdx)
- [Caddy: установка на Debian/Ubuntu](https://caddyserver.com/docs/install)
- [Caddy: автоматический HTTPS и ACME](https://caddyserver.com/docs/automatic-https)
- [Caddy: basic_auth и хэширование паролей](https://caddyserver.com/docs/caddyfile/directives/basic_auth), [команда hash-password](https://caddyserver.com/docs/command-line)
- [Xray: REALITY transport](https://xtls.github.io/en/config/transports/reality.html)

Ссылки проверены 2026-10-19. Перед новым развёртыванием проверять их и актуальную версию каждого проекта.
