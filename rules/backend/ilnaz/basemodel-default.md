# BaseModel по умолчанию

**Когда применять:** при дизайне любой новой Django-модели в Metrica.

**Правило:** наследоваться от `base.models.BaseModel` (даёт `created_at` + `updated_at`). Не подменять собственным `created_at` ради «минимализма», не «экономить» `updated_at` на immutable-моделях (логи, события). Исключение — только если пользователь явно об этом попросит.

**Пример:**

```python
# Хорошо
from base.models import BaseModel

class LoaderLog(BaseModel):
    ...

# Плохо
class LoaderLog(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)  # ← дубль того, что даёт BaseModel
```

**Почему:** все логи и бизнес-модели проекта (`TimeClockLog`, `WorkShiftLog`, `SystemSettings`, `LoaderLog`/`LoaderObjectLog` после MCA-317) идут через `BaseModel`. Я однажды отказался от `BaseModel` в `LoaderLog` под предлогом «лог immutable, updated_at не нужен» — получил «че за прикол, почему модель не от BaseModel». `BaseModel` — это базовая инфраструктура, не «накручивание полей сверх ТЗ» (см. [[strict-tz-over-patterns]]).
