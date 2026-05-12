# Soft delete — паттерн

**Когда применять:** при добавлении soft-delete к существующей сущности или при работе с уже soft-deleted моделями (`WorkShift`, `TimeClock`).

Эталонная реализация — `WorkShift` (MCA-185, MCA-197).

---

## Структура полей

**Правило:** в модель добавляются два поля.

```python
class WorkShift(models.Model):
    ...
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        User, null=True, blank=True, on_delete=models.SET_NULL,
        related_name='+',
    )
```

`deleted_by` — `SET_NULL`, чтобы удаление пользователя не уничтожало историю удалений.

---

## Главное правило: фильтр **только** в `get_queryset_services.py`

**НЕ ставить** фильтр `deleted_at__isnull=True` на class-level queryset ViewSet.

```python
# КАТЕГОРИЧЕСКИ НЕЛЬЗЯ — суперюзер получит 404 на soft-deleted смены
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.filter(deleted_at__isnull=True)
```

**Правильно:**

```python
# ViewSet — голый queryset
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.all()

    def get_all_queryset(self, qs):
        return WorkShiftGetQuerysetService().build(qs, self.request.user)


# get_queryset_services.py — фильтр с bypass для суперюзера
def build(qs, user):
    if user.is_superuser or user.is_staff:
        return qs   # суперюзер видит удалённые
    return qs.filter(deleted_at__isnull=True)
```

**Почему:** class-level `queryset` используется `get_object()` как база, и `ScopedViewSetMixin` применяет bypass **поверх** уже отфильтрованного queryset — но не может «отменить» фильтр, заданный на уровне класса. См. [[viewset-patterns]] (ловушка class-level queryset).

---

## Bypass суперюзера требует `get_all_queryset`

**Правило:** bypass суперюзера в `ScopedViewSetMixin` срабатывает **только** если ViewSet определяет `get_all_queryset`. Если метода нет — суперюзер проходит через обычный `get_team_queryset`.

| Сущность | Суперюзер видит удалённые? | Механизм |
|---|---|---|
| `WorkShift` | **Да** | Определён `get_all_queryset` → bypass → `WorkShiftGetQuerysetService.build()` возвращает qs без фильтра |
| `TimeClock` | **Нет** | `get_all_queryset` **не определён** → суперюзер проходит через сервисный слой → `deleted_at__isnull=True` для всех |

Отсутствие `get_all_queryset` — **архитектурное решение**, не забытый метод. Для `TimeClock` суперюзер не видит удалённые записи — это намеренное поведение: финальное состояние таймклока должно быть скрыто.

**Ловушка отладки:** если падают тесты на `/work-shifts/{id}/` и `/timeclocks/{id}/` одновременно — смотреть на URL в трейсбеке. 404 на TimeClock от суперюзера — корректно, **не баг**.

---

## Удаление через сервис

**Правило:** soft-delete и аудит-лог — внутри `*DeleteService(request)`. View остаётся тонким.

```python
class WorkShiftDeleteService:
    def __init__(self, request):
        self.request = request

    def delete(self, work_shift):
        with transaction.atomic():
            old_snapshot = create_snapshot(work_shift)
            work_shift.deleted_at = now()
            work_shift.deleted_by = self.request.user
            work_shift.save()
            log_work_shift_action('delete', work_shift, self.request, snapshot=old_snapshot)


class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        WorkShiftDeleteService(request).delete(instance)
        return Response(status=204)
```

`transaction.atomic()` — в сервисе. `request` — в конструктор (для IP/User-Agent в аудит-логе). См. [[service-layer]], [[audit-logging]].

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
