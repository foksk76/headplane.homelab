Язык: [English](../02-transfer-runtime-artifact.md) | [Русский](02-transfer-runtime-artifact.md)

# Перенос исполняемого архива

Эта инструкция копирует исполняемый архив с промежуточного хоста на VPS.
Имена хостов в примерах замените на свои.

## Рекомендуемый путь переноса

С рабочей станции или любого хоста, который видит обе машины:

```bash
scp root@build-host.internal:/root/headplane-v0.6.2-runtime-r2.tar.gz root@headscale.example.net:/root/
```

## Если переносите с хоста сборки

Сначала проверьте, что SSH-доступ до VPS работает:

```bash
ssh-add -l
ssh root@headscale.example.net hostnamectl
```

## Проверьте архив на VPS

```bash
ssh root@headscale.example.net 'ls -lh /root/headplane-v0.6.2-runtime-r2.tar.gz && sha256sum /root/headplane-v0.6.2-runtime-r2.tar.gz'
```

## Дополнительно посмотрите состав архива

```bash
ssh root@headscale.example.net 'tar -tzf /root/headplane-v0.6.2-runtime-r2.tar.gz | sed -n "1,80p"'
```

В списке должны быть важные файлы:

- `headplane/build/server/index.js`
- `headplane/build/hp_agent`
- `headplane/build/hp_healthcheck`
- `headplane/public/hp_ssh.wasm`
- `headplane/drizzle/meta/_journal.json`

## Навигация

Назад: [Сборка на промежуточном хосте](01-build-on-intermediate-host.md) | Вперед: [Установка на VPS](03-install-on-vps.md)
