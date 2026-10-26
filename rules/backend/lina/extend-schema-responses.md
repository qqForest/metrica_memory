---
paths:
  - "**/*.py"
---

# `@extend_schema` — ответы и параметры

**Когда применять:** при написании `@extend_schema` во вьюсете.

**Правило:** `responses` собирать хелпером `responses(...)` из `base.responses`, не голым словарём.
Коды бизнес-ошибок — через `codes=`, значения — константы из `*Exception(BaseAPIException)`.

```python
# Хорошо
@extend_schema(responses=responses(EventTypeSerializer, codes={status.HTTP_404_NOT_FOUND: [TenantException.TENANT_NOT_FOUND]}))

# Плохо
@extend_schema(responses={200: EventTypeSerializer})
```

**Почему:** хелпер добавляет базовые 400/401/403/405 одной схемой; голый словарь их теряет.
Пример `responses={201: ...}` в [[viewset-patterns]] — устаревший.

## Параметры — в `<app>/parameters.py`

`OpenApiParameter(...)` не писать инлайн в `@extend_schema`, а выносить в `<app>/parameters.py` (образец — `shifts/parameters.py`). Перед созданием проверить готовые, в том числе в `base/parameters.py` и соседних приложениях (например `SCHEDULE_ID_PARAM_OPTIONAL` в `shifts/parameters.py`).
