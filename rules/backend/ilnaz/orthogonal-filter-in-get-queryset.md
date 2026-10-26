# Orthogonal-фильтр — в переопределённом get_queryset

**Когда применять:** при добавлении фильтра, который применяется одинаково ко всем scope (team/own/all), но **не** должен применяться к суперюзеру.

**Правило:** ставить в переопределённом `get_queryset(self)` поверх `super().get_queryset()` с guard `if not is_superuser`. Не дублировать по `get_team_queryset`/`get_own_queryset`, не ставить в `get_all_queryset` (последний вызывается и для суперюзера).

**Решающее дерево:**
- Фильтр одинаков для всех scope, **суперюзер ему НЕ подчиняется** → `get_queryset` override с `if not is_superuser`.
- Фильтр одинаков, **включая суперюзера** (org-видимость) → [[org-scope-in-get-all-queryset]].
- Фильтр **разный** для team/own (по локациям/юзеру) → `get_team_queryset` / `get_own_queryset`.
- Фильтр **на class-level queryset** — нельзя, ломает bypass суперюзера.

**Пример (soft-delete):**

```python
class ExternalMetricViewSet(ScopedViewSetMixin, GenericViewSet, ...):
    queryset = ExternalMetric.objects.all()

    def get_team_queryset(self, qs):
        # Чистая scope-логика. Никакого soft-delete.
        return filter_by_establishments(self.request.user, qs, field_name='establishment_id')

    def get_queryset(self):
        # Orthogonal-фильтр поверх scope, с guard на суперюзера.
        qs = super().get_queryset()
        if not self.request.user.is_superuser:
            qs = qs.filter(deleted_at__isnull=True)
        return qs
```

**Почему:** на MCA-320 прошёл три итерации (`get_team` → `get_all_queryset` → `get_team` → `get_queryset`), пока не нашёл правильное место. Семантически чище: «после scope-фильтрации добавь общий orthogonal-фильтр». Связано: [[shared/soft-delete]], [[shared/viewset-patterns]].
