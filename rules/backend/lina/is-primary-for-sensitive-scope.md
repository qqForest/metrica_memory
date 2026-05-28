# is_primary=True для чувствительных team-scope

**Когда применять:** при написании `get_team_queryset` для ViewSet с чувствительными данными — TimeClock, его логи, любые эндпоинты записи/правки.

**Правило:** в чувствительных team-scope вьюсетах вызывать `filter_by_outlet` с `is_primary=True`:

```python
def get_team_queryset(self, qs):
    return filter_by_outlet(
        user=self.request.user,
        queryset=qs,
        field_name='timeclock__outlet_id',
        is_primary=True,
    )
```

Это ограничивает админа его **основной** локацией, а не всеми, где он числится через `OutletMembership`.

**Когда `is_primary=True` обязателен:**
- TimeClock и любые `TimeClockLog`-вьюсеты.
- Запись по таймклоку (PUT/PATCH).
- Любой эндпоинт, где видны время прихода/ухода, GPS, аудит.

**Когда можно без `is_primary`:**
- Read-only аналитика и общие списки смен — по текущему паттерну.

**Почему:** замечание @yolarus на PR #289 (MCA-202) — админ запасной локации не должен видеть историю правок таймклока этой локации. Прецедент уже есть: `TimeClockUpdateViewSet.get_team_queryset` использует `is_primary=True`. List-вьюсеты, скопированные без `is_primary`, — наследие менее строгой политики для чтения, и для чувствительных данных это утечка.

**Если сомневаешься:** спроси — эндпоинт читает чувствительные данные (время/GPS/аудит)?

## Связанное

- [[service-layer]] — общий слой сервисов
- [[viewset-patterns]] — scope-методы
