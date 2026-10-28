---
paths:
  - "**/filters*.py"
  - "**/filters/**"
---

# Общая логика фильтра — в класс фильтра

**Когда применять:** один и тот же фильтр нужен нескольким `FilterSet`.

**Правило:** сделать наследника фильтра с переопределённым `filter()`, а в фильтрсетах писать одну строку
с `field_name`. Не заводить базовый общий `FilterSet` и не копировать `filter_<имя>` в каждый класс.

```python
class InScheduleFilter(filters.BooleanFilter):
    def filter(self, qs, value):
        if value is None:
            return qs
        return qs.exclude(**{self.field_name: False}) if value else qs.filter(**{self.field_name: False})


in_schedule = InScheduleFilter(field_name='profile__role__in_schedule')
```

Класс кладётся в `filters.py` приложения, где все потребители; в `base/filters.py` — только когда
потребитель появился в другом приложении. Промежуточных переменных внутри `filter()` не заводить.

**Почему:** наследования фильтрсетов в проекте нет, все идут прямо от `filters.FilterSet`, а наследники
фильтров есть (`base/filters.py`). `field_name` снимает разницу в пути (`profile__role__...` у смен,
`role__...` у roster) — один класс на оба случая.
