---
paths:
  - "**/*.py"
---

# rules/backend/lina/

Личные правила Лины. Действуют только когда подключены через её `~/.claude/CLAUDE.md`.

## Что тут есть

### Стиль кода

- [`compact-lines.md`](compact-lines.md) — не дробить строки до 119 символов.
- [`orm-query-style.md`](orm-query-style.md) — сложные queryset по эталону `annotate_roster`: части в переменные, цепочка в скобках.
- [`kwargs-at-call.md`](kwargs-at-call.md) — аргументы позиционно, по имени только после пропущенного необязательного.
- [`naming-and-comments.md`](naming-and-comments.md) — полные имена, короткие docstring, **никаких** MCA-XXX в коде.
- [`typing-everywhere.md`](typing-everywhere.md) — type hints везде, без строковых аннотаций.
- [`no-redundant-checks.md`](no-redundant-checks.md) — повторы условий сводить, ветки под невозможное не писать.
- [`literal-over-single-use-constant.md`](literal-over-single-use-constant.md) — одноразовое значение литералом, без косвенности ради длины строки.
- [`query-over-python-loops.md`](query-over-python-loops.md) — один запрос вместо цикла в Python, разница множеств через `exclude`.

### Архитектурные конвенции

- [`exceptions-via-base.md`](exceptions-via-base.md) — `<app>/exceptions.py` + `BaseAPIException`, без `errors.py`.
- [`services-suffix.md`](services-suffix.md) — файлы в `<app>/services/` с суффиксом `_services.py`.
- [`split-viewsets-by-action.md`](split-viewsets-by-action.md) — при возможности разделять ViewSet'ы по action/ресурсу.
- [`classes-before-functions.md`](classes-before-functions.md) — классы выше функций, «спустить под класс» = переставить.
- [`no-reexport-init.md`](no-reexport-init.md) — импорт напрямую из модуля, без реэкспортов в `__init__.py`.
- [`atomic-related-writes.md`](atomic-related-writes.md) — связанные записи в БД в одной `transaction.atomic()`.
- [`select-for-update-by-ids.md`](select-for-update-by-ids.md) — блокировка по id, не поверх queryset с `distinct()`.
- [`extend-schema-responses.md`](extend-schema-responses.md) — `responses()` и `codes=`, параметры в `<app>/parameters.py`.
- [`thin-views.md`](thin-views.md) — view оркестрирует, query-параметры через сериализатор.
- [`write-access-in-queryset.md`](write-access-in-queryset.md) — запрет на запись в `get_queryset`, не в `validate`; 404.

### Тесты

- [`reuse-fixtures-before-adding.md`](reuse-fixtures-before-adding.md) — перед новой фикстурой проверить существующие, не плодить копии.
- [`what-not-to-test.md`](what-not-to-test.md) — эффект сценарием через API, без тестов на удалённое, без тестовых пакетов и фильтров варнингов.

### Процесс

- [`pr-description-style.md`](pr-description-style.md) — выжимка для description, один экран без скролла.
- [`pr-review-comments-style.md`](pr-review-comments-style.md) — короткие inline-комменты без эмодзи и ярлыков.

## Шаблон

```markdown
# Название

**Когда применять:** …

**Правило:** …

**Почему:** …
```
