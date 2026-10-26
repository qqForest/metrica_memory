---
paths:
  - "**/*.py"
---

# Кастомные исключения — через BaseAPIException

**Когда применять:** при добавлении нового бизнес-исключения в любое приложение.
Как устроен класс и когда нужен `ValidationError` — в [[custom-exceptions]].

**Правило:**

- Исключения приложения — только в `<app>/exceptions.py`. **Не** создавать отдельный `errors.py`.
- Наследовать от `base.exceptions.BaseAPIException`, **не** от голого `Exception`.

```python
# Плохо — отдельный errors.py и наследование от Exception
# shifts/errors.py
class TimeClockError(Exception):
    ...
```

**Почему:** `BaseAPIException` даёт правильное поведение DRF: статус-код, JSON-ответ, перехват
middleware. Наследник `Exception` выпадает как 500 вместо понятного статус-кода. Один
`exceptions.py` на приложение — все ошибки в одном месте; в проекте так сделано везде (11 приложений, ни
одного `errors.py`).

## Связанное

- [[custom-exceptions]] — структура класса, коды, `ValidationError` vs `*Exception`
- [[service-layer]] — где выкидывать исключения
