# Именованные аргументы при вызове

**Когда применять:** при вызове любой функции/метода **нашего** кода с двумя и более аргументами.

**Правило:** передавать аргументы по имени, не позиционно.

```python
# Хорошо
filter_by_outlet(
    user=self.request.user,
    queryset=qs,
    field_name='outlet_id',
    is_primary=True,
)
FileCache.pop(file_uuid=str(file_uuid), username=user.username)

# Плохо
filter_by_outlet(self.request.user, qs, 'outlet_id', True)
FileCache.pop(str(file_uuid), user.username)
```

**Исключения:**
- Встроенные типы и stdlib: `set()`, `len()`, `list.append()`, `datetime.combine()` — по позициям.
- Однопараметрические вызовы (`Model.objects.get(pk)`) — необязательно.

**Почему:** kwargs делают вызов самодокументированным и защищают от тихих регрессий при изменении сигнатуры. Поменяли порядок параметров — позиционные вызовы ломаются с непонятной ошибкой типа, kwargs продолжают работать.

**Связанное правило:** для функций со многими параметрами можно форсить kwarg-only через `*` в сигнатуре — но не обязательно.
