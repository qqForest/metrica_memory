---
paths:
  - "**/serializers*.py"
  - "**/serializers/**"
---

# DRF Serializer — паттерны

**Когда применять:** при создании или правке любого `serializers.Serializer` / `ModelSerializer` в Metrica Backend.

**Главный принцип:** **отдельный сериализатор на каждый контекст** (list, detail, create, update, response). Один универсальный для всех action **запрещён** — тянет лишние поля, забивает ресурсы, при сложных M2M-связях может класть систему.

Наследование и переиспользование сериализаторов **не приветствуется**.

---

## Четыре типа сериализаторов

### 1. Вложенный (Nested)

Минимум полей для отображения связанной сущности внутри другого сериализатора. **Отдельный класс с docstring**, явно указывающим контекст использования.

```python
class WorkShiftPositionSerializer(serializers.ModelSerializer):
    """Вложенный сериализатор для должности в списке рабочих смен."""
    class Meta:
        model = Position
        fields = ['id', 'name']
```

**Правило:** даже если `Position` уже используется в другом nested-сериализаторе — создать новый класс. `UserPositionSerializer` ≠ `WorkShiftPositionSerializer`, хоть и одна модель.

---

### 2. List

Объединяет nested-сериализаторы. Поля называются так же, как в модели — **без суффиксов** `_on_read`.

```python
class WorkShiftListSerializer(serializers.ModelSerializer):
    claims_count = serializers.IntegerField(default=0)
    position = WorkShiftPositionSerializer()
    schedule = WorkShiftScheduleSerializer()

    class Meta:
        model = WorkShift
        fields = ['id', 'date', 'position', 'schedule', 'claims_count']
```

---

### 3. Create/Update

Отдельный класс с `validate`/`create`/`update`. Вложенные данные — через `pop` + bulk. Бизнес-логика M2M — в `*Session` классе (см. [[service-layer]]).

```python
class RoleCreateUpdateSerializer(serializers.ModelSerializer):
    role_permissions = RolePermissionCreateSerializer(many=True)

    def create(self, validated_data):
        perms_data = validated_data.pop('role_permissions')
        role = Role.objects.create(**validated_data)
        RoleSession().create_role_permissions(role, perms_data)
        return role
```

---

### 4. Response (Output)

Минимальный сериализатор для ответа на create/update — **только `id`**.

```python
class RoleIdSerializer(serializers.ModelSerializer):
    class Meta:
        model = Role
        fields = ['id']
```

**Почему только id:** фронт сам перезапросит детальную сущность через GET. Это снимает N+1, который иначе пришлось бы оптимизировать в create/update через лишние `select_related`.

---

## SerializerMethodField

**Когда применять:** только когда `source=` не покрывает логику (условия, вычисления, сборка из нескольких источников).

**Правило:**
- Если данные выражаются через `source=` на декларативном поле — использовать декларативное поле.
- `@extend_schema_field` нужен **только** для сложных типов (вложенные сериализаторы). Для примитивов (`int | None`, `str`, `bool`) drf-spectacular выводит схему из аннотации возвращаемого типа.

**Пример:**

```python
# Плохо — лишний метод
start_time = serializers.SerializerMethodField()

def get_start_time(self, obj):
    return obj.effective_start_time

# Хорошо — декларативно
start_time = serializers.TimeField(source='effective_start_time', read_only=True)

# Не нужен @extend_schema_field — тип выводится из аннотации
last_log_id = serializers.SerializerMethodField()

def get_last_log_id(self, obj) -> int | None:
    ...

# Нужен @extend_schema_field — вложенный сериализатор
some_nested = serializers.SerializerMethodField()

@extend_schema_field(SomeNestedSerializer)
def get_some_nested(self, obj):
    ...
```

**Почему:** `SerializerMethodField` без причины — это скрытый вызов Python-функции на каждый объект (N вызовов на список из N). `source=` транслируется в JOIN на уровне ORM.

---

## Исключение: один сериализатор для простых сущностей

Если сущность простая (4–5 полей) — допустим **один** сериализатор с `read_only_fields`. Swagger автоматически отрисует разные схемы для чтения и записи.

```python
class PermissionModuleSerializer(serializers.ModelSerializer):
    class Meta:
        model = PermissionModule
        fields = ['id', 'name', 'display_name', 'description']
        read_only_fields = ['id', 'name']
```

**Почему:** для справочника `PermissionModule` нет смысла плодить два класса. Граница — 4–5 полей и отсутствие nested-данных.

---

## Связанное

- [[viewset-patterns]] — Input/Output паттерн (`get_serializer_class`, `IdSerializer` в ответе)
- [[service-layer]] — куда выносить M2M-логику из `create`/`update` сериализатора
- [[backend-style]] — общие правила импортов и стиля
