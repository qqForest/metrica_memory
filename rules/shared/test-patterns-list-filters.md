# Тесты — list-эндпоинты с фильтрами

**Когда применять:** при написании тестов на GET-список со scope, фильтрами или сложной бизнес-логикой видимости.

Дополняет [[test-patterns-crud]] специализированным паттерном для list+фильтры.

---

## Структура: три файла на один list-эндпоинт

```
tests/
    conftest.py                           # Один conftest в корне. Локальных НЕ создаём
    fixtures/
        user_profiles.py
        references.py
        shifts.py
    user_profiles/
        test_user_list_access.py          # Scope-видимость по ролям
        test_user_list_filters.py         # Простые фильтры
        test_user_list_schedule_filter.py # Сложный бизнес-фильтр
```

**Почему три файла:** каждый аспект тестируется изолированно. Filter-тест от **суперюзера** (ALL scope) — фильтр не пересекается со scope-логикой; access-тест без фильтров — scope не пересекается с фильтром.

Новые fixture-файлы регистрируются в `conftest.py` → `pytest_plugins`.

---

## Фабричные функции

Каждая модель — одна фабрика в fixture-файле своего приложения. Подробнее в [[test-patterns-crud]]. Здесь — list-специфика:

```python
def create_establishment(data: dict = None) -> Establishment:
    """Создать локацию."""
    defaults = {'name': 'Тест Кафе'}
    return Establishment.objects.create(**(defaults | (data or {})))


@pytest.fixture(scope='function')
def establishment_1(organization_1):
    return create_establishment({'name': 'Кафе Центр', 'organization': organization_1})
```

---

## Формат вызовов API

**Правило:** `params` и `data` — в переменную **строкой выше**. `make_get` — в одну строку. Никаких inline `params=` и многострочных вызовов.

```python
# Хорошо
async def test_filter_role(self, superuser, all_users):
    """Фильтр по роли."""
    params = {'role': 'Администратор'}
    response = await self.make_get(self.url, superuser.username, params=params)
    assert response['count'] == 1

# Плохо — inline params
await self.make_get(self.url, superuser.username, params={'role': 'Администратор'})

# Плохо — многострочный вызов
await self.make_get(
    self.url,
    superuser.username,
    params=params,
)
```

---

## Минимальные ассерты

**Правило:** не перечислять очевидное. Если `count == 5` и всего 5 объектов в фикстуре — достаточно `assert response['count'] == 5`. `in ids` / `not in ids` — **только** для пограничных случаев, где важно показать, кто именно исключён или включён.

```python
# Хорошо — суть: waiter не попадает из-за группы
ids = {r['id'] for r in response['results']}
assert all_users['waiter'].id not in ids

# Плохо — перечислены все, кто должен быть. count покрывает то же самое
assert all_users['admin'].id in ids
assert all_users['cook'].id in ids
assert all_users['manager'].id in ids
```

---

## Что тестировать: три категории

### 1. Access (отдельный файл `*_list_access.py`)

Scope-видимость по ролям. Покрывает весь спектр от 401 до суперюзера.

| Кейс | Что проверяем |
|---|---|
| 401 | Без авторизации |
| 403 | Пользователь без прав (scope=NONE) |
| Superuser | Видит всех |
| is_staff | Обходит scope, видит всех |
| Owner (ALL) | Видит всех |
| Admin (TEAM) + кросс | Видит локацию + кросс со сменами |
| Admin (TEAM) без кросс | Кросс-юзер БЕЗ смен — не виден |
| Employee (TEAM) + кросс | То же что admin |
| Глобальный админ (без локаций) | Обходит scope |
| Без OutletMembership | Видит только себя |
| Disabled membership | Юзер с disabled не виден через TEAM |

### 2. Простые фильтры (отдельный файл `*_list_filters.py`)

Каждый фильтр — отдельный тест. Запросы от **superuser** (ALL scope) — чтобы изолировать фильтр от scope-логики. Покрывать: `full_name`, `phone`, `role`, `position` (одиночный и мульти), `organization`, `establishment` (primary, мульти), `user_status`, `search`, `ordering`.

### 3. Сложный фильтр (отдельный файл, специфичный по домену)

Для фильтров с бизнес-логикой — schedule, кросс-локация, computed-поля.

| Кейс | Что проверяем |
|---|---|
| Schedule с groups | Фильтрует по позиции, waiter не попадает |
| Schedule без groups | Все сотрудники локации, cross не попадает |
| Несуществующий schedule | 0 результатов |
| Кросс через смены | Cross_user попадает (связь через WorkShift) |
| shift_date match/no match | Cross в/вне диапазона дат |
| computed null/false/true | Computed-поле имеет смысл во всех состояниях |

---

## Импорты и docstrings

- `from rest_framework import status` — только если используются 401/403. Для 200-теста не импортировать.
- Docstring — одна строка, сразу суть: `"""Фильтр по роли."""`, `"""Админ без локаций — глобальный админ."""`. Не перегружать деталями.

---

## Чеклист перед коммитом

1. `flake8 tests/` — чисто.
2. `pytest tests/` — зелёное.
3. Нет inline `params`/`data` в `make_get`/`make_patch`.
4. Нет перечисления всех `id`, когда достаточно `count`.
5. Нет уникальных имён/телефонов/email в фикстурах без функциональной причины.
6. Все `@pytest.fixture(scope='function')` — явно.
7. Все импорты наверху, в одну строку (см. [[backend-style]]).
8. Старые тесты на покрытый функционал удалены.

---

## Связанное

- [[test-patterns-crud]] — базовый паттерн CRUD-тестов
- [[tests-sync-to-async]] — async-нюансы
- [[viewset-patterns]] — ScopedViewSetMixin, scope-система
- [[backend-style]] — импорты, стиль
