# Никогда не писать миграции руками

**Когда применять:** при необходимости создать миграцию.

**Правило:** только через `manage.py makemigrations`. Если падает с `EOFError` (интерактивный prompt Django — типичный случай `auto_now_add` на существующей таблице, нужен дефолт для существующих строк), запускать через `-T` и пробрасывать ответы через stdin.

Не использовать `--noinput` для миграций с `auto_now_add` — Django откажется и сломает поток.

**Пример:**

```bash
printf '1\ntimezone.now\n1\ntimezone.now\n' | docker compose exec -T app_metrica python3 manage.py makemigrations <app>
```

`1` = «one-off default now», `timezone.now` = дефолт. Пара ответов на каждое поле, требующее prompt. Лишние строки stdin Django проглотит.

**Почему:** ручные миграции легко расходятся с тем, что сгенерил бы Django (порядок операций, имена индексов, флаги `preserve_default`), и потом это всплывает на боевой БД. Команда — единственный источник правды. Связано: [[shared/django-migration-pitfalls]], [[migrations-no-data]].
