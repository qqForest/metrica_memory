# Sync-миграции после правки старых

**Когда применять:** после любой правки уже существующего файла миграции — даже косметической (удаление `RunPython`/`RunSQL`, переименование переменных, исправление опечатки `OutletMembership` → `outletmembership` в `model_name`).

**Правило:**

1. Прогнать `python manage.py makemigrations --check`.
2. Если генерируется автомиграция вида `0044_alter_..._and_more.py` — это сигнал, что правка изменила state. **Эту sync-миграцию коммитить вместе** с исходной правкой, не оставлять «на потом».
3. Не оставлять расхождение `state ↔ models` — оно вылезет на следующем PR/коммите.

**Пример:** в PR #334 после удаления `RunPython`/`RunSQL` из 7 миграций Django сгенерил **5 автомиграций** (`entities/0010`, `notifications/0002`, `references/0044`, `shifts/0059`, `user_profiles/0026`). Все они должны были попасть в тот же PR.

**В CI** должна быть проверка `python manage.py makemigrations --check --dry-run` — копеечно ловит «забыли сделать makemigrations».

**Почему:** state модели и state цепочки миграций синхронизируются Django при `makemigrations`. Ручная правка миграции рассинхронизирует их — на свежей БД получишь не тот state. Связано: [[no-handwritten-migrations]], [[squash-migrations-in-feature]], [[django-migration-reverse-historical]].
