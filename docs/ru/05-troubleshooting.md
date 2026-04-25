Язык: [English](../05-troubleshooting.md) | [Русский](05-troubleshooting.md)

# Диагностика

## `integration.agent.pre_authkey must be a string (was missing)`

Причина:

- `v0.6.2` проверяет блок `integration.agent`, даже если там указано `enabled: false`

Исправление:

- полностью удалите `integration.agent`, если агент не используется
- или укажите реальный `pre_authkey`, если агент используется

## `Error: Can't find meta/_journal.json file`

Причина:

- исполняемый архив был создан без `drizzle/`

Исправление:

- пересоберите архив и включите:
  - `headplane/drizzle/`
  - `headplane/drizzle/meta/_journal.json`

## `/admin` открывает Headscale или пустой ответ вместо Headplane

Причина:

- Caddy все еще отправляет все пути в Headscale

Исправление:

- добавьте отдельный matcher `/admin/*`, который проксирует в `127.0.0.1:3000`
- оставьте маршрут по умолчанию на `127.0.0.1:8080`

## Вход не сохраняется

Причина:

- `server.cookie_secure` не соответствует реальному способу доступа

Исправление:

- если публичный доступ идет через HTTPS за Caddy, оставьте `cookie_secure: true`
- если проверяете напрямую по обычному HTTP, используйте `cookie_secure: false`

## Headplane запускается, но не видит настройки Headscale

Причина:

- неверный `headscale.config_path`
- Headplane не может прочитать файл
- интеграция через процесс не видит процесс Headscale

Исправление:

- проверьте реальный путь к конфигу Headscale
- запустите Headplane с правами, достаточными для `integration.proc.enabled`

## Путь `/admin` ломается после пересборки

Причина:

- `server.base_url` ошибочно включает `/admin`

Исправление:

- используйте `https://headscale.example.net`, а не `https://headscale.example.net/admin`

## Caddy предупреждает о форматировании

Причина:

- Caddyfile рабочий, но не отформатирован

Исправление:

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
```

Это косметика и не блокирует запуск, если `caddy validate` уже говорит
`Valid configuration`.

## Навигация

Назад: [Проверка и вход](04-verify-and-login.md) | Вперед: [Быстрый старт](../../README.ru.md#быстрый-старт)
