---
paths:
  - "**/*.py"
---

# Backend — общий стиль

Сводка ключевых стилистических правил Metrica Backend (Django/DRF, Python 3.12+, uv, pytest). Применять при написании и правке любого backend-кода.

---

## Импорты

**Когда применять:** при любом написании или правке Python-файла.

**Правило:**
- Все импорты — **только на верхнем уровне файла**. Ленивые импорты внутри функций/методов запрещены — circular import = сигнал к рефакторингу, не повод прятать import.
- Только **абсолютные** импорты: `from shifts.models import WorkShift`, не `from .models import WorkShift`.
- Длинные импорты переносятся через `\`, **не** через скобки `()`.

**Пример:**

```python
# Хорошо — в одну строку
from shifts.models import WorkShift, WorkShiftLog

# Хорошо — длинный импорт через backslash
from shifts.models \
    import WorkShift, WorkShiftLog, TimeClock, TimeClockLog, ShiftClaim

# Плохо — относительный
from .models import WorkShift

# Плохо — скобки
from shifts.models import (
    WorkShift, WorkShiftLog
)

# Плохо — внутри функции
def f():
    from shifts.models import WorkShift   # ← запрещено
```

**Почему:** правило зафиксировано Русланом ([[yolarus-reviews]]) в ревью MCA-179. Backslash оставляет diff читаемым (по одной строке на изменение). Абсолютные импорты исключают неоднозначность при перемещении модулей.

---

## QuerySet — ленивый

**Когда применять:** при работе с QuerySet в сервисах, viewsets, миграциях.

**Правило:**
- **Не** конвертировать QuerySet в `list()` заранее. При повторном обращении выполняется новый SQL с актуальными данными.
- Не нужно перечитывать QuerySet после `bulk_create` — он уже актуален.
- Базовый `queryset` во ViewSet — голый `Model.objects.all()`. Никаких `.filter()` на class-level.
- `select_related`/`prefetch_related` добавлять **только** в `list()` и `retrieve()` (для update/destroy ответ = id, тяжёлые JOIN-ы не нужны).

**Пример:**

```python
# Хорошо
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.all()

    def list(self, request, *args, **kwargs):
        self.queryset = self.queryset.select_related('position', 'schedule')
        return super().list(request, *args, **kwargs)

# Плохо — class-level filter ломает scope-bypass суперюзера и soft-delete
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.filter(deleted_at__isnull=True)   # ← НЕТ
```

**Почему:** class-level `queryset` фильтрует **до** `ScopedViewSetMixin`. Bypass суперюзера не снимет фильтр, заданный на уровне класса — суперюзер не увидит soft-deleted записи и получит 404. Подробности — [[soft-delete]].

---

## ORM-антипаттерны

**Когда применять:** при проверке существования + получении объекта; при массовых вставках/обновлениях.

**Правило:**
- `.exists()` + `.get()` → `.first()` + проверка `is None`. Один SQL вместо двух.
- Циклы с `Model.objects.create()` → `bulk_create` / `bulk_update`.

**Пример:**

```python
# Плохо — два SQL-запроса
if qs.filter(pk=pk).exists():
    obj = qs.get(pk=pk)
    ...

# Хорошо — один SQL
obj = qs.filter(pk=pk).first()
if obj is None:
    raise NotFound

# Плохо — N запросов
for rp in perms_data:
    RolePermission.objects.create(role=role, **rp)

# Хорошо — один запрос
RolePermission.objects.bulk_create([
    RolePermission(role=role, **rp) for rp in perms_data
])
```

**Почему:** в обоих случаях экономия SQL-запросов: каждый round-trip к PostgreSQL стоит миллисекунды, на цикле из 100 объектов это уже секунда лишнего ответа API.

---

## Никаких приватных методов в ViewSet/Serializer

**Когда применять:** при выделении логики, которая не помещается в одну-две строки прямого кода.

**Правило:**
- В ViewSet и Serializer **не допускаются** приватные методы (`_method`).
- Утилиты без состояния → функция в `base/services.py`.
- Доменная логика → класс `*Session` в `app/services/services.py`.
- Если действие вызывается 2+ раз внутри одной функции — вынести во **вложенную функцию с замыканием**, даже если это 2 строки.

**Пример:**

```python
# Плохо
class WorkShiftViewSet(ModelViewSet):
    def create(self, request, *args, **kwargs):
        ...
        self._create_audit_log(instance, request)   # ← запрещено

    def _create_audit_log(self, instance, request):
        ...

# Хорошо — логика в сервисе
class WorkShiftViewSet(ModelViewSet):
    def create(self, request, *args, **kwargs):
        ...
        WorkShiftCreateService(request).log(instance)

# Хорошо — вложенная функция при повторе внутри метода
def sync_permissions(modules_data):
    def create_perms(module, perms_data):
        Permission.objects.bulk_create([...])

    for module_data in modules_data:
        create_perms(module, perms_data)
    create_perms(fallback_module, default_perms)
```

**Почему:** сервисный класс изолирован и тестируется отдельно от HTTP-слоя; приватный метод связан с конкретным ViewSet/Serializer и тащит контекст HTTP-запроса в бизнес-логику. См. [[service-layer]].

---

## URL-структура

**Когда применять:** при регистрации новых маршрутов в роутере DRF.

**Правило:** подразделы — через **вложенный путь**, не дефис.

```
permissions/modules/    # правильно — подраздел
permission-modules/     # неправильно
```

**Почему:** URL отражает иерархию домена (модули прав — подраздел прав), а не плоский namespace.

---

## Docstrings

**Когда применять:** при написании функций сервисного слоя.

**Правило:** короткий однострочный docstring + `:param`/`:return:` при необходимости. **Не** Google-style (`Args:`/`Returns:`).

**Пример:**

```python
def filter_by_establishments(
    user: User,
    queryset: QuerySet,
    field_name: str = 'establishment_id',
) -> QuerySet:
    """Фильтр queryset по локациям пользователя.

    :param user: пользователь, для которого вычисляется доступ.
    :param queryset: исходный QuerySet.
    :return: отфильтрованный QuerySet.
    """
    ...
```

**Почему:** единый стиль с `base/services.py` и `user_profiles/services/services.py` — эталоны кодовой базы.

---

## Каскадное удаление справочников

**Когда применять:** при удалении справочных сущностей (модули прав, типы записей, теги).

**Правило:** каскадное удаление связанных записей — **ожидаемое поведение**. Не предлагать dry-run или soft-delete для справочников.

**Почему:** справочник — это набор констант. Если удалили модуль прав, права на него больше не нужны — администратор назначит новые. Soft-delete справочников плодит мусор в БД и усложняет UI.

---

## Связанное

- [[viewset-patterns]] — паттерны ViewSet, ловушки class-level queryset
- [[serializer-patterns]] — паттерны DRF-сериализаторов
- [[service-layer]] — сервисный слой
- [[soft-delete]] — soft-delete (важно: фильтр **не** на class-level)
