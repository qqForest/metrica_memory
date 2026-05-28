# Custom exceptions для бизнес-валидаций

**Когда применять:** при добавлении валидации, которая отражает **состояние записи в БД** или **бизнес-конфликт операции** (нельзя редактировать в этом состоянии, конфликт лицензий, неподходящая версия файла).

**Правило:** raise через `*Exception(BaseAPIException)` с явным **кодом**. `serializers.ValidationError` оставлять только для валидации полей (формат, длина, тип, обязательность).

**Пример:**

```python
# <app>/exceptions.py
from base.exceptions import BaseAPIException

class ExternalMetricException(BaseAPIException):
    """Структурированная ошибка CRUD ExternalMetric."""
    IMPORT_NOT_EDITABLE = 'IMPORT_NOT_EDITABLE'

# в сериализаторе/вью
raise ExternalMetricException(
    ExternalMetricException.IMPORT_NOT_EDITABLE,
    'Импортированную запись нельзя редактировать.',
)
```

Образцы в проекте — `loader/exceptions.py:ImportException`, `settings/exceptions.py:SystemSettingException`.

**Граница (когда `serializers.ValidationError` уместен):**
- `validate_<field>` на формат/длину/тип.
- Динамические проверки данных, не привязанные к состоянию модели («подмножество не входит в надмножество», «список пустой»).
- Несколько ошибок по разным полям, которые DRF собирает в `{field: [errors]}`.

**Почему:** кастомные исключения дают фронту устойчивый **код** (`{detail, code}`), текст можно менять без рассинхрона с UI. На MCA-320 Руслан написал «давай лучше кастомную ошибку 400, а не `serializers.ValidationError`» про `raise serializers.ValidationError({'non_field_errors': '...'})` в патч-валидаторе `ExternalMetric`. Дополняет [[shared/custom-exceptions]].
