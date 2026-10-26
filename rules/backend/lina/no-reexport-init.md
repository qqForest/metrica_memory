---
paths:
  - "**/*.py"
---

# Без реэкспортов в `__init__.py`

**Когда применять:** при добавлении нового класса в пакет (`models/`, `filters/`, `serializers/`).

**Правило:** импортировать напрямую из модуля, где класс живёт; в `__init__.py` пакета и `__all__`
не добавлять. Соседние реэкспорты — не образец; чистить их попутно не надо.

```python
# Хорошо
from references.filters.establishment import EstablishmentFilter
from shifts.models.schedule_target import ScheduleTarget

# Плохо
from references.filters import EstablishmentFilter
```

**Почему:** реэкспорт прячет реальное место класса и даёт два пути импорта одного имени. Руслан
планирует убрать реэкспорты целиком отдельной правкой (MCA-1361).
