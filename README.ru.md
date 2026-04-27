Язык: [English](README.md) | [Русский](README.ru.md)

# Сборка Headplane в домашней лаборатории

[![Документация](https://img.shields.io/badge/docs-Headplane%20homelab-blue)](README.ru.md)
[![Языки](https://img.shields.io/badge/language-EN%20%7C%20RU-informational)](README.md)
[![Лицензия](https://img.shields.io/github/license/foksk76/headplane.homelab)](LICENSE)
[![Последний релиз](https://img.shields.io/github/v/tag/foksk76/headplane.homelab?label=latest%20release)](https://github.com/foksk76/headplane.homelab/releases/latest)
[![Проверка ссылок](https://github.com/foksk76/headplane.homelab/actions/workflows/docs.yml/badge.svg)](https://github.com/foksk76/headplane.homelab/actions/workflows/docs.yml)

Я пришел к этой схеме из обычной потребности: нужен удобный доступ к
своим сервисам, но не хочется без необходимости выставлять их наружу. Headscale
помогает собрать управляемый контур подключения, а Headplane добавляет к нему
понятный веб-интерфейс. В итоге важные сервисы остаются под контролем, а
обслуживание становится предсказуемым.

Этот репозиторий описывает, как собрать Headplane на промежуточном хосте и
установить его на VPS рядом с уже работающими Headscale и Caddy.

> **Статус проекта:** рабочие заметки по установке Headplane `v0.6.2`, проверенные на промежуточном хосте сборки и Debian 13 VPS.

> **Языковая политика:** `README.md` — обзор на английском языке. `README.ru.md` — русская версия для домашней лаборатории и быстрого входа в проект. Переключатель языка остается первой строкой в обоих файлах.

## Зачем нужен этот репозиторий

Headplane можно собирать прямо на целевом сервере, но это не всегда удобно с
операционной точки зрения. В этом сценарии сборка происходила на отдельном
хосте с полным набором инструментов, а VPS получал только исполняемый архив,
службу `systemd`, конфиг и правку обратного прокси.

Здесь описано, как собрать Headplane `v0.6.2`, корректно упаковать исполняемые
файлы, перенести их на VPS с уже установленными Headscale и Caddy и не наступить
на найденные в реальной установке ловушки.

Если вы ищете первичную установку самого Headscale, начните с
[официальной документации Headscale по установке](https://headscale.net/stable/setup/install/official/).
Этот репозиторий начинается с момента, когда Headscale уже установлен и доступен.

Полезные внешние материалы:

- [Headplane](https://headplane.net/install)
- [конфигурация Headplane](https://headplane.net/configuration)
- [обратный прокси Headscale через Caddy](https://docs.headscale.org/ref/integration/reverse-proxy/#caddy)
- [практика обратного прокси на Caddy](https://swetrix.com/blog/caddy-reverse-proxy)

## Стиль документации

Документация устроена как короткий вход в проект: сначала схема и состав
компонентов, затем пошаговая сборка, установка, проверка и диагностика.

## Логическая схема установки компонентов

```text
build-host.internal
  -> сборка Headplane v0.6.2
  -> архив /root/headplane-v0.6.2-runtime-r2.tar.gz
  -> перенос на headscale.example.net
  -> /opt/headplane + /etc/headplane/config.yaml + systemd
  -> Caddy: /admin/* в Headplane, остальные пути в Headscale
```

## Основной сценарий использования

1. Подготовить Debian-хост сборки с `git`, `go`, `node` и `pnpm`.
2. Клонировать исходный репозиторий Headplane и переключиться на `v0.6.2`.
3. Выполнить полную сборку через `./build.sh`, чтобы результат включал:
   - сборку веб-части
   - `hp_agent`
   - `hp_healthcheck`
   - `hp_ssh.wasm`
   - рабочий набор `node_modules`
4. Создать исполняемый архив, в который обязательно входит `drizzle/`.
5. Перенести архив на VPS.
6. Установить Headplane в `/opt/headplane`, настроить `/etc/headplane/config.yaml`,
   добавить службу `systemd` и обновить Caddy для маршрута `/admin`.
7. Проверить локальные и внешние HTTP-пути и войти через ключ API Headscale.

## Для кого это

- уже есть Headscale на VPS, но нужен веб-интерфейс управления, которого в самом Headscale нет
- сборку удобнее делать на отдельном хосте, а на VPS переносить только готовый архив
- нужен понятный порядок установки с проверками после каждого важного шага

## Компоненты установки

Перед применением замените `headscale.example.net`, `build-host.internal`,
`<BUILD_HOST_IP>` и `<VPS_PUBLIC_IP>` на свои значения. В открытый репозиторий реальные FQDN и IP не
кладем: секретов там нет, но лишняя разведка нам тоже без надобности.

| Компонент | FQDN / IP | ОС | Основное ПО | Назначение |
|---|---|---|---|---|
| Хост сборки | `build-host.internal`, `<BUILD_HOST_IP>` | Debian 13 или совместимый Linux | `git`, Go 1.24+, Node.js 22.x, `pnpm` 10.x | Собирает Headplane и готовит архив без установки тяжелого инструментария на VPS. |
| VPS Headscale | `headscale.example.net`, `<VPS_PUBLIC_IP>` | Debian 13 | Headscale `v0.28.0`, Headplane `v0.6.2`, Caddy `2.6.2`, Node.js 22.x, `systemd` | Держит панель Headplane, сервер Headscale и TLS-вход через Caddy. |

Практика взята из официальных материалов Headplane и Headscale и типовых
подходов домашних лабораторий к Caddy: один публичный HTTPS-вход, Caddy отвечает за TLS,
внутренние сервисы слушают локальный адрес `127.0.0.1`, секреты генерируются отдельно и не
попадают в репозиторий. Это скучно, зато ночью не будит.

## Требования

### Хост сборки

- Debian 13 или похожий Linux-хост
- `git`
- `go` 1.24+ желательно
- Node.js 22.x
- `pnpm` 10.x
- сетевой доступ к GitHub, реестру npm и модулям Go

### Целевой VPS

- Debian 13 VPS или похожий Linux-хост
- рабочий `headscale`, установленный без контейнера
- рабочий `caddy`
- установленный `node` 22.x
- доступ к `/etc/headscale/config.yaml`
- возможность запускать Headplane от `root`, если используется интеграция через локальный процесс

### Проверенные версии

| Компонент | Версия | Примечания |
|---|---|---|
| Headplane | `v0.6.2` | зафиксированная версия исходного проекта |
| Headscale | `v0.28.0` | проверено на целевом VPS |
| Node.js | `v22.22.2` | проверено на хосте сборки и целевом VPS |
| pnpm | `10.4.0` | проверено на хосте сборки |
| Go | `1.24.4` | проверено на хосте сборки |
| Caddy | `2.6.2` | проверено на целевом VPS |
| Debian | `13 (trixie)` | проверено на обоих хостах |

## Структура репозитория

- `README.md` — основной обзор проекта на английском
- `README.ru.md` — основной русский перевод
- `docs/` — пошаговые инструкции на английском языке
- `docs/ru/` — пошаговые инструкции на русском языке
- `docs/ru/01-build-on-intermediate-host.md` — сборка на `build-host.internal`
- `docs/ru/02-transfer-runtime-artifact.md` — перенос исполняемого архива на VPS
- `docs/ru/03-install-on-vps.md` — установка на `headscale.example.net`
- `docs/ru/04-verify-and-login.md` — проверка сервиса и вход
- `docs/ru/05-troubleshooting.md` — диагностика
- `HANDOFF.md` — текущее состояние передачи работ
- `NEXT_STEPS.md` — следующие улучшения репозитория и процесса установки
- `CHANGELOG.md` — история изменений документации
- `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, `LICENSE` — правила участия, безопасности, поддержки и лицензия

## Быстрый старт

Короткий путь: собрать Headplane на промежуточном хосте, перенести исполняемый
архив, установить его на VPS, обновить Caddy и проверить `/admin`.

### 1. Прочитайте пошаговые инструкции по порядку

1. [Сборка на промежуточном хосте](docs/ru/01-build-on-intermediate-host.md)
2. [Перенос исполняемого архива](docs/ru/02-transfer-runtime-artifact.md)
3. [Установка на VPS](docs/ru/03-install-on-vps.md)
4. [Проверка и вход](docs/ru/04-verify-and-login.md)
5. [Диагностика](docs/ru/05-troubleshooting.md)

### 2. Минимальная сводка по сборке

```bash
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
pnpm install
./build.sh
```

Потом создайте архив для установки, в котором сохранен каталог миграций:

```bash
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

### 3. Минимальная сводка по установке на VPS

```bash
mkdir -p /opt/headplane /etc/headplane /var/lib/headplane /usr/libexec/headplane
cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz

install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck
```

Потом создайте:

- `/etc/headplane/config.yaml`
- `/etc/systemd/system/headplane.service`
- маршрут `/admin/*` в `/etc/caddy/Caddyfile`

### 4. Проверьте результат

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager

curl -I http://127.0.0.1:3000/admin/
curl -I http://127.0.0.1:8080/health
curl -I https://headscale.example.net/admin/
curl -I https://headscale.example.net/health
```

Если все хорошо:

- `headplane` активен
- `headscale` активен
- `caddy` активен
- локальный `/admin/` отвечает
- внешний `https://headscale.example.net/admin/` попадает в Headplane

## Подводные камни

- В `v0.6.2` нельзя оставлять `integration.agent` в конфиге, если агент выключен. Если блок существует, валидатор все равно потребует `integration.agent.pre_authkey`.
- Нельзя исключать `drizzle/` из исполняемого архива. Headplane нужен `drizzle/meta/_journal.json` при старте.
- `server.base_url` должен быть `https://headscale.example.net`, без `/admin`.
- Caddy должен отправлять `/admin/*` в Headplane, а остальное в Headscale.
