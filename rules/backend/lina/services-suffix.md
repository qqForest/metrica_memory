---
paths:
  - "**/*.py"
---

# Суффикс `_services.py` в пакете services/

**Когда применять:** при создании нового файла внутри `<app>/services/`.

**Правило:** все модули внутри `<app>/services/` именуются с суффиксом `_services.py`.

```
# Хорошо
loader/services/excel_loader_services.py
loader/services/file_cache_services.py
loader/services/commit_services.py
shifts/services/timeclock_services.py
shifts/services/get_queryset_services.py

# Плохо
loader/services/external_metric_committer.py   # без суффикса
loader/services/excel_loader.py                 # без суффикса
```

**Имя описывает содержимое:** `commit_services.py` для коммит-логики, `import_services.py` для импорт-логики, `get_queryset_services.py` для сборки querysetов со scope.

**Исключение:** `services.py` (без префикса) внутри пакета `services/` — для общих утилит и реестров уровня приложения. Например `loader/services/services.py` с `LOADER_TYPE_MODEL` и `validate_loader_columns`.

**Почему:** в проекте десятки доменных сервисов. Единый суффикс делает поиск (`*_services.py`, `find services -name '*_services.py'`) одной командой — сразу видна картина доменной логики приложения.

## Связанное

- [[service-layer]] — что вообще выносить в сервисы
