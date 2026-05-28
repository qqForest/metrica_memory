# Type hints — везде, без строковых аннотаций

**Когда применять:** при написании любой новой функции или метода.

**Правило:**

1. **Все** функции и методы — type hints на аргументах **и** возвращаемом значении.
2. Аннотации — **реальные** типы, не строковые. Импортировать классы напрямую.
3. При циклических импортах — `from __future__ import annotations` или `if TYPE_CHECKING:` с реальным импортом, а **не** строка `'User'` в кавычках.

```python
# Хорошо
from user_profiles.models import User
from django.db.models import QuerySet

def filter_by_outlet(
    user: User,
    queryset: QuerySet,
    field_name: str = 'outlet_id',
    is_primary: bool = False,
) -> QuerySet:
    ...


# Хорошо — циклический импорт через TYPE_CHECKING
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from shifts.models import WorkShift

def update_shift(shift: WorkShift) -> None:
    ...


# Плохо — строковая аннотация
def filter_by_outlet(user: 'User', queryset, field_name='outlet_id'):
    ...


# Плохо — без аннотации возврата
def filter_by_outlet(user: User, queryset: QuerySet, field_name: str = 'outlet_id'):
    ...
```

**Почему:** type hints — это контракт функции в коде. Строковые `'User'` парсятся только инструментами (IDE, mypy), при чтении глазами теряют связь с импортом. `from __future__ import annotations` делает все аннотации lazy на уровне модуля — это правильное решение для циклических зависимостей, а не точечные строки.

## Связанное

- [[backend-style]] — импорты и общий стиль
