Язык: [English](../04-verify-and-login.md) | [Русский](04-verify-and-login.md)

# Проверка и вход

## Проверьте службы

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager
```

Здоровый пример:

- `headplane.service` в состоянии `active (running)`
- `headscale.service` в состоянии `active (running)`
- `caddy.service` в состоянии `active (running)`

## Проверьте порты

```bash
ss -ltnp | egrep ':80 |:443 |:3000 |:8080 |:9090 '
```

Обычно ожидаются такие слушатели:

- `127.0.0.1:3000` — Headplane
- `127.0.0.1:8080` — Headscale
- `127.0.0.1:9090` — метрики Headscale
- `*:80` и `*:443` — Caddy

## Проверьте локальные HTTP-ответы

```bash
curl -I http://127.0.0.1:3000/admin/
curl -I http://127.0.0.1:8080/health
```

Ожидаемые примеры:

- локальный путь Headplane `/admin/`: `302 Found`
- health-путь Headscale: `200 OK`

## Проверьте внешние HTTP-ответы

Замените `headscale.example.net` на публичный FQDN своего VPS.

```bash
curl -I https://headscale.example.net/admin/
curl -I https://headscale.example.net/admin/machines
curl -I https://headscale.example.net/health
```

Ожидаемые примеры:

- `/admin/` может перенаправить на `/admin/machines`
- `/admin/machines` до входа перенаправляет на `/admin/login`
- `/health` возвращает `200 OK`

## Создайте первый ключ для входа

Если OIDC не настроен, Headplane использует ключ API Headscale для входа.

```bash
headscale apikeys create --expiration 90d
```

Затем откройте:

```text
https://headscale.example.net/admin/login
```

Вставьте сгенерированный ключ API в форму входа Headplane.

## Дополнительно проверьте журналы

```bash
journalctl -u headplane -n 100 --no-pager
journalctl -u caddy -n 100 --no-pager
```

В стабильном запуске на проверенной установке были такие строки:

```text
[config] INFO: Found a valid Headscale configuration file at /etc/headscale/config.yaml
[config] INFO: Using Proc integration
[config] INFO: Found headscale serve (PID ...)
[server] INFO: Running on 127.0.0.1:3000
```

## Навигация

Назад: [Установка на VPS](03-install-on-vps.md) | Вперед: [Диагностика](05-troubleshooting.md)
