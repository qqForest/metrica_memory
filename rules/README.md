# rules/

Короткие правила-выжимки, которые Claude должен соблюдать. Подключаются через `~/.claude/CLAUDE.md` ссылкой на нужные файлы.

## Структура

- [`karpathy.md`](./karpathy.md) — четыре базовых принципа, обязательны для всех.
- `shared/` — общекомандные правила (см. ниже).
- `backend/<имя>/` — личные правила, которые применяет только владелец.

## Что лежит в `shared/`

**Язык и процесс:**
- [`ru-language.md`](./shared/ru-language.md) — отвечаем по-русски, идентификаторы оригинальные, орфография корректная.
- [`git-workflow.md`](./shared/git-workflow.md) — ветки, Draft PR, лейблы, 400 строк, Squash & Merge.

**Бэкенд — общий стиль:**
- [`backend-style.md`](./shared/backend-style.md) — импорты, queryset, ORM-антипаттерны, запрет приватных методов, URL-структура, docstrings.

**Бэкенд — DRF паттерны:**
- [`viewset-patterns.md`](./shared/viewset-patterns.md) — 4 типа ViewSet, `get_serializer_class`, Input/Output, `IsSuperuser`, ловушки class-level queryset и `get_all_queryset`.
- [`serializer-patterns.md`](./shared/serializer-patterns.md) — отдельный сериализатор на контекст, `SerializerMethodField`, исключение для простых сущностей.
- [`service-layer.md`](./shared/service-layer.md) — функции в `base/services.py`, `*Session` классы, сервис с `request` для аудит-лога.

**Бэкенд — специфичные паттерны:**
- [`soft-delete.md`](./shared/soft-delete.md) — `deleted_at`/`deleted_by_id`, фильтр в `get_queryset_services.py`, bypass через `get_all_queryset`.
- [`audit-logging.md`](./shared/audit-logging.md) — парная `*Log` модель, snapshot-before-mutation, динамика из `_meta.concrete_fields`, `user=None`.
- [`django-migration-pitfalls.md`](./shared/django-migration-pitfalls.md) — исторические модели, `update_fields`, замена FK.

**Бэкенд — тесты:**
- [`test-patterns-crud.md`](./shared/test-patterns-crud.md) — один файл = один action, `BaseTestCase` + async, фабрики, матрица кодов, только API-тесты.
- [`test-patterns-list-filters.md`](./shared/test-patterns-list-filters.md) — три файла (access/filters/complex), superuser для фильтров, минимум ассертов.
- [`tests-sync-to-async.md`](./shared/tests-sync-to-async.md) — `sync_to_async` для синхронных Celery-вызовов в async-тестах.

## Как подключить себе

В твоём `~/.claude/CLAUDE.md` уже есть строчки вида:

```markdown
Следуй [`rules/karpathy.md`](.../rules/karpathy.md).
Командные правила: [`rules/shared/`](.../rules/shared/).
Личные правила: [`rules/backend/<имя>/`](.../rules/backend/<имя>/).
```

Claude автоматически читает связанные файлы, когда они упомянуты в CLAUDE.md.

## Формат правила

```markdown
# Название правила

**Когда применять:** конкретная ситуация.

**Правило:** одна-две строки, что делать.

**Пример:** короткий блок кода (по возможности).

**Почему:** мотивация. Это самое важное — даёт возможность судить пограничные случаи.
```

Если правило длиннее половины экрана — скорее всего, его пора разбить на несколько.
