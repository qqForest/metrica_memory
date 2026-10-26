---
paths:
  - "**/*.py"
---

# Аудит-логирование (`*Log` модели)

**Когда применять:** при добавлении истории изменений к бизнес-сущности.

Эталонные реализации — `WorkShiftLog`, `TimeClockLog`. `TimeClockLog` — более новая версия паттерна; при рефакторинге `WorkShiftLog` перенимать его подходы.

---

## Структура `*Log` модели

**Правило:** парная `*Log` модель содержит:

- `snapshot` — `JSONField` со снимком всех полей после действия.
- `changes` — `JSONField` с diff (`{field: {old, new}}`).
- `action` — choice из `LogActionChoices` (`create`, `update`, `delete`, `clock_in`, `clock_out`, …).
- `initiator_type` — choice из `LogInitiatorTypeChoices` (`manager`, `employee`, `system`).
- `ip_address` — IPv4/IPv6, nullable.
- `user_agent` — `CharField`, nullable.
- FK на родительскую сущность, FK на `User` (nullable для системных вызовов).

`LogActionChoices` и `LogInitiatorTypeChoices` выносятся в `<app>/choices.py` — общие для всех `*Log` модуля.

**Индексы:** составные для эффективных запросов по истории.

```python
class Meta:
    indexes = [
        models.Index(fields=['timeclock', '-created_at']),
        models.Index(fields=['user', '-created_at']),
        models.Index(fields=['action', '-created_at']),
    ]
```

---

## Snapshot-before-mutation

**Правило:** snapshot создаётся **до** мутации, передаётся как `dict`. Diff сравнивает два dict — **не** повторный `objects.get()` из БД.

```python
# Хорошо — один запрос на чтение (исходный объект уже загружен)
old_snapshot = create_snapshot(timeclock)

timeclock.status = new_status
timeclock.save()

new_snapshot = create_snapshot(timeclock)
changes = calculate_changes(old_snapshot, new_snapshot)
log_action(timeclock, action='update', snapshot=new_snapshot, changes=changes)


# Плохо — повторный SQL для получения old state
old = TimeClock.objects.get(pk=timeclock.pk)   # +1 запрос
timeclock.status = new_status
timeclock.save()
```

**Почему:** паттерн с повторным `objects.get()` создаёт лишний SQL-запрос на каждую операцию. Snapshot-before-mutation отдаёт `old` бесплатно — объект уже загружен.

---

## Динамическая генерация из `_meta.concrete_fields`

**Правило:** snapshot и список полей для diff генерируются динамически — **не** ручным перечислением.

```python
def create_snapshot(instance):
    snapshot = {}
    for f in instance._meta.concrete_fields:
        val = f.value_from_object(instance)
        if isinstance(val, datetime):
            val = val.isoformat()
        elif isinstance(val, Decimal):
            val = str(val)
        snapshot[f.attname] = val
    return snapshot
```

Из diff исключать неизменяемые: `id`, `created_at`, `updated_at`, FK на родителя.

**Почему:** при добавлении новых полей в модель (`start_ltd`, `end_ltd`, …) они автоматически попадают в snapshot и diff. Не нужно править логирование при каждом расширении модели — частая причина забытых полей в аудите.

---

## Транзакционность

**Правило:** создание лога **не** оборачивается в `try/except`. Если запись лога не удалась — откатывается и основная операция.

```python
with transaction.atomic():
    timeclock.clock_out_time = now
    timeclock.save()
    log_action(timeclock, action='clock_out', ...)
```

**Почему:** аудит-лог критичен — операция без лога недопустима. Лучше получить 500 и откатить, чем потерять историю.

Дополнительный `logger.info` внутри `log_action` **не нужен** — если лог записан в БД, дополнительное логирование в stdout избыточно.

---

## Обработка `user=None` (системные вызовы)

**Правило:** любой код логирования, вызываемый из Celery-задач, должен корректно обрабатывать `user=None` / `request=None`.

```python
def determine_initiator_type(user):
    if user is None:
        return 'system'
    if user.is_staff or user.is_superuser:
        return 'manager'
    return 'employee'


def log_work_shift_action(action, work_shift, request, snapshot=None):
    user = request.user if request else None
    ip = get_ip(request) if request else None
    ua = request.META.get('HTTP_USER_AGENT', '') if request else ''
    initiator_type = determine_initiator_type(user)
    ...
```

**Почему:** `auto_close_shifts()` и другие Celery-задачи вызываются без пользователя. Ранее `log_work_shift_action` обращался к `user.is_staff` без проверки на `None` — падал с `AttributeError`. Проверка `user is None` обязательна **перед** обращением к атрибутам.

---

## Размещение логирования — в сервисе, не во view

**Правило:** аудит-логирование (`log_*_action`) **внутри** сервисного класса. `request` передаётся в конструктор. View остаётся тонким.

См. [[service-layer]] — `WorkShiftDeleteService(request).delete(instance)`.

**Почему:** при переносе операции из view в Celery-задачу или импорт-flow логирование уже на месте — переносить нечего. View с логированием — это связывание HTTP-слоя с бизнес-логикой.

---

## Различия `initiator_type`

- `WorkShiftLog` проверяет `is_staff` (логирует только действия менеджеров).
- `TimeClockLog` **не** проверяет `is_staff/is_superuser` — важно логировать clock-in/out от самих сотрудников, иначе теряется половина истории.

Выбор зависит от семантики сущности.

---

## Связанное

- [[service-layer]] — сервис с `request` в конструкторе
- [[soft-delete]] — логирование soft-delete как действия
- [[django-migration-pitfalls]] — FK fix в `*Log` модели (исторические модели)
- [[backend-style]] — общие правила
