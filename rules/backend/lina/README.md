# rules/backend/lina/

Личные правила Лины. Действуют только когда подключены через её `~/.claude/CLAUDE.md`.

## Что тут есть

### Стиль кода

- [`compact-lines.md`](compact-lines.md) — не дробить строки до 119 символов.
- [`orm-query-style.md`](orm-query-style.md) — сложные queryset по эталону `annotate_roster`: части в переменные, цепочка в скобках.
- [`kwargs-at-call.md`](kwargs-at-call.md) — именованные аргументы при вызове.
- [`naming-and-comments.md`](naming-and-comments.md) — полные имена, короткие docstring, **никаких** MCA-XXX в коде.
- [`typing-everywhere.md`](typing-everywhere.md) — type hints везде, без строковых аннотаций.

### Архитектурные конвенции

- [`exceptions-via-base.md`](exceptions-via-base.md) — `<app>/exceptions.py` + `BaseAPIException`, без `errors.py`.
- [`services-suffix.md`](services-suffix.md) — файлы в `<app>/services/` с суффиксом `_services.py`.
- [`is-primary-for-sensitive-scope.md`](is-primary-for-sensitive-scope.md) — `filter_by_outlet(is_primary=True)` для timeclock и логов.
- [`split-viewsets-by-action.md`](split-viewsets-by-action.md) — при возможности разделять ViewSet'ы по action/ресурсу.

### Тесты

- [`reuse-fixtures-before-adding.md`](reuse-fixtures-before-adding.md) — перед новой фикстурой проверить существующие, не плодить копии.

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
