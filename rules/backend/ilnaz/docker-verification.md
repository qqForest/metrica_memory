# Команды через Docker

**Когда применять:** при запуске тестов, миграций, любых команд `manage.py`, утилит `collect_permissions.py` и т.п.

**Правило:** запускать через `docker compose exec app_metrica python3 ...`, не локально (`uv run ...`, `python3 ...` напрямую).

**Пример:**

```bash
docker compose exec app_metrica python3 manage.py migrate
docker compose exec app_metrica python3 -m pytest -n auto -x -q
docker compose exec app_metrica python3 collect_permissions.py
```

**Почему:** проект работает в Docker-окружении. Локальный запуск использует чужой Python/чужие зависимости — поведение может не совпасть с runtime контейнера. БД, Redis и Celery тоже только в контейнерах.
