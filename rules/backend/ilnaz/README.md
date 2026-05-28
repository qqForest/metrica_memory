# rules/backend/ilnaz/

Личные правила Ильназа. Подключаются через его `~/.claude/CLAUDE.md` / `CLAUDE.md` проекта; **не** действуют для других участников команды.

Каждый файл — одно атомарное правило в формате:

```markdown
# Название

**Когда применять:** …

**Правило:** …

**Пример:** …  (опционально)

**Почему:** …
```

## Что тут есть

### Процесс / git

- [`branch-from-development.md`](./branch-from-development.md) — ветки фич от `development`, не от `main`.
- [`no-commit-push-without-permission.md`](./no-commit-push-without-permission.md) — не коммитить и не пушить без явного разрешения, даже в auto mode.
- [`docker-verification.md`](./docker-verification.md) — команды через Docker, не локально.
- [`review-code-not-pr-meta.md`](./review-code-not-pr-meta.md) — при ревью смотреть на код и задачу; не выносить лейблы/чек-листы/оформление PR как замечания.

### Backend стиль

- [`no-relative-imports.md`](./no-relative-imports.md) — только абсолютные импорты (дубль [[shared/backend-style]]).
- [`basemodel-default.md`](./basemodel-default.md) — все модели от `base.BaseModel`, без подмены `created_at` своим.
- [`basename-single-source.md`](./basename-single-source.md) — `basename` ViewSet'а в одной точке правды; в `router.register` через `ViewSet.basename`.
- [`no-dead-guards.md`](./no-dead-guards.md) — не копировать `if x is None` без проверки `null=True` в модели или контракта DRF.
- [`strict-tz-over-patterns.md`](./strict-tz-over-patterns.md) — ТЗ важнее паттернов, без накручивания полей сверх требований.
- [`custom-exceptions-business.md`](./custom-exceptions-business.md) — `*Exception` для бизнес-валидаций, `serializers.ValidationError` — только для полей.
- [`no-existing-text-rewrites.md`](./no-existing-text-rewrites.md) — не подменять существующие русские строки косметически; расширять только когда новый текст несёт информацию, которой нигде ещё нет.
- [`no-task-id-in-code.md`](./no-task-id-in-code.md) — никаких `MCA-XX` в коде (комментарии, docstring, имена тестов); место для номера задачи — commit/PR/ветка.

### Scope-система / queryset

- [`org-scope-in-get-all-queryset.md`](./org-scope-in-get-all-queryset.md) — org-фильтр и soft-delete для не-суперюзера в `get_all_queryset`.
- [`orthogonal-filter-in-get-queryset.md`](./orthogonal-filter-in-get-queryset.md) — orthogonal-фильтр (soft-delete и т.п.) — в `get_queryset` override с guard на суперюзера.
- [`viewset-queryset-app-match.md`](./viewset-queryset-app-match.md) — `queryset.model` должен принадлежать app ViewSet, иначе RBAC регистрирует под чужим module-key.

### Миграции

- [`no-handwritten-migrations.md`](./no-handwritten-migrations.md) — только `makemigrations`, никогда не писать руками; при EOFError — stdin-проброс.
- [`never-drop-db.md`](./never-drop-db.md) — не сносить БД ради переделки миграции (ни на dev/prod, ни локально).
- [`squash-migrations-in-feature.md`](./squash-migrations-in-feature.md) — несколько миграций на одну модель склеиваются до merge.
- [`migrations-no-data.md`](./migrations-no-data.md) — миграции = архитектура, данные = из дампа; никаких `RunPython`/`RunSQL` с DML.
- [`sync-migrations-after-edit.md`](./sync-migrations-after-edit.md) — после правки старых миграций — `makemigrations --check`, коммитить sync-миграции в тот же PR.
- [`django-migration-reverse-historical.md`](./django-migration-reverse-historical.md) — Django reverse работает через historical state цепочки, не текущее состояние БД/модели.

### Тесты

- [`pytest-only.md`](./pytest-only.md) — только pytest, никогда `unittest.TestCase`.
- [`parallel-tests.md`](./parallel-tests.md) — `-n auto -x -q` всегда.
- [`no-unit-tests.md`](./no-unit-tests.md) — без юнит-тестов сервисов, только API-тесты через `BaseTestCase`.
- [`tests-coverage.md`](./tests-coverage.md) — покрывать все 3 кейса видимости для каждой роли с реальными фикстурами.
- [`test-ordering.md`](./test-ordering.md) — порядок в файле: 200/201/204 → 400 → 401 → 403 → 404 (внутри 404: `soft_deleted` → `missing`).
