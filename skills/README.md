# skills/

Слэш-команды, которыми мы пользуемся в Claude Code. Каждый скилл — это папка с файлом `SKILL.md` (или один `<name>.md`).

## Как подключить себе

Симлинком из этой папки в `~/.claude/skills/`:

```bash
# общекомандный
ln -s ~/Projects/metrica_memory/skills/shared/<skill-name> ~/.claude/skills/<skill-name>

# чужой персональный (если хочешь попробовать)
ln -s ~/Projects/metrica_memory/skills/backend/lina/<skill-name> ~/.claude/skills/<skill-name>
```

После симлинка скилл доступен как `/<skill-name>` в Claude Code.

## Структура

- `shared/` — общекомандные скиллы. Подключаем все.
- `backend/<имя>/` — личные скиллы, открытые для подсматривания.

## Где взять идеи

Многие хорошие скиллы уже живут у каждого из нас в `~/.claude/skills/`. Если какой-то реально пригодился — копируй сюда (см. [CONTRIBUTING.md](../CONTRIBUTING.md)).

Примеры того, что напрашивается в `shared/`:
- `/ticket` — разбор тикета техподдержки → задача в Трекере
- `/standup` — формирование стендапа
- `/server`, `/server-prod` — SSH-диагностика серверов
