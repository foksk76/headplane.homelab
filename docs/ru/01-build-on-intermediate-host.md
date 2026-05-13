Язык: [English](../01-build-on-intermediate-host.md) | [Русский](01-build-on-intermediate-host.md)

# Сборка на промежуточном хосте

Эта инструкция описывает сборку Headplane `v0.6.2` на промежуточном хосте,
например `build-host.internal`.

Имя хоста в примерах заменяемое. Подойдет любой доступный хост сборки, если на
нем хватает диска, памяти и есть исходящий доступ к источникам пакетов.

## Цель

Получить исполняемый архив, который можно перенести на VPS без повторной сборки.

## Условия на хосте

- Debian 13
- оболочка `root` или доступ через `sudo`
- исходящий доступ к GitHub, реестру npm и модулям Go

## Замечание по версии апстрима

По состоянию на 13 мая 2026 года последним стабильным релизом Headplane
остается `v0.6.2`. На публичном сайте документации уже местами видно более
новые изменения, поэтому для точного поведения `v0.6.2` этот репозиторий
опирается на исходники именно релиза с соответствующим тегом.

## Установите инструменты сборки

```bash
apt update
apt install -y curl git golang ca-certificates gnupg apt-transport-https
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
corepack enable
corepack prepare pnpm@10.4.0 --activate
```

Проверьте версии:

```bash
git --version
go version
node -v
pnpm --version
```

Ожидаемый пример:

```text
git version 2.47.3
go version go1.24.4 linux/amd64
v22.22.2
10.4.0
```

Ограничения из `package.json` у upstream `v0.6.2`:

- Node.js `>=22.18 <23`
- pnpm `>=10.4 <11`
- Go `1.23+` — практический минимум для `./build.sh`, потому что скрипт берет
  `wasm_exec.js` из нового пути внутри набора сборочных инструментов Go

## Склонируйте исходный Headplane

```bash
cd /root
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
```

Дополнительная проверка:

```bash
git describe --tags --exact-match
```

Ожидаемый вывод:

```text
v0.6.2
```

## Установите зависимости и соберите проект

Используйте полный сценарий сборки, а не только `pnpm build`:

```bash
pnpm install
./build.sh
```

Зачем нужен `./build.sh`:

- собирает серверную и клиентскую веб-часть
- собирает `build/hp_agent`
- собирает `build/hp_healthcheck`
- собирает `public/hp_ssh.wasm`
- оставляет в `node_modules` только рабочие зависимости

## Проверьте результат сборки

```bash
ls -lh \
  build/server/index.js \
  build/hp_agent \
  build/hp_healthcheck \
  public/hp_ssh.wasm \
  public/wasm_exec.js
```

## Создайте исполняемый архив

Проверенная установка использовала архив, в котором сохранен каталог миграций.

```bash
cd /root
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

Дополнительно можно посчитать контрольную сумму:

```bash
sha256sum /root/headplane-v0.6.2-runtime-r2.tar.gz
```

Пример из проверенной установки:

```text
9582cacf2222a2a55a6ab174ebd0494a23a67d0cf9881a45e5ad765db98b2a1d  /root/headplane-v0.6.2-runtime-r2.tar.gz
```

## Почему нужен `drizzle/`

Headplane запускает миграции базы данных при старте. Если
`drizzle/meta/_journal.json` отсутствует, запуск падает с ошибкой:

```text
Error: Can't find meta/_journal.json file
```

Этот сбой уже ловили на проверенной установке, поэтому `drizzle/` входит в
чек-лист упаковки. Да, миграции тоже хотят ехать в прод, кто бы мог подумать.

## Навигация

Назад: [Быстрый старт](../../README.ru.md#быстрый-старт) | Вперед: [Перенос исполняемого архива](02-transfer-runtime-artifact.md)
