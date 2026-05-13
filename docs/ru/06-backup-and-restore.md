Язык: [English](../06-backup-and-restore.md) | [Русский](06-backup-and-restore.md)

# Резервное копирование и восстановление

Эта инструкция описывает схему резервного копирования и восстановления перед
изменениями в Headplane, Headscale или обратном прокси на рабочем VPS.

> Статус: повторно проверено на живом откате из резервной копии, снятой до
> включения OIDC, на Debian 13. В проверенный сценарий вошли очистка более
> новых файлов локального поставщика удостоверений перед восстановлением и
> подтверждение того, что ключи API Headscale, созданные уже после резервной
> копии, перестают работать после возврата `db.sqlite`.

## Цель

Сохранить точку отката для конфигурации и локального состояния перед правкой:

- `/etc/headplane`
- `/etc/headscale`
- `/etc/caddy/Caddyfile`
- `headplane.service`
- SQLite-баз Headplane и Headscale
- дополнительных файлов локального поставщика удостоверений, если OIDC
  обслуживается через такой вспомогательный сервис, как Dex

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
  /etc/dex \
  /etc/systemd/system/headplane.service \
  /usr/lib/systemd/system/headscale.service \
  /etc/systemd/system/dex.service \
  /usr/local/bin/dex \
  /var/lib/headplane/hp_persist.db \
  /var/lib/headscale/db.sqlite \
  /var/lib/headscale/noise_private.key \
  /var/lib/headscale/derp_server_private.key
```

Замечания:

- `--ignore-failed-read` полезен, потому что не на каждом хосте есть все
  дополнительные файлы
- отсутствие дополнительного DERP-ключа не считается аварией
- отсутствие `/etc/dex`, `dex.service` или `/usr/local/bin/dex` на машине,
  где локальный поставщик удостоверений никогда не поднимался, тоже нормально
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
systemctl stop dex || true
systemctl stop headplane
systemctl stop caddy
systemctl stop headscale
```

Если откат идет с более нового состояния, где OIDC уже был включен, на более
старый архив, снятый до этого, сначала удалите новые OIDC-файлы. Простая
распаковка старого архива не удаляет более поздние хвосты сама по себе.

Проверенный пример очистки:

```bash
rm -rf /etc/dex
rm -f /etc/systemd/system/dex.service
rm -f /usr/local/bin/dex
rm -f /etc/headplane/secrets/oidc_client_secret
rm -f /etc/headplane/oidc_client_secret
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
- если архив был снят до включения OIDC, на странице входа больше нет элементов
  единого входа

Если вы восстановили более старую базу Headscale, дополнительно проверьте, что
ключи API, выпущенные уже после отметки времени этой резервной копии, больше не
работают. Это ожидаемое поведение и хороший признак того, что откат реально
вернул локальное состояние назад.

## Что эта инструкция не восстанавливает

- состояние внешнего поставщика удостоверений
- DNS-записи
- сертификаты, которыми управляют вне архивируемых файлов
- изменения данных Headscale, сделанные уже после резервной копии

Последний пункт важен: восстановление `db.sqlite` откатывает состояние
Headscale к моменту создания архива. Делайте это только когда действительно
хотите нажать красную кнопку.

## Навигация

Назад: [Включение единого входа через OIDC](05-enable-sso-oidc.md) | Вперед: [Обновление Headplane](07-upgrade-headplane.md)
