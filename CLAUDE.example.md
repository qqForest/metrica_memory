<!--
  Глобальный шаблон для ~/.claude/CLAUDE.md.

  Как пользоваться:
  1. cp CLAUDE.example.md ~/.claude/CLAUDE.md
  2. Заменить все {{ПЛЕЙСХОЛДЕРЫ}} на свои значения.
  3. Удалить блоки в фигурных скобках, которые тебе не подходят (роли и т.п.).
  4. Личный файл ~/.claude/CLAUDE.md не коммитится — он только твой.
-->

# Профиль

- **Имя:** {{NAME}}
- **Роль:** {{ROLE}}  <!-- backend / frontend / qa / design / pm -->
- **Команда:** Metrica
- **Tracker handle:** {{TRACKER_LOGIN}}  <!-- логин в Яндекс Трекере, для назначения задач -->
- **GitHub:** {{GITHUB_LOGIN}}
- **Часовой пояс:** {{TIMEZONE}}  <!-- например, Europe/Moscow -->

# Язык

Всегда отвечай по-русски. Технические термины и идентификаторы оставляй в оригинальной форме. Сохраняй орфографию: никогда не заменяй буквы с диакритикой на ASCII-аналоги (`ё` ≠ `е`).

# Базовые принципы

Следуй четырём принципам из [`rules/karpathy.md`](../Projects/metrica_memory/rules/karpathy.md):

1. **Think Before Coding** — озвучь допущения, спроси если не уверен, не выбирай молча между интерпретациями.
2. **Simplicity First** — минимум кода, решающего задачу. Без спекулятивной гибкости и обработки невозможных ошибок.
3. **Surgical Changes** — трогай только то, что напрямую связано с задачей. Не рефакторь по пути.
4. **Goal-Driven Execution** — переведи размытую задачу в проверяемый критерий «готово» и докажи, что он выполнен.

Командные правила: [`rules/shared/`](../Projects/metrica_memory/rules/shared/).

<!--
  ROLE-SPECIFIC БЛОКИ.
  Оставь блок своей роли, остальные — удали.
-->

{{#if ROLE == backend}}
# Бэкенд-специфика

- Стек Metrica: Python 3.12+ / Django 5.2 / DRF / PostgreSQL. Менеджер пакетов — **uv**.
- Перед изменением миграций — обязательно посмотреть текущее состояние схемы.
- Для интеграционных тестов — реальная БД (PostgreSQL), не моки и не SQLite.
- Чек-лист сдачи задачи: см. `/pr-metrica`.

Командные backend-правила (читать перед написанием кода):

- [`backend-style.md`](../Projects/metrica_memory/rules/shared/backend-style.md) — импорты, queryset, ORM-антипаттерны, запрет приватных методов.
- [`viewset-patterns.md`](../Projects/metrica_memory/rules/shared/viewset-patterns.md) — 4 типа ViewSet, ловушки class-level queryset.
- [`serializer-patterns.md`](../Projects/metrica_memory/rules/shared/serializer-patterns.md) — отдельный сериализатор на контекст.
- [`service-layer.md`](../Projects/metrica_memory/rules/shared/service-layer.md) — `*Session` классы, `request` в конструктор сервиса.
- [`soft-delete.md`](../Projects/metrica_memory/rules/shared/soft-delete.md) — фильтр **только** в `get_queryset_services.py`.
- [`audit-logging.md`](../Projects/metrica_memory/rules/shared/audit-logging.md) — snapshot-before-mutation, `*Log` модели.
- [`django-migration-pitfalls.md`](../Projects/metrica_memory/rules/shared/django-migration-pitfalls.md) — исторические модели, `update_fields`.
- [`test-patterns-crud.md`](../Projects/metrica_memory/rules/shared/test-patterns-crud.md) — `BaseTestCase` + async, фабрики, матрица кодов.
- [`test-patterns-list-filters.md`](../Projects/metrica_memory/rules/shared/test-patterns-list-filters.md) — access/filters/complex.
- [`tests-sync-to-async.md`](../Projects/metrica_memory/rules/shared/tests-sync-to-async.md) — для синхронных Celery-вызовов в тестах.

Мои личные правила: [`rules/backend/{{NAME_LOWER}}/`](../Projects/metrica_memory/rules/backend/{{NAME_LOWER}}/).
{{/if}}

{{#if ROLE == frontend}}
# Фронтенд-специфика

- Стек Metrica: <!-- TODO: дополнить когда появится фронтендер -->
- Перед UI-изменениями — запустить dev-сервер и проверить в браузере.

Мои личные правила: [`rules/frontend/{{NAME_LOWER}}/`](../Projects/metrica_memory/rules/frontend/{{NAME_LOWER}}/).
{{/if}}

{{#if ROLE == qa}}
# QA-специфика

- <!-- TODO: дополнить когда появится QA -->

Мои личные правила: [`rules/qa/{{NAME_LOWER}}/`](../Projects/metrica_memory/rules/qa/{{NAME_LOWER}}/).
{{/if}}

# Личные предпочтения

<!--
  Сюда пиши то, что специфично только для тебя:
  - стиль коммуникации (терсе/подробно),
  - привычные инструменты,
  - запреты («не используй emoji», «не пиши TODO в коде»),
  - формат ответов.
-->

- {{PREF_1}}
- {{PREF_2}}

# Контекст рабочих ресурсов

- **Проекты:** `~/Projects/metrica-erp`, `~/Projects/metrica-front` <!-- скорректируй под себя -->
- **Серверы:** dev — `93.77.187.173`, prod — `158.160.55.133`. Sudo-доступ через скиллы `server_dev-sudo`, `server_prod-sudo`.
- **Tracker:** очередь `MCA` (Metrica Backend), задачи переводятся в «Ревью» скиллом `/pr-metrica`.
- **Wiki:** Яндекс Wiki через MCP `yandex-wiki`.
- **Команда:**
  - Ильназ — бэкенд
  - Лина — бэкенд

# Память

Личная файловая память: `~/.claude/projects/-Users-{{USER}}-Projects-metrica-memory/memory/`. Команды на чтение и запись описаны в системной части — пиши `user/feedback/project/reference` записи как обычно.

# Командный репозиторий

`~/Projects/metrica_memory` — общий мозг команды. Если находишь полезное правило/скилл и думаешь «коллега тоже выиграет» — оформляй в PR в `shared/` или хотя бы в `<роль>/<имя>/`, чтобы коллега мог посмотреть.
