---
paths:
  - "**/migrations/**"
---

# Django миграции — ловушки

**Когда применять:** при написании data-миграций (`RunPython`), переименовании FK, изменении NOT NULL ограничений.

---

## Исторические модели — только реальные поля БД

**Правило:** в data-миграциях обращаться **только к реальным полям БД** модели. Не к `@property`, не к кастомным методам, не к `objects` (если это кастомный менеджер).

Django data-миграции работают с **историческими моделями** — замороженным снимком полей на момент создания миграции. В них **отсутствуют**:

- `@property` декораторы
- Кастомные менеджеры (`objects = CustomManager()`)
- Кастомные методы модели (`save()`, `clean()`, `full_clean()`)
- Computed/cached fields
- Сигналы (`pre_save`, `post_save`)

**Опасность:** присвоение несуществующему property создаёт обычный Python-атрибут, который **молча игнорируется** при `save()`. Никакой ошибки. Данные просто не записываются в БД.

**Пример инцидента (потеря имён 17 курьеров):**

```python
# Миграция 0008_copy_profile_to_user.py
def forward(apps, schema_editor):
    User = apps.get_model('user_profiles', 'User')
    Profile = apps.get_model('user_profiles', 'Profile')
    for profile in Profile.objects.all():
        user = User.objects.get(pk=profile.user_id)
        user.full_name = profile.full_name   # ← full_name — это @property!
        user.save()
```

`full_name` на `User` — read-only property, вычисляемый из `first_name + patronymic + last_name`. В исторической модели property не существует — присвоение создало Python-атрибут, реальные колонки остались пустыми.

**Безопасный паттерн:**

```python
def forward(apps, schema_editor):
    User = apps.get_model('user_profiles', 'User')
    Profile = apps.get_model('user_profiles', 'Profile')
    for profile in Profile.objects.all():
        user = User.objects.get(pk=profile.user_id)
        # Работаем только с реальными полями
        user.first_name = profile.first_name
        user.patronymic = profile.patronymic
        user.last_name = profile.last_name
        user.save(update_fields=['first_name', 'patronymic', 'last_name'])
```

---

## `update_fields` — защита от молчаливого игнорирования

**Правило:** использовать `save(update_fields=[...])` в data-миграциях.

**Почему:** если указать **несуществующее** поле в `update_fields`, Django выбросит `ValueError` вместо молчаливого игнорирования. Это превращает баг с property в явную ошибку миграции.

```python
user.save(update_fields=['full_name'])
# ValueError: The following fields do not exist in this model: full_name
```

---

## Замена FK с изменением target модели

**Когда применять:** FK меняет не только имя, но и target (например, `work_shift` → `timeclock`, где первый ссылался на `WorkShift`, второй — на `TimeClock`).

**Правило:** `RenameField` **не работает** — Django видит это как `remove + add` и требует default для NOT NULL FK.

**Решение — data migration в 3 шага:**

1. Добавить **новый** FK (`timeclock_id`) как `nullable`.
2. Заполнить данные (через snapshot lookup или бизнес-логику).
3. Удалить старый FK (`work_shift_id`), пересоздать индексы.

---

## Workaround для `makemigrations` с NOT NULL FK

**Проблема:** `makemigrations` не может сгенерировать миграцию для замены одного NOT NULL FK на другой — видит как `remove + add` и требует default.

**Workaround:**

1. Временно сделать новый FK `nullable` в модели (`null=True`).
2. Запустить `makemigrations` — Django сгенерирует `AddField`.
3. **Вручную** добавить `AlterField` в ту же миграцию, возвращающий NOT NULL.
4. Вернуть `null=False` в модели.

**Важно:** если миграция уже применена на dev/prod — **нельзя** редактировать её. Нужна отдельная миграция с исправлением.

---

## Чеклист перед коммитом data-миграции

1. Внутри `RunPython` — только `apps.get_model(...)`, не прямой импорт модели.
2. Нет обращений к `@property`, методам модели или кастомным менеджерам.
3. `save(update_fields=[...])` — указан явно.
4. Edge-cases протестированы: NULL-связи, пустые поля, orphaned records.
5. Для NOT NULL FK добавления — учтён workaround выше.
6. `reverse_code` — либо `noop`, либо обратимая логика.

---

## Связанное

- [[backend-style]] — общие правила
- [[audit-logging]] — FK fix в TimeClockLog как пример замены FK с разным target
