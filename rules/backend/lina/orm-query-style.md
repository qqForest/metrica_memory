---
paths:
  - "**/*.py"
---

# Стиль сложных ORM-запросов — эталон `annotate_roster`

**Когда применять:** при написании или правке любого места, где queryset собирается из нескольких
аннотаций, подзапросов, фильтров и prefetch.

**Эталон:** `shifts/services/work_shift_services.py` → `annotate_roster`. Перед тем как писать
сложный запрос — открыть эту функцию и повторить её раскладку.

## Правила, снятые с эталона

**1. Каждая часть выражения — своя именованная переменная.** Формула читается сверху вниз, а не
разбирается из вложенной матрёшки.

```python
# Хорошо
plan_hours = duration_to_seconds(plan_duration_expr()) / Value(60 * 60)
rate = Coalesce(F('position__rate'), Value(0.0), output_field=FloatField())
total_hours = Coalesce(
    Sum(user_hours, filter=Q(period_shifts__isnull=False)), Value(0.0), output_field=FloatField(),
)

# Плохо — всё внутрь annotate одним куском
.annotate(planned_payroll=Round(Coalesce(Sum(duration_to_seconds(...) / Value(3600), ...
```

**2. Цепочка методов — в круглых скобках, точка в начале строки, отступ 4 пробела.** Источник —
первой строкой внутри скобок, отдельно от вызовов.

```python
shift_qs = (
    shift_qs
    .select_related('schedule__establishment', 'tag__color')
    .annotate(shift_hours=Round(plan_hours, 2))
    .order_by('schedule_id', Coalesce(F('custom_start_time'), F('time_slot__start_time')))
)
```

**3. Один вызов, не влезающий в строку, — аргументы обычным отступом 4, висячая запятая,
закрывающая скобка на своей строке.** Никакого выравнивания по открывающей скобке — в проекте
так не пишут.

```python
shift_qs = WorkShift.objects.filter(
    deleted_at__isnull=True, schedule__deleted_at__isnull=True, schedule__is_archived=False,
)
```

Аргументы при этом набиваются до предела строки, а не по одному на строку — см. [[compact-lines]].

**4. Условные части — последовательными присваиваниями в ту же переменную**, а не одним
разветвлённым выражением.

```python
if not user.is_superuser:
    shift_qs = shift_qs.filter(schedule__in=user_schedules)
shift_qs = RosterShiftFilter(request.query_params, queryset=shift_qs).qs
if not is_admin:
    shift_qs = shift_qs.filter(is_published=True)
```

**5. Итог — в переменную, `return` отдельной строкой.** Длинный `return (...)` не собирать.

**Почему:** сложные запросы читают чаще, чем пишут. Именованные куски позволяют понять формулу без
разбора вложенности, а единая раскладка цепочки убирает спор о скобках и переносах — они больше не
ставятся «как придётся» в каждом новом месте.

## Если выражение не собирается

Про ORM-арифметику не говорить «не работает». PostgreSQL умеет `date + time` и `interval * scalar`, Django —
арифметику с `F()` и `timedelta`. Обычно нужен явный `output_field` через `ExpressionWrapper` — так и
писать, либо проверить сгенерированный SQL на PostgreSQL. Тесты на SQLite такое не доказывают.

## Связанное

- [[compact-lines]] — не дробить влезающее в строку
- [[backend-style]] — лимит строки, импорты через `\`
