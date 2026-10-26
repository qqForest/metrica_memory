---
paths:
  - "**/*.py"
---

# Аргументы при вызове — позиционно

**Когда применять:** при вызове функции/метода нашего кода.

**Правило:** аргументы передавать позиционно, чтобы вызов помещался в одну строку (см. [[compact-lines]]).
По имени — только необязательный параметр, перед которым пропущен другой необязательный.

```python
# Хорошо
decline_claim(claim, request.user, serializer.validated_data['decline_reason'])
raise ShiftException(ShiftException.SHIFT_NOT_FOUND, detail, status.HTTP_404_NOT_FOUND)
raise ShiftException(ShiftException.ROLE_NOT_IN_SCHEDULE, detail, field_name='profile_id')  # status_code пропущен

# Плохо — kwargs растягивают вызов
decline_claim(claim=claim, user=request.user, reason=serializer.validated_data['decline_reason'])
```

Kwarg-only `*` в сигнатурах не форсить. Существующие вызовы с kwargs попутно не переписывать.

**Почему:** так пишут в проекте (development на 23.10.2026: 75% вызовов позиционные, исключения — 111 из 119).
Руслан против вызовов, растянутых именованными аргументами на несколько строк (MCA-764, 07.08.2026).
