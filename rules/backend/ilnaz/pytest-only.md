# Тесты только на pytest

**Когда применять:** при написании любого нового теста.

**Правило:** pytest fixtures, plain `assert`, классы без наследования от `TestCase`. Не использовать `django.test.TestCase`, не использовать `self.assertEqual` / `self.assertTrue` и аналоги.

**Пример:**

```python
# Хорошо
class TestWorkShiftCreate(BaseTestCase):
    async def test_create_201(self, ...):
        response = await self.client.post(...)
        assert response.status_code == 201

# Плохо
class TestWorkShiftCreate(django.test.TestCase):
    def test_create_201(self):
        response = self.client.post(...)
        self.assertEqual(response.status_code, 201)
```

**Почему:** единый стиль тестов в проекте. pytest-фикстуры удобнее `setUp`, plain `assert` даёт лучшую диагностику падений (pytest показывает значения через introspection).
