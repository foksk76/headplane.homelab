Язык: [English](../06-backup-and-restore.md) | [Русский](06-backup-and-restore.md)

# Резервное копирование и восстановление

Эта инструкция описывает схему резервного копирования и восстановления перед
изменениями в Headplane, Headscale или обратном прокси на рабочем VPS.

## Цель

Сохранить точку отката для конфигурации и локального состояния перед правкой:

- `/etc/headplane`
- `/etc/headscale`
- `/etc/caddy/Caddyfile`
- `headplane.service`
- SQLite-баз Headplane и Headscale

## Когда делать резервную копию

Снимайте новую резервную копию перед:

- включением единого входа или изменением OIDC
- правкой маршрутов Caddy
- правкой служб `systemd`
- изменением конфигурации Headscale или Headplane на уже используемом хосте

## Рекомендуемое имя архива

Используйте имя, в котором есть репозиторий, хост и отметка времени в UTC:

```text
headplane.homelab__headscale.example.net__YYYYMMDDTHHMMSSZ-backup.tar.gz
```

Пример:

```text
headplane.homelab__headscale.example.net__20260513T115434Z-backup.tar.gz
```

## Создайте резервную копию на целевом VPS

```bash
backup_name='headplane.homelab__headscale.example.net__20260513T115434Z-backup.tar.gz'

tar -czf "/root/${backup_name}" \
  --ignore-failed-read \
  --warning=no-file-changed \
  --absolute-names \
  /etc/headplane \
  /etc/headscale \
  /etc/caddy/Caddyfile \
  /etc/systemd/system/headplane.service \
  /usr/lib/systemd/system/headscale.service \
  /var/lib/headplane/hp_persist.db \
  /var/lib/headscale/db.sqlite \
  /var/lib/headscale/noise_private.key \
  /var/lib/headscale/derp_server_private.key
```

Замечания:

- `--ignore-failed-read` полезен, потому что не на каждом хосте есть все
  дополнительные файлы
- отсутствие дополнительного DERP-ключа не считается аварией
- архив нужен для отката, а не для прогулок по публичным сервисам

## Скопируйте архив на хост сборки

Хорошая привычка — держать вторую копию вне целевой машины:

```bash
mkdir -p /root/backups
scp "root@headscale.example.net:/root/${backup_name}" /root/backups/
sha256sum "/root/backups/${backup_name}"
```

## Быстрая проверка архива

```bash
tar -tzf "/root/backups/${backup_name}" | sed -n '1,80p'
```

Убедитесь, что в списке есть ожидаемые конфиги и обе SQLite-базы.

## Восстановите конфигурацию и локальное состояние

Перед восстановлением остановите службы:

```bash
systemctl stop headplane
systemctl stop caddy
systemctl stop headscale
```

Потом распакуйте архив:

```bash
tar -xzf "/root/${backup_name}" -P
```

Дальше перечитайте `systemd` и снова поднимите службы:

```bash
systemctl daemon-reload
systemctl start headscale
systemctl start headplane
systemctl start caddy
```

## Проверки после восстановления

```bash
systemctl status headscale --no-pager
systemctl status headplane --no-pager
systemctl status caddy --no-pager

curl -I http://127.0.0.1:8080/health
curl -I https://headscale.example.net/admin/
```

Ожидаемая картина:

- Headscale в состоянии `active`
- Headplane в состоянии `active`
- Caddy в состоянии `active`
- `/health` отвечает через Headscale
- `/admin/` снова попадает в Headplane

## Что эта инструкция не восстанавливает

- состояние внешнего поставщика удостоверений
- DNS-записи
- сертификаты, которыми управляют вне архивируемых файлов
- изменения данных Headscale, сделанные уже после резервной копии

Последний пункт важен: восстановление `db.sqlite` откатывает состояние
Headscale к моменту создания архива. Делайте это только когда действительно
хотите нажать красную кнопку.

## Навигация

Назад: [Включение единого входа через OIDC](05-enable-sso-oidc.md) | Вперед: [Диагностика](07-troubleshooting.md)
