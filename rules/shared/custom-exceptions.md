---
paths:
  - "**/*.py"
---

# Кастомные ошибки vs `serializers.ValidationError`

**Когда применять:** при выборе, как сообщать о бизнес-ошибке 400 из ViewSet или сериализатора Metrica Backend.

В Metrica есть базовый класс `base.exceptions.BaseAPIException` со структурой `{detail, code}`. Каждое приложение объявляет свои `*Exception(BaseAPIException)` с константами-кодами (`loader/exceptions.py:ImportException(INVALID_FILE=...)`, `settings/exceptions.py:SystemSettingException(NOT_SYSTEM_SETTING=...)`).

---

## Главное правило

| Тип ошибки | Что использовать | Пример |
|---|---|---|
| Валидация **поля** (тип, длина, формат, обязательность) | `serializers.ValidationError({'field': '...'})` или встроенные DRF-валидаторы | «`email` неверного формата», «`name` обязательно» |
| Бизнес-валидация **состояния записи** или конфликт состояний | `*Exception(BaseAPIException)` с явным кодом | «Импортированную запись нельзя редактировать», «Конфликт лицензий», «Файл не той версии» |

Бизнес-ошибки идут с **кодом** для фронта, чтобы тот мог различить причины 400 без парсинга текста.

---

## Структура кастомного класса

```python
# loader/exceptions.py
from base.exceptions import BaseAPIException


class ExternalMetricException(BaseAPIException):
    """Структурированная ошибка CRUD ExternalMetric."""
    IMPORT_NOT_EDITABLE = 'IMPORT_NOT_EDITABLE'
```

- Один класс на домен/сущность.
- Коды — `UPPER_SNAKE_CASE` строки, **публичные константы класса**.
- Сообщение передаётся в `__init__` второй позицией; статус-код опциональный третий аргумент (`status.HTTP_400_BAD_REQUEST` по умолчанию).

**Использование:**

```python
# loader/serializers.py — патч-валидатор
def validate(self, attrs: dict) -> dict:
    if self.instance.file_path:
        raise ExternalMetricException(
            ExternalMetricException.IMPORT_NOT_EDITABLE,
            'Импортированную запись нельзя редактировать.',
        )
    return attrs
```

JSON-ответ:

```json
{"detail": "Импортированную запись нельзя редактировать.", "code": "IMPORT_NOT_EDITABLE"}
```

---

## Антипаттерн: `serializers.ValidationError` для бизнес-состояний

```python
# Плохо — для фронта нет кода, только текст
raise serializers.ValidationError(
    {'non_field_errors': 'Импортированную запись нельзя редактировать.'},
)
```

**Почему плохо:**
- Фронт не отличит эту ошибку от ошибки валидации поля.
- Сообщение русское — фронту нельзя на него ориентироваться (i18n, рефакторинг текста).
- На ревью MCA-320 явное замечание: «давай лучше кастомную ошибку 400, а не `serializers.ValidationError`».

---

## Когда `serializers.ValidationError` уместен

- Валидация формата поля (`max_length`, `EmailField`, `RegexValidator`, кастомный `validate_<field>`).
- Динамические проверки данных, не привязанные к состоянию модели:
  - «Список не может быть пустым»
  - «Подмножество не входит в надмножество»
- Когда у тебя несколько ошибок разом по разным полям — DRF удобно собирает их в `{field: [errors]}`.

Граница: если ошибка про **уже существующее состояние записи в БД** или **конфликт бизнес-операции** — это `*Exception`. Если про **формат входных данных** — это `serializers.ValidationError`.

---

## Связанное

- `base/exceptions.py` — `BaseAPIException`
- `loader/exceptions.py`, `settings/exceptions.py` — образцы доменных исключений
- [[serializer-patterns]] — валидация полей через `validate_<field>`
- [[viewset-patterns]] — где бросать ошибки в action-методах
