# Сервисный слой

**Когда применять:** при выделении бизнес-логики из ViewSet/Serializer; при добавлении транзакций, логирования или сложной фильтрации.

**Главный принцип:** бизнес-логика живёт в сервисном слое. Не в приватных методах ViewSet/Serializer. Не в сигналах.

---

## Утилита без состояния — функция

**Куда:** `base/services.py` (общие для всех приложений).

**Правило:**
- Type hints — на всех параметрах и возвращаемом значении.
- Early return для суперюзера/владельца — сокращает вложенность.
- Дефолтные значения в параметрах — задел на будущий рефакторинг терминологии.
- Q-объекты inline, **не** выносить в именованные переменные.
- Docstring с `:param` при необходимости.

**Пример:**

```python
def filter_by_outlet(
    user: User | AbstractUser | AnonymousUser,
    queryset: QuerySet,
    field_name: str = 'outlet_id',
    is_primary: bool = False,
    in_organization: bool = False,
) -> QuerySet:
    """Фильтр queryset по локациям пользователя."""
    if user.is_superuser or user.role == user.Roles.OWNER:
        return queryset
    ...
```

---

## Доменная логика — класс `*Session`

**Куда:** `app/services/services.py` каждого Django-приложения (`RoleSession`, `UserSession`, `ShiftSession`, …).

**Правило:**
- Один класс на домен.
- Все методы — **публичные** (приватных `_method` не должно быть нигде в кодовой базе).
- Bulk-операции (`bulk_create`/`bulk_update`) предпочтительнее циклов с одиночными `create()`.
- Сложная scope-логика (`get_team_queryset` с org-фильтрацией) — тоже сюда.

**Пример:**

```python
class RoleSession:
    def create_role_permissions(self, role: Role, perms: list[dict]) -> None:
        """Создаёт M2M role_permissions для роли."""
        RolePermission.objects.bulk_create([
            RolePermission(role=role, **rp) for rp in perms
        ])
```

---

## Сервис с `request` — для аудит-лога и транзакций

**Когда применять:** действие требует аудит-лога (IP, User-Agent) или транзакционной обвязки.

**Правило:**
- `request` передаётся **в конструктор** сервиса.
- `transaction.atomic()` — **внутри** сервиса, не во view.
- View остаётся тонким — делегирует одним вызовом.
- Для системных вызовов (Celery) — `request=None` / `user=None` должны корректно обрабатываться.

**Пример:**

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


# View — тонкий
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        WorkShiftDeleteService(request).delete(instance)
        return Response(status=204)
```

**Почему `transaction.atomic()` в сервисе:** транзакция охватывает изменение основной сущности + создание лога — обе операции либо коммитятся вместе, либо откатываются. Если оставить во view, в сервисах с побочными операциями (Celery, импорты) транзакция не покроет лог. Перенесено по ревью yolarus (MCA-197).

---

## Что НЕ выносить в сервис

| Ситуация | Куда |
|---|---|
| Фильтрация, общая для многих ViewSet | Функция в `base/services.py` |
| Создание/обновление M2M-связей | `*Session` класс |
| Сложная scope-логика | `*Session` класс |
| Действие повторяется 2+ раз внутри **одной** функции | Вложенная функция-замыкание |
| Тривиальная одно-строчная операция в одном месте | Inline в коде вызова |

Запрет приватных методов касается ViewSet и Serializer. Внутри сервисного класса всё публичное — приватные методы и там избегаем, но граница строже на верхнем уровне.

---

## Связанное

- [[viewset-patterns]] — ViewSet вызывают сервисы для бизнес-логики и scope
- [[serializer-patterns]] — `*Session` создаёт M2M из `create()`/`update()` сериализатора
- [[audit-logging]] — сервис как точка размещения `log_*_action()`
- [[soft-delete]] — `*DeleteService(request)` для soft-delete
- [[backend-style]] — запрет приватных методов, общие правила
