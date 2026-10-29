---
paths:
  - "**/services*.py"
  - "**/services/**"
---

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
def filter_by_establishments(
    user: User | AbstractUser | AnonymousUser,
    queryset: QuerySet,
    field_name: str = 'establishment_id',
    in_organization: bool = False,
) -> QuerySet:
    """Фильтр queryset по локациям пользователя."""
    if user.is_superuser or user.role == user.Roles.OWNER:
        return queryset
    ...
```

---

## Доменная логика — класс `*Service`

**Куда:** `app/services/services.py` (или `app/services/<entity>_services.py` для одной сущности) каждого Django-приложения (`RoleService`, `UserService`, `ShiftService`, …).

**Правило:**
- Один класс на домен.
- Все методы — **публичные** (приватных `_method` не должно быть нигде в кодовой базе).
- Bulk-операции (`bulk_create`/`bulk_update`) предпочтительнее циклов с одиночными `create()`.
- Сложная scope-логика (`get_team_queryset` с org-фильтрацией) — тоже сюда.

**Пример:**

```python
class RoleService:
    def create_role_permissions(self, role: Role, perms: list[dict]) -> None:
        """Создаёт M2M role_permissions для роли."""
        RolePermission.objects.bulk_create([
            RolePermission(role=role, **rp) for rp in perms
        ])
```

### Legacy: `*Session`

В кодовой базе есть классы `*Session` (`SystemLoaderSettingsSession`, `UserLoaderSettingsSession`, `RoleSession`, …) — это **устаревший** вариант наименования. На ревью MCA-320 Руслан написал: «давай на `ExternalMetricService` поменяем, `Session` — первое что в голову пришло, так уже сервисы пишем почти везде».

**Для новых сервисов** — использовать суффикс `*Service`. Существующие `*Session` — не переименовывать массово, но при правке файла можно заодно переименовать.

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
| Создание/обновление M2M-связей | `*Service` класс |
| Сложная scope-логика | `*Service` класс |
| Действие повторяется 2+ раз внутри **одной** функции | Вложенная функция-замыкание |
| Тривиальная одно-строчная операция в одном месте | Inline в коде вызова |

Запрет приватных методов касается ViewSet и Serializer. Внутри сервисного класса всё публичное — приватные методы и там избегаем, но граница строже на верхнем уровне.

---

## Связанное

- [[viewset-patterns]] — ViewSet вызывают сервисы для бизнес-логики и scope
- [[serializer-patterns]] — `*Service` создаёт M2M из `create()`/`update()` сериализатора
- [[audit-logging]] — сервис как точка размещения `log_*_action()`
- [[soft-delete]] — `*DeleteService(request)` для soft-delete
- [[backend-style]] — запрет приватных методов, общие правила
