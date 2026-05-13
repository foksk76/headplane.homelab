Язык: [English](../optional-enable-oidc.md) | [Русский](optional-enable-oidc.md)

# Дополнительно: включение OIDC в Headplane

Эта инструкция добавляет вход через OIDC с помощью штатного механизма единого
входа Headplane.
Предполагается, что базовая установка из этого репозитория уже работает.

> Статус: этот путь OIDC собран по актуальной официальной документации
> Headplane по SSO и конфигурации, но еще не прогнан насквозь на анонимизированном
> примере VPS из этого репозитория.

## Цель

Сохранить текущий путь `/admin`, добавить вход через внешнего поставщика
удостоверений и оставить вход по ключу API как запасной административный
вариант.

## Что нужно заранее

- работающая установка Headplane
- `server.base_url`, уже указывающий на публичный URL без `/admin`
- поставщик OpenID Connect со стандартным `issuer`
- ключ API Headscale с достаточно длинным сроком действия для серверных
  действий OIDC

Полезно знать заранее:

- Headplane рекомендует по возможности использовать один и тот же OIDC-клиент
  и для Headscale, и для Headplane
- если в самом Headscale используются локальные пользователи, а не OIDC,
  автоматического сопоставления на первом входе не будет; пользователь один раз
  выберет свою запись Headscale во время первичной настройки

## Зарегистрируйте адрес возврата

У поставщика удостоверений зарегистрируйте такой адрес возврата:

```text
https://headscale.example.net/admin/oidc/callback
```

Если Headscale уже использует того же поставщика удостоверений и вы хотите
переиспользовать один OIDC-клиент, убедитесь, что у этого клиента зарегистрирован
и адрес возврата Headscale.

## Подготовьте файлы с секретами

Создайте небольшой каталог под секреты:

```bash
install -d -m 0700 /etc/headplane/secrets
printf '%s\n' 'REPLACE_WITH_OIDC_CLIENT_SECRET' > /etc/headplane/secrets/oidc_client_secret
printf '%s\n' 'REPLACE_WITH_LONG_LIVED_HEADSCALE_API_KEY' > /etc/headplane/secrets/headscale_api_key
chmod 0600 /etc/headplane/secrets/oidc_client_secret /etc/headplane/secrets/headscale_api_key
```

Подставьте реальный секрет клиента OIDC от своего поставщика удостоверений и
реальный ключ API Headscale для серверных действий Headplane. Секреты любят
тишину, а не публичные репозитории.

## Дополните `/etc/headplane/config.yaml`

Добавьте в существующий конфиг такие значения:

```yaml
headscale:
  url: "http://127.0.0.1:8080"
  public_url: "https://headscale.example.net"
  config_path: "/etc/headscale/config.yaml"
  config_strict: true
  api_key_path: "/etc/headplane/secrets/headscale_api_key"

oidc:
  issuer: "https://idp.example.com"
  client_id: "REPLACE_WITH_OIDC_CLIENT_ID"
  client_secret_path: "/etc/headplane/secrets/oidc_client_secret"
  scope: "openid email profile"
  use_pkce: true
```

Замечания:

- `headscale.api_key_path` обязателен для OIDC в Headplane
- `client_secret_path` позволяет не хранить секрет клиента прямо в основном
  конфиге
- Headplane сам умеет находить `authorization`, `token` и `userinfo` endpoint
  по метаданным `issuer`
- `use_pkce: true` — хороший вариант по умолчанию, потому что некоторые
  поставщики без него начинают капризничать

Если поставщику удостоверений нужны дополнительные параметры авторизации,
добавьте их в `oidc.extra_params`.

## Перезапустите Headplane

```bash
systemctl restart headplane
systemctl status headplane --no-pager
```

Если служба не поднялась чисто, посмотрите журнал:

```bash
journalctl -u headplane -n 100 --no-pager
```

## Проверьте сценарий входа

Откройте:

```text
https://headscale.example.net/admin/login
```

Что ожидать:

- первый пользователь, вошедший через OIDC, получает роль `Owner`
- следующие пользователи получают роль `Member`, пока им не выдадут права
- сеансы по ключу API по-прежнему имеют полный доступ и обходят ролевую модель
- если в Headscale используются локальные пользователи, Headplane один раз
  попросит выбрать соответствующую запись Headscale

После первого успешного входа через OIDC зайдите на страницу пользователей и
раздайте роли руками. Маленькая скучная операция, зато потом меньше веселья в
журналах.

## Быстрые проверки

```bash
curl -I https://headscale.example.net/admin/login
journalctl -u headplane -n 100 --no-pager | grep -i oidc
```

Полезные признаки:

- `/admin/login` возвращает `200 OK`
- в журнале службы видно, что OIDC-сценарий стартует без ошибок адреса возврата
  и `issuer`

## Навигация

Назад: [Проверка и вход](04-verify-and-login.md) | Вперед: [Диагностика](05-troubleshooting.md)
