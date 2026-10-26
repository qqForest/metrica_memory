---
paths:
  - "**/*.py"
---

# rules/backend/lina/

Личные правила Лины. Подключены симлинком `~/.claude-second/rules/lina`. Кодовые (с `paths: ["**/*.py"]`)
подгружаются, когда сессия открывает `.py`; процессные — в каждой сессии.

## Что тут есть

### Стиль кода

- [`compact-lines.md`](compact-lines.md) — не дробить строки до 119 символов.
- [`orm-query-style.md`](orm-query-style.md) — сложные queryset по эталону `annotate_roster`: части в переменные, цепочка в скобках.
- [`kwargs-at-call.md`](kwargs-at-call.md) — аргументы позиционно, по имени только после пропущенного необязательного.
- [`naming-and-comments.md`](naming-and-comments.md) — полные имена, короткие docstring, **никаких** MCA-XXX в коде.
- [`typing-everywhere.md`](typing-everywhere.md) — type hints везде, без строковых аннотаций.
- [`no-redundant-checks.md`](no-redundant-checks.md) — повторы условий сводить, ветки под невозможное не писать.
- [`literal-over-single-use-constant.md`](literal-over-single-use-constant.md) — одноразовое значение литералом, без косвенности ради длины строки.
- [`query-over-python-loops.md`](query-over-python-loops.md) — один запрос вместо цикла в Python, разница множеств через `exclude`.
- [`return-one-line.md`](return-one-line.md) — `return` одной строкой, длинное — в переменные выше.
- [`build-objects-in-python.md`](build-objects-in-python.md) — вложенный объект в сериализаторе, `JSONObject` только ради фильтра/сортировки.

### Архитектурные конвенции

- [`exceptions-via-base.md`](exceptions-via-base.md) — `<app>/exceptions.py` + `BaseAPIException`, без `errors.py`.
- [`services-suffix.md`](services-suffix.md) — файлы в `<app>/services/` с суффиксом `_services.py`.
- [`split-viewsets-by-action.md`](split-viewsets-by-action.md) — при возможности разделять ViewSet'ы по action/ресурсу.
- [`classes-before-functions.md`](classes-before-functions.md) — классы выше функций, «спустить под класс» = переставить.
- [`no-reexport-init.md`](no-reexport-init.md) — импорт напрямую из модуля, без реэкспортов в `__init__.py`.
- [`atomic-related-writes.md`](atomic-related-writes.md) — связанные записи в БД в одной `transaction.atomic()`.
- [`select-for-update-by-ids.md`](select-for-update-by-ids.md) — блокировка по id, не поверх queryset с `distinct()`.
- [`extend-schema-responses.md`](extend-schema-responses.md) — `responses()` и `codes=`, параметры в `<app>/parameters.py`.
- [`thin-views.md`](thin-views.md) — view оркестрирует, query-параметры через сериализатор.
- [`write-access-in-queryset.md`](write-access-in-queryset.md) — запрет на запись в `get_queryset`, не в `validate`; 404.
- [`access-errors-404.md`](access-errors-404.md) — недоступный объект: 404 `<СУЩНОСТЬ>_NOT_FOUND`, без id в `detail`, с `field_name`.
- [`permission-by-screen-section.md`](permission-by-screen-section.md) — право вьюсета по разделу экрана прав в макете.
- [`shared-filter-class.md`](shared-filter-class.md) — общий фильтр — наследник `Filter` с `field_name`.
- [`leave-code-scheduled-for-removal.md`](leave-code-scheduled-for-removal.md) — код с задачей сноса не трогать.

### Тесты

- [`reuse-fixtures-before-adding.md`](reuse-fixtures-before-adding.md) — перед новой фикстурой проверить существующие, не плодить копии.
- [`what-not-to-test.md`](what-not-to-test.md) — эффект сценарием через API, без тестов на удалённое, без тестовых пакетов и фильтров варнингов.
- [`test-file-layout.md`](test-file-layout.md) — файл на action (от `development`) или на эндпоинт (эпик прав), параметр — по файлам ресурсов.
- [`existing-tests-first.md`](existing-tests-first.md) — сначала починить фикстуры и прогнать существующие тесты.

### Процесс

- [`pr-description-style.md`](pr-description-style.md) — выжимка для description, один экран без скролла.
- [`pr-review-comments-style.md`](pr-review-comments-style.md) — короткие inline-комменты без эмодзи и ярлыков.
- [`commit-message.md`](commit-message.md) — `MCA-XXXX: <до трёх слов по-английски>`.
- [`pending-review.md`](pending-review.md) — черновой review не публикация, ради правки не пересоздавать.
- [`reviewer-technique-literally.md`](reviewer-technique-literally.md) — названный ревьюером приём сначала довести буквально.
- [`read-other-branches-without-checkout.md`](read-other-branches-without-checkout.md) — чужую ветку читать `git show`, без checkout.
- [`port-changes-to-fresh-base.md`](port-changes-to-fresh-base.md) — перенос точечно поверх `development`, без merge-миграций.
- [`no-rerun-after-cosmetics.md`](no-rerun-after-cosmetics.md) — после косметики полный прогон не повторять.
- [`no-env-data-backfill.md`](no-env-data-backfill.md) — backfill, админку, `create_superuser` под выкатку не предлагать.
- [`tracker.md`](tracker.md) — задачи не заводить, без призывов, чек-листы эпика прав, статус из Трекера.
- [`ask-wiki-agent.md`](ask-wiki-agent.md) — вопросы по документации вики — агенту `wiki-research`, сам не ищу.
- [`propose-skill.md`](propose-skill.md) — предложить оформить повторяемый приём скиллом.

## Шаблон

```markdown
# Название

**Когда применять:** …

**Правило:** …

**Почему:** …
```
