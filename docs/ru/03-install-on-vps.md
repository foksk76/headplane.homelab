Язык: [English](../03-install-on-vps.md) | [Русский](03-install-on-vps.md)

# Установка на VPS

Эта инструкция устанавливает Headplane на VPS, где уже работают Headscale и
Caddy.

## Создайте каталоги

```bash
mkdir -p /opt/headplane
mkdir -p /etc/headplane
mkdir -p /var/lib/headplane
mkdir -p /usr/libexec/headplane
```

## Распакуйте исполняемый архив

```bash
cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz
```

Установите вспомогательные бинарники в фиксированные пути:

```bash
install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck
```

## Сгенерируйте секрет для cookie

Headplane ожидает 32-символьный секрет, если `server.cookie_secret` задан прямо
в конфиге.

Пример:

```bash
openssl rand -hex 16
```

Вывод команды используйте вместо `REPLACE_WITH_32_CHARS`. Это 16 случайных байт
в виде 32 шестнадцатеричных символов. Не переиспользуйте значение между разными
установками и не коммитьте реальный секрет. Если удобнее хранить секрет в файле,
Headplane поддерживает `cookie_secret_path`: тогда в конфиге оставьте только
путь, а сам файл закройте правами доступа.

## Создайте `/etc/headplane/config.yaml`

Проверенный вариант для установки без контейнера:

```yaml
server:
  host: "127.0.0.1"
  port: 3000
  base_url: "https://headscale.example.net"
  cookie_secret: "REPLACE_WITH_32_CHARS"
  cookie_secure: true
  cookie_max_age: 86400
  data_path: "/var/lib/headplane"

headscale:
  url: "http://127.0.0.1:8080"
  public_url: "https://headscale.example.net"
  config_path: "/etc/headscale/config.yaml"
  config_strict: true

integration:
  proc:
    enabled: true
```

Важно:

- замените `headscale.example.net` на публичный FQDN своего VPS
- оставьте `server.host: "127.0.0.1"`, чтобы Headplane был доступен только через Caddy
- `server.port: 3000` — локальный HTTP-порт Headplane для Caddy
- `server.base_url` не должен включать `/admin`
- `cookie_max_age: 86400` означает один день в секундах
- `data_path` должен быть постоянным и доступным на запись для процесса Headplane
- если агент не используется, полностью уберите `integration.agent` в `v0.6.2`
- `headscale.url` должен указывать на локальный Headscale, а не на публичный HTTPS-адрес
- `headscale.public_url` должен совпадать с публичным URL Headscale для клиентов
- `headscale.config_path` должен указывать на реальный конфиг Headscale
- `integration.proc.enabled: true` означает работу Headplane с локальным процессом Headscale

## Создайте службу `systemd`

Путь:

```text
/etc/systemd/system/headplane.service
```

Содержимое:

```ini
[Unit]
Description=Headplane v0.6.2
After=network-online.target headscale.service
Wants=network-online.target
Requires=headscale.service
StartLimitIntervalSec=0

[Service]
Type=simple
User=root
WorkingDirectory=/opt/headplane
Environment=HEADPLANE_CONFIG_PATH=/etc/headplane/config.yaml
ExecStart=/usr/bin/node /opt/headplane/build/server/index.js
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Перечитайте конфигурацию `systemd` и запустите сервис:

```bash
systemctl daemon-reload
systemctl enable --now headplane
```

## Обновите Caddy

Проверенная маршрутизация:

```caddyfile
headscale.example.net {
  @admin_no_slash path /admin
  redir @admin_no_slash /admin/ 308

  @headplane path /admin/*
  reverse_proxy @headplane 127.0.0.1:3000

  reverse_proxy 127.0.0.1:8080
}
```

Затем:

```bash
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

## Дополнительно: заложите OIDC сразу

Если позже нужен встроенный вход через OIDC, при завершении базовой установки
сразу держите в голове такие вещи:

- `server.base_url` должен оставаться на публичном URL без `/admin`
- адрес возврата (`redirect URI`) у поставщика удостоверений должен быть
  `{server.base_url}/admin/oidc/callback`
- для OIDC в Headplane `v0.6.2` нужен `oidc.headscale_api_key`
- вход по ключу API остается доступным и после включения OIDC

Полный сценарий настройки вынесен в отдельную инструкцию:

- [Включение единого входа через OIDC](05-enable-sso-oidc.md)

## Дополнительно: включите агент позже

Если позже понадобится веб-SSH, используйте такую форму:

```yaml
integration:
  proc:
    enabled: true
  agent:
    enabled: true
    pre_authkey: "REUSABLE_PREAUTHKEY"
    executable_path: "/usr/libexec/headplane/agent"
    work_dir: "/var/lib/headplane/agent"
```

`REUSABLE_PREAUTHKEY` генерируется на VPS через Headscale. Флаги зависят от
версии Headscale, поэтому сначала проверьте локальную справку:

```bash
headscale preauthkeys create --help
```

Типичный вариант для многоразового ключа:

```bash
headscale users list
headscale preauthkeys create \
  --user <USER_ID> \
  --reusable \
  --expiration 90d
```

Для инфраструктурных узлов с тегами в новых версиях Headscale команда может
использовать `--tags tag:<TAG>` без `--user`. Надежный путь без героизма:
сначала смотреть `headscale preauthkeys create --help` на установленной версии,
потом копипастить.

Сгенерированный ключ используйте вместо `REUSABLE_PREAUTHKEY`. Это секрет:
любой, у кого он есть, может добавить узел до истечения срока действия или
отзыва ключа. Чтобы уменьшить радиус поражения конфига, используйте
`pre_authkey_path` и храните ключ в файле, доступном только `root`.

## Навигация

Назад: [Перенос исполняемого архива](02-transfer-runtime-artifact.md) | Вперед: [Проверка и вход](04-verify-and-login.md)
