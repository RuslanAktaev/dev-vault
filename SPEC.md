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
- Ветка `main`. Первый коммит содержит `SPEC.md` и `CLAUDE.md`.
- Пока не отслеживаются (untracked): `.obsidian/` и `Без названия.base`.

### Скилл `obsidian` (`~/.claude/skills/obsidian/`)
- `SKILL.md` — когда срабатывает скилл, синтаксис CLI, рецепты и правила работы.
- `scripts/obs` — обёртка над `obsidian`:
  - убирает две строки мусора из stdout;
  - возвращает exit 1, если в выводе есть `Error:`;
  - прерывает вызов через `OBS_TIMEOUT` (по умолчанию 30 с).
- `commands.txt` — полный вывод `obsidian help`.
- Проверено: create, read, append, property:set, search, tasks, task, outline, rename, move, delete, base:query, daily:path.

### Obsidian
- Приложение 1.13.7 работает поверх **устаревшего установщика 1.10.6** (`/Applications/Obsidian.app`).
- Скачан `~/Downloads/Obsidian-1.13.7.dmg`, но пока не установлен.

## Известные особенности CLI
- Любая команда возвращает exit 0, даже при ошибке. Поэтому всегда вызывать через `obs`.
- `obsidian help` без аргументов зависает. Использовать `help <command>` или `commands.txt`.
- `rename` при успехе ничего не печатает. `move` требует, чтобы целевая папка существовала.
- Для `.base` нужен `path=` (`file=` не находит базу).
- Открытое модальное окно в Obsidian молча блокирует файловые операции. Проверка: `obs dev:dom selector=".modal-container" total`.
- `\n` в `content=` превращается в перевод строки.

## Открытые задачи
- [ ] Установить Obsidian 1.13.7 из DMG (заменить `/Applications/Obsidian.app`).
- [ ] После обновления проверить, пропала ли строка «installer is out of date» и перестал ли зависать `obsidian help`.
- [ ] Решить, что коммитить из `.obsidian/` (например, игнорировать `workspace.json`), и закоммитить конфиг vault.
