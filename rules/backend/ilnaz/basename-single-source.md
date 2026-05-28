# basename — единая точка правды

**Когда применять:** при создании или правке ViewSet с собственным `basename`.

**Правило:** `basename` объявляется один раз — атрибутом класса **и** продублирован в `__init__` (требование `collect_permissions.py`, который читает basename из инстанса). В `router.register()` строку не повторять — передавать `basename=ViewSet.basename`. Префикс в `register` — обычная строка (`'loader/metrics/external'`), `r''` не нужно без спецсимволов. URL — вложенный путь (`metrics/external/`), не дефис (`external-metrics/`).

**Пример:**

```python
# loader/views.py
class ExternalMetricViewSet(ScopedViewSetMixin, ...):
    basename = 'externalmetrics'

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.basename = 'externalmetrics'

# loader/urls.py
router.register(
    'loader/metrics/external',
    ExternalMetricViewSet,
    basename=ExternalMetricViewSet.basename,
)

# Плохо — три точки правды
router.register(r'loader/external-metrics', ExternalMetricViewSet, basename='externalmetrics')
```

**Почему:** дублирование литерала в `router.register` создаёт три точки правды (атрибут / `__init__` / register). При переименовании одна забывается, `collect_permissions.py` молча перестаёт находить ViewSet. Зафиксировано в ревью MCA-320.
