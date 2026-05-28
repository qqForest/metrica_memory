# Не копировать защитные условия без проверки модели

**Когда применять:** при добавлении проверки `if x is None`, `if obj and obj.field`, особенно для FK/связей моделей и атрибутов DRF-сериализаторов.

**Правило:** перед `if obj.field is None` — открыть определение поля и убедиться, что `null=True` или `blank=True` стоят. Если нет — защита мёртвая. Аналогично для DRF: `self.instance and ...` в `partial_update`/`update`-сериализаторе бесполезен, потому что DRF гарантирует `self.instance` в этом flow.

**Не наследовать** guard из соседнего кода — он мог быть написан по той же ошибке.

**Пример:**

```python
# Плохо — DRF гарантирует self.instance в partial_update
def validate(self, attrs):
    if self.instance and self.instance.file_path:
        raise ...

# Хорошо
def validate(self, attrs):
    if self.instance.file_path:
        raise ...

# Плохо — Schedule.establishment = ForeignKey(..., on_delete=CASCADE) без null=True
if schedule.establishment_id is None:
    return ...

# Хорошо — на схеме не может быть None, защита удаляется
```

**Почему:** мёртвая ветка вводит читателя в заблуждение про возможные состояния модели/фреймворка. На MCA-320 Руслан повторил замечание дважды — про `schedule_tzinfo` и `self.instance and self.instance.file_path`. Связано: [[strict-tz-over-patterns]] (тот же дух — не копировать паттерны вслепую).
