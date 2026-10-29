---
paths:
  - "**/rules/**/*.md"
---

# rules/backend/lina/

Личные правила Лины. Подключены симлинком `~/.claude-second/rules/lina`. Кодовые правила подгружаются, когда
сессия открывает файл под маску из `paths` (тесты — `**/tests/**`, вьюсеты — `**/views*`, общий стиль —
`**/*.py`); процессные (без `paths`) — в каждой сессии. Маску ставить как можно уже: всё, что под `**/*.py`,
попадает в контекст при первом же чтении кода.

## Что тут есть

### Стиль кода (`**/*.py`)

- [`code-style.md`](code-style.md) — позиционные аргументы, `return` в строку, литерал вместо одноразовой
  константы, классы выше функций, без реэкспортов, без лишних проверок, type hints, `_services.py`, связанные
  записи в одной транзакции, код под снос не трогать.
- [`compact-lines.md`](compact-lines.md) — не дробить строки до 119 символов.
- [`orm-query-style.md`](orm-query-style.md) — сложные queryset по эталону `annotate_roster`.
- [`naming-and-comments.md`](naming-and-comments.md) — полные имена, короткие docstring, без MCA-XXX в коде.
- [`query-over-python-loops.md`](query-over-python-loops.md) — один запрос вместо цикла в Python.
- [`exceptions.md`](exceptions.md) — `<app>/exceptions.py` + `BaseAPIException`; недоступный объект — 404 без id.

### По слоям

- [`viewsets.md`](viewsets.md) — тонкая view, доступ на запись в `get_queryset`, разделение по action.
- [`permission-by-screen-section.md`](permission-by-screen-section.md) — право вьюсета по разделу экрана прав.
- [`extend-schema-responses.md`](extend-schema-responses.md) — `responses()` и `codes=`, параметры в
  `<app>/parameters.py`.
- [`shared-filter-class.md`](shared-filter-class.md) — общий фильтр — наследник `Filter` с `field_name`.
- [`build-objects-in-python.md`](build-objects-in-python.md) — вложенный объект в сериализаторе, не `JSONObject`.
- [`select-for-update-by-ids.md`](select-for-update-by-ids.md) — блокировка по id, не поверх `distinct()`.

### Тесты (`**/tests/**`)

- [`tests.md`](tests.md) — раскладка по файлам, сначала существующие тесты, переиспользование фикстур, чего
  не тестировать.

### Процесс (всегда)

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
---
paths:
  - "<самая узкая маска>"
---

# Название

**Когда применять:** …

**Правило:** …

**Почему:** …
```
