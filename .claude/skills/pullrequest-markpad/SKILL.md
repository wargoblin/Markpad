---
description: "Полный PR pipeline для Markpad: prepare → review → submit → notify"
user-invocable: true
disable-model-invocation: true
---

# Pull Request — полный pipeline (Markpad)

Выполняет все четыре фазы PR workflow последовательно для контрибьюта в `alecdotdev/Markpad` через форк `wargoblin/Markpad`:

1. **`/pr-prepare`** — sync, rebase, squash, self-check, push
2. **`/pr-review`** — Codex review loop в форке
3. **`/pr-submit`** — cross-fork PR в upstream + Copilot review loop
4. **`/pr-notify`** — позвать `@alecdotdev` на ревью

**Вызов:** `/pullrequest-markpad` или `/pullrequest-markpad wait`

## Режимы

| Режим | Вызов | Поведение |
|-------|-------|-----------|
| **Auto (default)** | `/pullrequest-markpad` | Создаёт upstream PR автоматически после review |
| **Wait** | `/pullrequest-markpad wait` | Спрашивает подтверждение перед отправкой в upstream |

**Arguments:** `$ARGUMENTS`
- Если `$ARGUMENTS` содержит `"wait"` → передать wait mode в фазу submit
- Иначе → auto mode

## Workflow

Выполни последовательно все шаги из четырёх скиллов:

### Фаза 1: Prepare
Следуй инструкциям из `.claude/skills/pr-prepare/SKILL.md`.
Если фаза завершилась ошибкой (конфликты, нечего коммитить, упали проверки) — **остановись**.

### Фаза 2: Review
Следуй инструкциям из `.claude/skills/pr-review/SKILL.md`.
Если review выявил проблемы, которые не удалось исправить за 10 итераций — сообщи пользователю и **остановись**.

### Фаза 3: Submit
Следуй инструкциям из `.claude/skills/pr-submit/SKILL.md`.
Передай wait mode если он был указан.

### Фаза 4: Notify
Следуй инструкциям из `.claude/skills/pr-notify/SKILL.md`.
Позови `@alecdotdev` на ревью с вежливым комментарием.

## Когда использовать отдельные фазы

| Ситуация | Что вызвать |
|----------|-------------|
| Полный цикл с нуля | `/pullrequest-markpad` |
| Ветка уже готова, нужен только review | `/pr-review` |
| Review пройден, нужно отправить в upstream | `/pr-submit` |
| Есть открытый PR с комментариями Copilot | `/pr-submit` (обнаружит существующий PR и обработает комментарии) |
| Нужно только подготовить ветку | `/pr-prepare` |
| PR в upstream готов, позвать мейнтейнера | `/pr-notify` |

## Markpad-специфичные напоминания

- **Stack:** Tauri v2 + Svelte 5 (runes) + TypeScript + Rust. AGENTS.md в корне репо — главный источник правил стиля.
- **CI checks:** `npm run check` (svelte-check) + `cargo test` в `src-tauri/`. Нет frontend-тестов.
- **Remotes:** `origin = wargoblin/Markpad`, `upstream = alecdotdev/Markpad`.
- **Default branch:** `master` (а не `main`).
- **Conventional commits:** не обязательны (см. историю апстрима), достаточно краткого описательного заголовка до 70 символов.
- **Maintainer:** `@alecdotdev`.
