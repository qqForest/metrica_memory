---
paths:
  - "**/tests/**"
---

# Тесты — CRUD-действия ViewSet

**Когда применять:** при написании тестов на любой ViewSet с CRUD-действиями.

В Metrica приняты **только API-тесты** через `BaseTestCase`. Юнит-тесты сервисного слоя (`test_*_service.py`) **не пишутся и удаляются** — вся бизнес-логика тестируется через API. Реальный PostgreSQL, не SQLite. Параллельный прогон: `pytest -n auto`.

---

## Структура файлов

**Правило:** **один файл = один action**. Имя файла — `test_<модель>_<action>.py`.

```
tests/
    base/
        base_test.py          # BaseTestCase — не трогать
    fixtures/
        user_profiles.py      # Юзеры, роли
        shifts.py             # Schedule, WorkShift, …
    user_profiles/
        test_permission_module_list.py
        test_permission_module_retrieve.py
        test_permission_module_update.py
```

**Правило:** папки внутри `tests/<app>/` **не создаём** — файлы плоско.

---

## Шаблон тестового класса

```python
from rest_framework import status
from tests.base.base_test import BaseTestCase


class TestCasePermissionModuleRetrieve(BaseTestCase):
    """Тест - деталка модуля прав."""
    url = '/permissions/modules/'

    async def test_retrieve_200(self, superuser, permission_test_module):
        """Суперюзер видит деталку."""
        url = f'{self.url}{permission_test_module.id}/'
        response = await self.make_get(url, superuser.username)
        assert response['id'] == permission_test_module.id
        assert response['name'] == 'test_module'

    async def test_retrieve_401(self, permission_test_module):
        url = f'{self.url}{permission_test_module.id}/'
        await self.make_get(url, status_code=status.HTTP_401_UNAUTHORIZED)

    async def test_retrieve_403(self, user_without_permissions, permission_test_module):
        url = f'{self.url}{permission_test_module.id}/'
        await self.make_get(url, user_without_permissions.username, status_code=status.HTTP_403_FORBIDDEN)

    async def test_retrieve_404(self, superuser):
        url = f'{self.url}99999/'
        await self.make_get(url, superuser.username, status_code=status.HTTP_404_NOT_FOUND)
```

**Правила класса:**
1. Наследует `BaseTestCase`. Все методы — `async`.
2. `url` — атрибут класса.
3. Для detail/patch/delete — `url` строится через f-string в **отдельной переменной строкой выше** вызова (не inline в `make_get`).
4. Минимум ассертов: на 200 — ключевые поля. На ошибки — достаточно `status_code` (он проверяется в `_make_request`).
5. Имена тестов: `test_<action>_<код>` или `test_<модель>_<action>` для 200.
6. Порядок тестов — см. отдельный раздел «Порядок тестов в файле» ниже.

---

## Порядок тестов в файле

**Правило (строгое):** тесты в каждом файле идут **только** в этом порядке:

```
200 / 201 / 204  →  400  →  401  →  403  →  404
```

- **200/201/204** — позитивный flow (успешный create/update/delete/read).
- **400** — авторизованный пользователь шлёт невалидный payload (дубль, missing field, неверное состояние записи). Это всё ещё позитивный auth-контекст, поэтому идёт **выше** 401/403.
- **401** — без авторизации.
- **403** — авторизован, но нет прав (`user_without_permissions` или scope).
- **404** — последними; внутри 404 порядок: `_soft_deleted` → `_missing`. Сначала scope-404 (запись существует, но скрыта фильтром), потом несуществующий id.

**Почему такой порядок:** ревьюер читает файл сверху вниз, ожидая «happy path → granularity растёт → доступ → отсутствие записи». На ревью MCA-320 Руслан **три раза** оставил «401, 403 выше 404» — это самая частая регрессия, фиксируем явно.

**Пример (delete с soft-delete):**

```python
class TestCaseExternalMetricDelete(BaseTestCase):
    url = '/loader/metrics/external/'

    async def test_delete_204(self, ...): ...                  # 204
    async def test_delete_204_for_import(self, ...): ...       # 204 (другой кейс)
    async def test_delete_401(self, ...): ...                  # 401
    async def test_delete_403(self, ...): ...                  # 403
    async def test_delete_404_soft_deleted(self, ...): ...     # 404 scope
    async def test_delete_404_missing(self, ...): ...          # 404 not-found
```

**Если в одном action есть несколько 200-кейсов или несколько 400-кейсов** — внутри группы порядок свободный, главное чтобы между группами он не ломался.

**Чек перед коммитом:** в каждом изменённом `test_*.py` бегло пробежать по именам методов сверху вниз — коды статусов должны монотонно расти `200 ≤ 400 ≤ 401 ≤ 403 ≤ 404`. Если в IDE отображается outline функций — это и есть проверка.

---

## Фикстуры — через фабричные функции

**Куда:** `tests/fixtures/<app>.py`.

**Правило:**
- Фабрика — **единственная** точка создания объектов. Прямой `Model.objects.create()` в фикстурах запрещён.
- `data: dict = None` для переопределения дефолтов; мержинг через `|`.
- Все `@pytest.fixture(scope='function')` — **явно**, не полагаться на дефолт pytest.
- Не плодить уникальные имена/фамилии/телефоны/email без функциональной причины. Только то, что отличает объект (username, role, position).
- `id` не оборачивать в `str()` — передавать `int`.

**Пример:**

```python
USER_DATA = {
    'username': 'username',
    'password': make_password(BaseTestCase.password),
    'first_name': 'Имя',
    'last_name': 'Фамилия',
    'is_superuser': True,
    'is_staff': True,
    'is_active': True,
}

def create_user(data: dict = None) -> User:
    """Создать фикстуру пользователя."""
    return User.objects.create(**(USER_DATA | (data or {})))


@pytest.fixture(scope='function')
def superuser():
    return create_user()


@pytest.fixture(scope='function')
def user_without_permissions(role_without_permissions):
    return create_user({
        'is_superuser': False,
        'is_staff': False,
        'custom_role': role_without_permissions,
    })
```

Для unique-конфликтов из миграций — `get_or_create`.

---

## Матрица покрытия по action

| Action | 200/201/204 | 401 | 403 | 404 |
|---|---|---|---|---|
| list | + | + | + | — |
| retrieve | + | + | + | + |
| create | + | + | + | — |
| partial_update | + | + | + | + |
| delete | + | + | + | + |

List и create не тестируют 404 — они не оперируют конкретным объектом по id.

---

## Шаблоны action

### Create (POST)

```python
async def test_create_201(self, superuser):
    data = {'name': 'test', ...}
    response = await self.make_post(self.url, superuser.username, data, status_code=status.HTTP_201_CREATED)
    assert response['id']   # IdSerializer возвращает только id

async def test_create_401(self):
    await self.make_post(self.url, None, {}, status_code=status.HTTP_401_UNAUTHORIZED)

async def test_create_403(self, user_without_permissions):
    await self.make_post(self.url, user_without_permissions.username, {}, status_code=status.HTTP_403_FORBIDDEN)
```

### Partial Update (PATCH)

```python
async def test_patch_200(self, superuser, test_object):
    url = f'{self.url}{test_object.id}/'
    data = {'description': 'Новое'}
    response = await self.make_patch(url, superuser.username, data)
    assert response['description'] == 'Новое'
```

### Delete (DELETE)

```python
async def test_delete_204(self, superuser, test_object):
    url = f'{self.url}{test_object.id}/'
    await self.make_delete(url, superuser.username)
```

---

## Ловушка: `default_scope` системных ролей

Системные роли (`owner`, `admin`, `user`) создаются через `get_or_create` в миграциях. Если роль уже существует в тестовой БД с устаревшим `default_scope='employee'`, метод `get_admin_condition()` будет возвращать `False` даже для `owner_user`.

**Признак:** тест с `owner_user` падает на пустой `results`, хотя данные созданы.

**Решение:** для тестов «данные доступны полностью» использовать `superuser` (`is_superuser=True`) — он делает bypass всех scope-проверок и не зависит от `default_scope` роли. `owner_user` оставлять только когда тест **специально** проверяет scope owner-а.

---

## Чеклист перед коммитом

1. `flake8 tests/` — чисто.
2. `pytest tests/` — всё зелёное.
3. Нет `str()` обёрток вокруг `id`.
4. Нет inline `params`/`data` в `make_get`/`make_patch`.
5. Нет лишних ассертов (перечисление всех `id`, когда достаточно `count`).
6. Нет уникальных имён/телефонов/email в фикстурах без функциональной причины.
7. Все `@pytest.fixture(scope='function')` — явно.
8. Все импорты наверху, в одну строку (см. [[backend-style]]).
9. Юнит-тесты сервисов (`test_*_service.py`) и старые тесты на покрытый функционал удалены.

---

## Связанное

- [[test-patterns-list-filters]] — специализация для list-эндпоинтов с фильтрами
- [[tests-sync-to-async]] — `sync_to_async` для синхронных Celery-вызовов в async-тестах
- [[viewset-patterns]] — что тестировать
- [[backend-style]] — импорты, общие правила
