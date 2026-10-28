---
paths:
  - "**/*.py"
---

# Исключения — где заводить и что отвечать на недоступный объект

Как устроен класс и когда нужен `ValidationError` — в [[custom-exceptions]].

## Через `BaseAPIException`

Исключения приложения — только в `<app>/exceptions.py`, наследник `base.exceptions.BaseAPIException`. Не
`errors.py` и не голый `Exception`: тот выпадает 500-й.

## Недоступный объект — 404 без перечисления id

- Объект из тела или пути пользователю не принадлежит — **404 с кодом `<СУЩНОСТЬ>_NOT_FOUND`**, не 400
  (`check_user_establishments`, `check_tenant_access`). Чек-лист просит 400, ревьюер 404 — делаем 404,
  расхождение в описании PR.
- Id в `detail` не перечислять, все причины (нет записи, не та роль, нет доступа) — одним ответом: иначе
  видно, что чужая запись существует.
- `field_name` — четвёртым аргументом, чтобы фронту было к чему привязать ошибку.
- Код — в `exceptions.py` приложения, чья сущность (`USER_NOT_FOUND` — в `UserException`).

```python
raise UserException(UserException.USER_NOT_FOUND, 'Сотрудник не найден.', status.HTTP_404_NOT_FOUND, 'admin_ids')
```
