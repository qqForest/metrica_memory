---
paths:
  - "**/services*.py"
  - "**/services/**"
  - "**/models*.py"
  - "**/models/**"
---

# Soft delete — паттерн

**Когда применять:** при добавлении soft-delete к существующей сущности или при работе с уже soft-deleted моделями (`WorkShift`, `TimeClock`, `ExternalMetric`).

Актуальный эталон — `ExternalMetric` (MCA-320). Старые реализации `WorkShift` (MCA-185, MCA-197) и `TimeClock` сделаны через `get_queryset_services.py` — это **legacy**, для новых сущностей не повторять.

---

## Структура полей

**Правило:** в модель добавляются два поля.

```python
class WorkShift(models.Model):
    ...
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        User, null=True, blank=True, on_delete=models.SET_NULL,
        verbose_name='Удалил',
    )
```

`deleted_by` — `SET_NULL`, чтобы удаление пользователя не уничтожало историю удалений.

### `related_name` — по умолчанию, не запрещать

**Правило:** не указывать `related_name='+'` без явной причины. Дефолтный reverse-accessor (`user.externalmetric_set` и подобные) полезен для отладки/админки/случайных запросов.

**Почему:** на ревью MCA-320 Руслан явно отметил `related_name='+'` на `ExternalMetric.deleted_by` комментарием «зачем запрет?» — паттерн скопировали из этого правила, и он оказался лишним ограничением.

```python
# Раньше предлагали — больше так не делать без причины
deleted_by = models.ForeignKey(
    User, null=True, blank=True, on_delete=models.SET_NULL,
    related_name='+',   # ← убрать, если нет конкретной нужды
)

# Дефолт — оставлять Django-овский reverse
deleted_by = models.ForeignKey(
    User, null=True, blank=True, on_delete=models.SET_NULL,
    verbose_name='Удалил',
)
```

Запрет (`related_name='+'`) допустим **только** когда:
- На одну User несколько FK от одной модели и нужно избежать коллизии имён (тогда осмысленные `related_name` каждому, а не `'+'`).
- Reverse-набор гарантированно никогда не понадобится **и** не нужен для отладки/админки.

В большинстве кейсов — оставляем дефолт.

---

## Главное правило: фильтр `deleted_at__isnull=True` в `get_queryset` override

**НЕ ставить** фильтр `deleted_at__isnull=True` на class-level queryset ViewSet.

**НЕ дублировать** фильтр по каждому scope-методу (`get_team_queryset`, `get_own_queryset`) — это даёт повтор и сбивает читателя.

**Правильно** — переопределить `get_queryset` поверх `ScopedViewSetMixin`, добавить фильтр одним местом с проверкой `is_superuser`. Суперюзер видит soft-deleted, все остальные — нет. Пример и ловушка class-level queryset — в [[viewset-patterns]] (orthogonal-фильтры).

**История:** на ревью MCA-320 решение прошло три итерации — `get_team/own` → `get_all_queryset` (с guard `if not is_superuser`) → `get_team/own` → итог `get_queryset` override. Зафиксировано как канон.

---

## Legacy: bypass через `get_all_queryset` + `*GetQuerysetService`

Старые сущности (`WorkShift`) используют выделенный `*GetQuerysetService` и фильтрацию внутри `get_all_queryset`. Это работающий, но **устаревший** паттерн — для новых сущностей не повторять.

| Сущность | Суперюзер видит удалённые? | Механизм |
|---|---|---|
| `ExternalMetric` (новый эталон) | Да | `get_queryset` override с `if not is_superuser` |
| `WorkShift` (legacy) | Да | `get_all_queryset` → `WorkShiftGetQuerysetService.build()` |
| `TimeClock` (legacy) | Нет | `get_all_queryset` не определён; фильтр в scope-методах |

`TimeClock` — намеренно: финальное состояние таймклока скрыто от всех. Если для новой сущности нужно «суперюзер тоже не видит удалённые» — добавить guard в `get_queryset` override (или просто не делать `if not is_superuser`).

**Ловушка отладки:** если падают тесты на `/work-shifts/{id}/` и `/timeclocks/{id}/` одновременно — смотреть на URL в трейсбеке. 404 на TimeClock от суперюзера — корректно, **не баг**.

---

## Удаление через сервис

**Правило:** soft-delete и аудит-лог — внутри `*DeleteService(request)`. View остаётся тонким. Пример `WorkShiftDeleteService` — в [[service-layer]].

`transaction.atomic()` — в сервисе. `request` — в конструктор (для IP/User-Agent в аудит-логе). См. [[audit-logging]].

---

## Reverse relations — легко пропустить

При аудите кодовой базы (где нужно добавить `deleted_at__isnull=True`) **не** ограничиваться поиском `WorkShift.objects`. Проверить **обратные** связи:

```python
# Эти места легко пропустить
tag.workshifts.filter(...)
record.workshifts.count()
type.workshifts.exists()
```

Паттерн `.workshifts.` (или `.<related_name>.`) искать отдельным grep.

---

## Тестирование

**Сценарии:**

1. `test_destroy_soft_delete_fields` — DELETE + GET суперюзером, проверить `deleted_at` и `deleted_by_id` в JSON.
2. `test_destroy_404_already_deleted` — повторный DELETE по уже soft-deleted смене возвращает 404 (не 204).
3. `test_partial_update_400_shift_deleted` — обновление soft-deleted смены возвращает 404 (если не суперюзер).

Юнит-тесты сервиса удалять — только API-тесты (см. [[test-patterns-crud]]).

---

## Связанное

- [[viewset-patterns]] — class-level queryset ломает scope-bypass
- [[service-layer]] — `*DeleteService` с `request` в конструкторе
- [[audit-logging]] — логирование soft-delete как действия
