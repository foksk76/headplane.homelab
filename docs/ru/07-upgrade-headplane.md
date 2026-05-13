Язык: [English](../07-upgrade-headplane.md) | [Русский](07-upgrade-headplane.md)

# Обновление Headplane

Эта инструкция описывает штатный путь обновления нативной установки Headplane,
которым пользуется этот репозиторий при сверке живого VPS с последним
стабильным релизом.

> Статус: по состоянию на 13 мая 2026 года последним стабильным релизом
> остается `v0.6.2`. Проверенный в этом репозитории VPS уже работал на
> `v0.6.2`, поэтому живой раскатки нового бинаря в этой проверке не
> понадобилось. Ниже описан путь обновления более старой нативной установки до
> этого стабильного состояния.

## Цель

Аккуратно подтянуть более старую нативную установку Headplane вперед, не
съезжая с реального поведения официального релиза `v0.6.2`.

## Сначала проверьте текущие версии

На VPS:

```bash
node -v
headscale version
sed -n '1,40p' /opt/headplane/package.json
systemctl is-active headplane headscale caddy
```

Ожидаемая картина:

- `headplane` уже в состоянии `active`
- `headscale` уже в состоянии `active`
- `caddy` уже в состоянии `active`
- в `/opt/headplane/package.json` видна текущая раскатанная версия Headplane

## Что менялось внутри ветки `0.6.x`

Ниже перечислены изменения конфигурации и эксплуатации, которые реально важны
при обновлении внутри ветки `0.6.x`.

### `0.6.0`

- Headplane требует Headscale `0.26.0` или новее.
- Маршрут `/admin` стал чувствительнее к кривому проксированию, поэтому не
  режьте этот префикс до того, как запрос дойдет до Headplane.

### `0.6.1`

- Headplane начал использовать `/var/lib/headplane/hp_persist.db`.
- Для чувствительных значений появились поля с суффиксом `_path`, в том числе:
  - `server.cookie_secret_path`
  - `integration.agent.pre_authkey_path`
  - `oidc.client_secret_path`
  - `oidc.headscale_api_key_path`

### `0.6.2`

- Добавлена поддержка Headscale `0.28.0`.
- Появились настраиваемые `server.cookie_max_age` и `server.cookie_domain`.
- Предпочтительным способом сборки полного исполняемого набора стал
  `./build.sh`.
- В OIDC есть несколько важных моментов:
  - `server.base_url` стал главным источником для адреса возврата
  - `oidc.redirect_uri` объявлен устаревшим
  - `oidc.use_pkce` стал явной настройкой
  - появился `oidc.enabled` для явного включения OIDC
  - `oidc.token_endpoint_auth_method` стал необязательным

## Замечание по именам полей

Публичный сайт документации не в каждом примере жестко привязан к версии. Для
точного поведения `v0.6.2`, на котором держится этот репозиторий, при спорных
местах ориентируйтесь на исходники релиза с нужным тегом.

В частности, для OIDC в `v0.6.2` используйте:

- `oidc.headscale_api_key`, или
- `oidc.headscale_api_key_path`

Не переписывайте это на `headscale.api_key` в пределах этой стабильной ветки.

## Перед раскаткой снимите резервную копию

Сначала откройте инструкцию:

- [Резервное копирование и восстановление](06-backup-and-restore.md)

Это та самая скучная часть, которая потом экономит нервы и крепкие слова.

## Пересоберите исполняемый набор на промежуточном хосте

```bash
cd /root
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
pnpm install
./build.sh

tar -czf /root/headplane-v0.6.2-runtime-r2.tar.gz \
  --exclude="*.map" \
  --exclude="node_modules/.vite-temp" \
  -C /root \
  headplane/package.json \
  headplane/pnpm-lock.yaml \
  headplane/config.example.yaml \
  headplane/build \
  headplane/public \
  headplane/node_modules \
  headplane/drizzle
```

## Перед перезапуском проверьте конфиг

На VPS минимум проверьте такие вещи:

- `server.base_url` указывает на публичный URL без `/admin`
- `server.cookie_secure: true` остается включенным, если снаружи HTTPS за Caddy
- `server.cookie_max_age` и, при необходимости, `server.cookie_domain`
  выставлены осознанно
- для OIDC используется `oidc.enabled: true` и
  `oidc.headscale_api_key` или `oidc.headscale_api_key_path`
- если агент не нужен, в `v0.6.2` лучше убрать `integration.agent` целиком

## Раскатите новый исполняемый набор на VPS

Скопируйте новый архив на VPS, затем:

```bash
systemctl stop headplane

cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz

install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck

systemctl daemon-reload
systemctl start headplane
```

Если поменялись unit-файл или конфигурация Caddy, перечитайте и их. Если не
поменялись, не надо устраивать лишний шум только ради ощущения бурной
деятельности.

## Проверка после обновления

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager

curl -I https://headscale.example.net/admin/login
curl -I https://headscale.example.net/health
journalctl -u headplane -n 100 --no-pager
```

Если OIDC включен, отдельно проверьте:

```bash
journalctl -u headplane -n 100 --no-pager | grep -i oidc
```

## Что уже проверено в этом репозитории

Во время проверки 13 мая 2026 года:

- подтверждено, что последняя стабильная версия upstream — `v0.6.2`
- проверенный VPS уже работал на `v0.6.2`
- живая раскатка нового бинаря на этом хосте не потребовалась
- после проверки сервисы остались в здоровом состоянии

## Навигация

Назад: [Резервное копирование и восстановление](06-backup-and-restore.md) | Вперед: [Диагностика](08-troubleshooting.md)
