# Org-фильтр и soft-delete для не-суперюзера — в get_all_queryset

**Когда применять:** для ViewSet с `ScopedViewSetMixin`, чья модель привязана к `Establishment` (метрики, импорты, расписания и т.п.).

**Правило:** общий уровень видимости задавать в `get_all_queryset(qs)`:

1. **Org-фильтр.** Если у юзера есть `organization`, ограничить queryset записями его организации: `qs.filter(establishment__organization_id=user.organization_id)`. У юзеров без organization — без фильтра.
2. **Soft-delete фильтр** для всех, кроме суперюзера. Суперюзер видит soft-deleted (норма проекта).

`get_team_queryset` остаётся узким — отвечает только за `filter_by_outlet(...)` по своим локациям, **не** дублирует фильтры из `get_all_queryset`.

**Пример:**

```python
class ExternalMetricViewSet(ScopedViewSetMixin, ...):
    queryset = ExternalMetric.objects.all()

    def get_all_queryset(self, qs):
        user = self.request.user
        if user.organization_id:
            qs = qs.filter(establishment__organization_id=user.organization_id)
        if not user.is_superuser:
            qs = qs.filter(deleted_at__isnull=True)
        return qs

    def get_team_queryset(self, qs):
        # БЕЗ повторного deleted_at__isnull=True — он уже в get_all_queryset
        return filter_by_outlet(self.request.user, qs, field_name='establishment_id')

    def get_own_queryset(self, qs):
        return self.get_team_queryset(qs)
```

**Почему:** `get_all_queryset` в `ScopedViewSetMixin` вызывается и для суперюзера, и для обычного юзера **перед** scope-разделением (`project/permissions.py`). Идеальное место для общего фильтра. Зафиксировано в MCA-320. Связано: [[orthogonal-filter-in-get-queryset]] (разница между фильтрами, которые применяются к суперюзеру и нет), [[shared/viewset-patterns]].
