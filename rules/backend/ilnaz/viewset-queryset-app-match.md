# queryset.model должен принадлежать app ViewSet

**Когда применять:** при создании ViewSet с placeholder-queryset (`.none()`) или class-level `queryset = OtherModel.objects.all()`. Особенно — для action-only ViewSet (импорт/коммит/служебные эндпоинты), где «своей» модели в строгом смысле нет.

**Правило:** модель в `queryset` (даже `.none()`) должна принадлежать тому же app, что и сам ViewSet. Если ViewSet лежит в `loader/views.py`, использовать `LoaderLog.objects.none()`, а не `Establishment.objects.none()`.

**Пример:**

```python
# Плохо — ViewSet в loader, queryset из references
# loader/views.py
class ParserLoaderViewSet(ScopedViewSetMixin, GenericViewSet):
    queryset = Establishment.objects.none()  # ← module в RBAC будет 'references'!

# Хорошо
class ParserLoaderViewSet(ScopedViewSetMixin, GenericViewSet):
    queryset = LoaderLog.objects.none()
```

**Симптом неправильной модели:** в `tests/conftest.py` RBAC-таблица содержит `('references', 'loader')` вместо `('loader', 'loader')`. После исправления queryset — пересинхронизировать conftest.

**Почему:** `collect_permissions.py` / scope-registry берут `app_label` из `queryset.model._meta`. Если модель из чужого app — permissions попадают в registry под чужим module-key, и scope-таблица перекосится. Обнаружено в PR #334 (`723ac65f`).
