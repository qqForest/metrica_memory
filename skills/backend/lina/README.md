# skills/backend/lina/

Личные скиллы Лины. Можно подсматривать и копировать. Если что-то реально полезное — кидаем в `skills/shared/` через PR.

## Что тут есть

- [`get_desc.md`](get_desc.md) — `/get_desc`, генерация PR description в формате ЧТО / ПОЧЕМУ / Тесты + отдельный блок для тестировщика.

## Как подключить

Single-file скиллы (`<name>.md` с frontmatter `description:`) — симлинком в `~/.claude/commands/`:

```bash
ln -s ~/Projects/metrica_memory/skills/backend/lina/get_desc.md ~/.claude/commands/get_desc.md
```

Скиллы-папки (`<name>/SKILL.md`) — в `~/.claude/skills/`:

```bash
ln -s ~/Projects/metrica_memory/skills/backend/lina/<skill-name> ~/.claude/skills/<skill-name>
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
