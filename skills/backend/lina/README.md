# skills/backend/lina/

Личные скиллы Лины. Можно подсматривать и копировать. Если что-то реально полезное — кидаем в `skills/shared/` через PR.

## Что тут есть

- [`get_desc.md`](get_desc.md) — `/get_desc`, описание PR: Что / Почему / Тесты + блоки для смежников и куда их публиковать.
- [`new_job/`](new_job/SKILL.md) — задача MCA от разбора до тестов: контекст → план → код → проверка → тесты.
- [`self-review/`](self-review/SKILL.md) — перечитать свой код по чек-листу из шести пунктов перед PR.
- [`ship_pr/`](ship_pr/SKILL.md) — сдать в ревью: коммит, пуш, PR, ссылка и перевод в Трекере, чек-лист.
- [`review-comment-triage/`](review-comment-triage/SKILL.md) — разбор замечаний ревью по одному.
- [`get_sp/`](get_sp/SKILL.md) — оценка задачи в story points по шкале команды.

## Как подключить

Single-file скиллы (`<name>.md` с frontmatter `description:`) — симлинком в `~/.claude-second/commands/`:

```bash
ln -s ~/Projects/metrica_memory/skills/backend/lina/get_desc.md ~/.claude-second/commands/get_desc.md
```

Скиллы-папки (`<name>/SKILL.md`) — в `~/.claude-second/skills/`:

```bash
ln -s ~/Projects/metrica_memory/skills/backend/lina/<skill-name> ~/.claude-second/skills/<skill-name>
```

## Как добавить новый

**Single-file** — один `<name>.md` с frontmatter в этой папке.

**Skill-папка** — `mkdir <skill-name> && $EDITOR <skill-name>/SKILL.md`. Минимум:

```markdown
---
name: skill-name
description: Одна строка — что делает.
---

# Что и как делать (на русском).
```
