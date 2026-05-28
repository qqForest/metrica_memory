# Никаких относительных импортов

**Когда применять:** при написании или правке любых импортов.

**Правило:** только абсолютные импорты (`from shifts.models import WorkShift`), не относительные (`from .models import WorkShift`).

**Пример:**

```python
# Хорошо
from shifts.models import WorkShift
from shifts.choices import TimeClockStatusChoices

# Плохо
from .models import WorkShift
from .choices import TimeClockStatusChoices
```

**Почему:** проектный стандарт; Руслан повторял в ревью. Дублирует [[shared/backend-style]] раздел «Импорты», личное напоминание чтобы не забывать.
