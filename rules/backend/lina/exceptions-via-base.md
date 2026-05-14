# Кастомные исключения — через BaseAPIException

**Когда применять:** при добавлении нового бизнес-исключения в любое приложение.

**Правило:**

- Все кастомные исключения приложения — в `<app>/exceptions.py`.
- Наследуются от `base.exceptions.BaseAPIException` (он наследуется от DRF `APIException`).
- **Не** создавать отдельный `errors.py` параллельно с `exceptions.py`.
- **Не** дублировать общий базовый класс в приложении.

```python
# Хорошо — shifts/exceptions.py
from base.exceptions import BaseAPIException

class TimeClockAlreadyClosedError(BaseAPIException):
    default_detail = 'Таймклок уже закрыт.'
    default_code = 'timeclock_already_closed'


# Плохо — отдельный errors.py
# shifts/errors.py
class TimeClockError(Exception):
    ...
```

**Почему:** `BaseAPIException` уже есть в `base/exceptions.py` — он даёт правильное поведение DRF (статус-код, сериализация в JSON-ответ, перехват middleware). Своя иерархия наверх «Exception» ломает обработку: ошибки выпадают как 500 вместо понятного статус-кода.

Один файл `exceptions.py` на приложение — все ошибки в одном месте, легко искать.

## Связанное

- [[service-layer]] — где выкидывать исключения
