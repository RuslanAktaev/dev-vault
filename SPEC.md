# SPEC — dev-vault

_Обновлено: 2026-09-19_

## Назначение

`dev-vault` — Obsidian-vault в git-репозитории (`/Users/ruslanaktaev/dev/dev-vault`). Claude работает с ним через Obsidian CLI с помощью глобального скилла `obsidian`.

## Текущее состояние

### Vault
- Vault зарегистрирован в Obsidian под именем `dev-vault`.
- Заметок пока нет. Есть только пустая база `Без названия.base` с одним табличным видом «Таблица».
- Настройки: включено автоматическое обновление внутренних ссылок (`.obsidian/app.json` → `"alwaysUpdateLinks": true`). Поэтому rename/move не вызывают блокирующее окно «Обновить ссылки?».

### Git
- Ветка `main`. Конфиг vault (`.obsidian/`) и `Без названия.base` в git.
- `.gitignore` исключает `.DS_Store`, `workspace*.json`, `.obsidian/cache`, `.trash/`, а также код плагинов (`main.js`, `styles.css`). У плагинов хранятся только `manifest.json` и `data.json`.
- Установлен community-плагин `terminal`.

### Скилл `obsidian` (`~/.claude/skills/obsidian/`)
- `SKILL.md` — когда срабатывает скилл, синтаксис CLI, рецепты и правила работы.
- `scripts/obs` — обёртка над `obsidian`:
  - убирает строки мусора лаунчера из stdout (после обновления установщика до 1.13.7 их уже нет, фильтр оставлен на всякий случай);
  - возвращает exit 1, если в выводе есть `Error:`;
  - прерывает вызов через `OBS_TIMEOUT` (по умолчанию 30 с).
- `commands.txt` — полный вывод `obsidian help`.
- Проверено: create, read, append, property:set, search, tasks, task, outline, rename, move, delete, base:query, daily:path.

### Obsidian
- Установлен Obsidian 1.13.7, установщик тоже 1.13.7 (`obsidian version` → `1.13.7 (installer 1.13.7)`).
- `~/Downloads/Obsidian-1.13.7.dmg` больше не нужен, его можно удалить.

## Известные особенности CLI
- Любая команда возвращает exit 0, даже при ошибке. Поэтому всегда вызывать через `obs`.
- `obsidian help` без аргументов после обновления до 1.13.7 отрабатывает примерно за 2 с (476 строк).
- `rename` при успехе ничего не печатает. `move` требует, чтобы целевая папка существовала.
- Для `.base` нужен `path=` (`file=` не находит базу).
- Открытое модальное окно в Obsidian молча блокирует файловые операции. Проверка: `obs dev:dom selector=".modal-container" total`.
- `\n` в `content=` превращается в перевод строки.

## Открытые задачи
- Нет.
