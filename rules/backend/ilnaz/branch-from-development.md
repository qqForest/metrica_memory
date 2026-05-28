# Ветки от development

**Когда применять:** при создании новой feature/bugfix-ветки.

**Правило:** ветки фич создавать от `development`, **не** от `main`. Хотфиксы — от `main`.

**Пример:**

```bash
git checkout development && git pull
git checkout -b feature/MCA-XX-slug
```

**Почему:** прямая поправка пользователя: «никаких от main, от development!». `main` обновляется через merge с `development` при релизах — ветки от `main` расходятся с актуальным состоянием.
