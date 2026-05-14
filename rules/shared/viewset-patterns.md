# DRF ViewSet — паттерны

**Когда применять:** при создании или правке любого `ViewSet` в Metrica Backend.

В кодовой базе четыре типа ViewSet. Выбор зависит от сложности сущности.

---

## Тип 1: Полный CRUD со scope (`ScopedViewSetMixin` + `ModelViewSet`)

Основной и наиболее частый тип. Используется для сущностей с разными сериализаторами на чтение/запись и scope-доступом.

**Правило:**
- Базовый `queryset` — **голый** `Model.objects.all()`.
- `select_related`/`prefetch_related` — только в `list()` и `retrieve()`, не на class-level.
- `get_serializer_class()` обязателен — **никогда** не мутировать `self.serializer_class` напрямую в методах.
- Экземпляр сериализатора — через `self.get_serializer(...)`, не `SerializerClass(data=...)`.
- Input/Output паттерн: входящий сериализатор для валидации, `IdSerializer` для ответа.
- Каждый метод декорируется `@extend_schema(request=..., responses={...})` — Swagger не выводит схему автоматически.
- `transaction.atomic()` — в **сервисе**, не во view и не в сериализаторе.
- Scope-методы (`get_own_queryset`, `get_team_queryset`) — минималистичные, Q-объекты inline.

**Пример:**

```python
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.all()
    serializer_class = WorkShiftDetailSerializer

    def get_serializer_class(self):
        if self.action == 'list':
            return WorkShiftListSerializer
        if self.action in ['create', 'update', 'partial_update']:
            return WorkShiftCreateUpdateSerializer
        return WorkShiftDetailSerializer

    def list(self, request, *args, **kwargs):
        self.queryset = self.queryset.select_related('position', 'schedule')
        return super().list(request, *args, **kwargs)

    @extend_schema(request=WorkShiftCreateUpdateSerializer, responses={201: WorkShiftIdSerializer})
    def create(self, request, *args, **kwargs):
        input_serializer = self.get_serializer(data=request.data)
        input_serializer.is_valid(raise_exception=True)
        instance = WorkShiftCreateService(request).create(input_serializer.validated_data)
        return Response(WorkShiftIdSerializer(instance).data, status=201)

    def get_own_queryset(self, qs):
        return qs.filter(Q(profile=self.request.user) | Q(profile__isnull=True))

    def get_team_queryset(self, qs):
        user = self.request.user
        q = Q(profile=user)
        if user.organization:
            q |= Q(schedule__establishment__organization_id=user.organization.id)
        return qs.filter(q)
```

**Почему мутация `self.serializer_class` запрещена:** устойчивый антипаттерн (регрессирует в новых ViewSet). `return` на последней строке `get_serializer_class()` перетрёт ранее присвоенное значение. Единственный канонический способ — переопределение метода.

---

## Тип 2: Справочник (`GenericViewSet` + миксины)

Для справочных сущностей с ограниченным набором действий (модули прав, типы записей).

**Правило:**
- **Миксины** вместо `ModelViewSet` — подключаем только необходимое.
- `http_method_names` — для блокировки PUT при наличии `UpdateModelMixin`. PUT подразумевает полный объект, а с nullable-полями его можно прислать с одним полем → нарушение семантики. PATCH — правильный метод.
- `IsSuperuser` (`base/permissions.py`), **не** DRF-овский `IsAdminUser` (тот проверяет `is_staff`, а не `is_superuser`).
- Пагинация **обязательна** даже для справочников: `LimitOffsetPagination`, `limit=10`. Фронт сам пришлёт `limit=200`, если нужно всё.

**Пример:**

```python
@extend_schema(tags=['Роли и права доступа'])
class PermissionModuleViewSet(GenericViewSet, ListModelMixin, RetrieveModelMixin, UpdateModelMixin):
    """Справочник модулей прав доступа."""
    queryset = PermissionModule.objects.all()
    permission_classes = [IsSuperuser]
    serializer_class = PermissionModuleSerializer
    http_method_names = ['get', 'patch', 'head', 'options']
```

**Почему:** `ModelViewSet` тянет destroy/create — лишняя поверхность. Миксины декларативно показывают, что справочник можно только листать и обновлять.

---

## Тип 3: Кастомные actions (`@action`)

Для сущностей со сложным workflow (заявки, тикеты): CRUD ограничен миксинами, бизнес-действия — через `@action`.

**Пример:**

```python
@extend_schema(tags=['Заявки на смену'])
class ShiftClaimViewSet(
    ScopedViewSetMixin, GenericViewSet,
    mixins.CreateModelMixin, mixins.RetrieveModelMixin, mixins.ListModelMixin,
):
    queryset = ShiftClaim.objects.all()
    serializer_class = ShiftClaimSerializer
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
    filterset_class = ShiftClaimFilter

    @action(detail=True, methods=['post'])
    def approve(self, request, pk=None):
        ...

    @action(detail=True, methods=['patch'])
    def decline(self, request, pk=None):
        ...
```

---

## Тип 4: Scope-логика в сервисе

Когда `get_team_queryset` содержит нетривиальную фильтрацию — выносить в `*Service` класс (legacy: `*Session`).

```python
def get_team_queryset(self, qs):
    return UserService().get_team_queryset(qs, self.request.user, self.request.method)
```

ViewSet остаётся тонким, сложная фильтрация тестируется отдельно от HTTP. См. [[service-layer]].

---

## Orthogonal-фильтры через `get_queryset` override

**Когда применять:** фильтр, который применяется **ко всем scope-ам одинаково** (soft-delete, `is_archived`, `is_active` и т.п.) — поверх scope-разделения.

**Правило:** такой фильтр ставится в переопределённом `get_queryset(self)` ViewSet'а, **после** `super().get_queryset()`. Не дублируется по `get_team/own_queryset`, не ставится в `get_all_queryset` (последний вызывается и для суперюзера — фильтр уйдёт и ему). Guard `if not user.is_superuser` — если по бизнес-семантике суперюзер должен видеть скрытое.

**Пример (soft-delete):**

```python
class ExternalMetricViewSet(ScopedViewSetMixin, GenericViewSet, ...):
    queryset = ExternalMetric.objects.all()

    def get_all_queryset(self, qs):
        # orthogonal-фильтр для ВСЕХ (включая суперюзера), например по организации
        user = self.request.user
        if user.organization_id:
            qs = qs.filter(establishment__organization_id=user.organization_id)
        return qs

    def get_team_queryset(self, qs):
        return filter_by_outlet(self.request.user, qs, field_name='establishment_id')

    def get_own_queryset(self, qs):
        return self.get_team_queryset(qs)

    def get_queryset(self):
        # orthogonal-фильтр для всех, КРОМЕ суперюзера
        qs = super().get_queryset()
        if not self.request.user.is_superuser:
            qs = qs.filter(deleted_at__isnull=True)
        return qs
```

**Семантическое разделение:**

| Слой | Что туда кладём | Применяется к |
|---|---|---|
| `get_all_queryset` | Общий фильтр (org-видимость и т.п.) | Всем юзерам, включая суперюзера |
| `get_team_queryset` / `get_own_queryset` | Только scope-логика (по локациям, по юзеру) | Только конкретному scope |
| `get_queryset` override | Orthogonal-фильтры с возможным guard на суперюзера (soft-delete) | После scope-разделения, перед action |

Источник: ревью MCA-320 (yolarus). См. [[soft-delete]].

---

## Переключение permissions по action

**Когда применять:** ViewSet требует разных permission-классов для разных action (например, суперюзер для записи, любой аутентифицированный — для чтения).

**Правило:** переопределить `get_permissions()` вместо составного `permission_classes = [A, B]`.

**Пример:**

```python
def get_permissions(self):
    if self.action in ['create', 'update', 'partial_update', 'destroy']:
        return [IsSuperuser()]
    return [IsAuthenticated()]
```

**Почему:** список `permission_classes = [A, B]` работает по AND — **все** классы должны вернуть True. Связка `[IsSuperuser, IsAuthenticated]` для «или суперюзер, или read-only» не работает: `IsSuperuser` заблокирует обычного пользователя на чтение. Кастомный `IsSuperuserOrReadOnly` плодит однострочные классы — `get_permissions()` чище.

---

## Явное переопределение методов

**Правило:** явно прописывать `create()`, `update()`, `destroy()` вместо декораторов уровня класса (`@extend_schema_view`).

**Почему:** со временем на каждый метод наслаиваются валидации, доп. логика, разная `@extend_schema`. Декораторы уровня класса не дают это разделить.

---

## Ловушки

### Class-level `.filter()` ломает scope-bypass

```python
# Плохо — суперюзер не увидит soft-deleted записи (получит 404)
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = WorkShift.objects.filter(deleted_at__isnull=True)
```

`get_object()` использует class-level queryset. `ScopedViewSetMixin` применяет bypass **поверх** уже отфильтрованного — но не может «отменить» фильтр класса. Фильтрация soft-delete — через `get_queryset` override с guard `if not user.is_superuser`. См. [[soft-delete]].

### `ScopedViewSetMixin` bypass требует `get_all_queryset`

Bypass суперюзера срабатывает только если ViewSet определяет `get_all_queryset`. Без него суперюзер проходит через `get_team_queryset`.

```python
# Bypass СРАБОТАЕТ
class WorkShiftViewSet(ScopedViewSetMixin, ModelViewSet):
    def get_all_queryset(self, qs):
        return WorkShiftGetQuerysetService().build(qs, self.request.user)

# Bypass НЕ сработает — суперюзер идёт через team
class TimeClockViewSet(ScopedViewSetMixin, ModelViewSet):
    queryset = TimeClock.objects.all()
    # нет get_all_queryset
```

Отсутствие `get_all_queryset` — это **архитектурное решение**, не забытый метод. Для `TimeClock` это намеренно — удалённые записи не видит никто.

---

## Связанное

- [[backend-style]] — общие правила (импорты, queryset, ORM-антипаттерны)
- [[serializer-patterns]] — сериализаторы, используемые во ViewSet
- [[service-layer]] — куда выносить логику из ViewSet
- [[soft-delete]] — почему фильтр не на class-level
