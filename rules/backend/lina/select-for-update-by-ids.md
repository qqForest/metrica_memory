---
paths:
  - "**/*.py"
---

# select_for_update — по id, не поверх distinct

**Когда применять:** блокировка строк (`select_for_update`) на queryset, пришедшем из фильтра доступа.

**Правило:** из такого queryset брать только id и блокировать чистым запросом:

```python
allowed_ids = qs.values_list('id', flat=True)
plans = SalesPlan.objects.filter(id__in=allowed_ids).select_for_update(of=('self',))
```

Эталон — `lock_plans` в `sales_plan/services/sales_plan_services.py`.

**Почему:** фильтры доступа вроде `filter_by_organization` заканчиваются `.distinct()`, а PostgreSQL запрещает
`FOR UPDATE` с `DISTINCT` (`NotSupportedError` → 500). Ограничение действует только на внешний SELECT, distinct в
подзапросе допустим — запрос остаётся один. Тесты на SQLite этого не ловят: там `select_for_update` — no-op.
