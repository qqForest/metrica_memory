---
paths:
  - "**/*.py"
---

# Тесты — `sync_to_async` для синхронного кода

**Когда применять:** при тестировании Celery-функций, прямых ORM-вызовов или сервисов из async-метода `BaseTestCase`.

---

## Правило

`BaseTestCase` — **async**-класс. Все методы тестов `async`. Прямые вызовы синхронных функций (Celery-задачи, ORM, сервисы) внутри async-теста падают с `SynchronousOnlyOperation`.

Решение — обернуть синхронный вызов в `sync_to_async`:

```python
from asgiref.sync import sync_to_async

async def test_auto_clock_out(self, ...):
    await sync_to_async(auto_close_shifts)()
    response = await self.make_get(f'/timeclocks/{timeclock.id}/', superuser.username)
    assert response['status'] == 'auto_closed'
```

---

## Дополнительные правила

- **Результаты проверять через API** (`make_get` детальки), **не** прямыми ORM-вызовами после синхронной функции. ORM в async-контексте после `sync_to_async` создаёт цепочку проблем — API проверка надёжнее.
- **Warnings от `sync_to_async` + `xdist` в teardown** — не критичны, игнорируются. Если падают в CI — это не баг тестируемого кода.

---

## Когда **не** применять

- Если функция уже `async` — вызывать напрямую через `await`.
- Если тест синхронный (не наследует `BaseTestCase`) — `sync_to_async` не нужен.
- Если результат вычислений не нужен в тесте — можно просто проверить состояние через API.

---

## Связанное

- [[test-patterns-crud]] — общий паттерн CRUD-тестов (BaseTestCase, async)
- [[test-patterns-list-filters]] — list-эндпоинты
