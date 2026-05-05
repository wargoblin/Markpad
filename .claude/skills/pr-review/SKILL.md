---
description: "Multi-bot review в форке wargoblin/Markpad: Codex + Copilot + Claude, обработать комментарии в цикле, закрыть PR"
user-invocable: true
disable-model-invocation: true
---

# PR Review — Multi-bot review в форке (Markpad)

Создаёт draft PR в форке, вызывает **всех трёх ревьюеров** (Codex, Copilot, Claude Code Review), обрабатывает комментарии в цикле, закрывает PR.

**Вызов:** `/pr-review`

**Prerequisite:** ветка должна быть запушена в форк (через `/pr-prepare` или вручную).

## Почему все три

Опыт PR `feat/auto-save` показал: **Codex один — поверхностный**. Он дал 2 комментария в форке, после чего на upstream PR Copilot нашёл ещё 12 — Windows-specific bugs, state machine edges, TOCTOU. Каждый ревьюер имеет свои blind spots:

- **Codex** — Rust/Cargo деталями, Svelte 5 runes, language-level issues
- **Copilot** — state machines, race conditions, cross-platform behavior, semantic gaps между UI labels и кодом
- **Claude Code Review** — архитектура, naming, читаемость, "стоит ли вообще делать так"

Если в форке настроены не все три — используй те, что есть. Минимум один обычно работает.

## Шаги

### 1. Создай review PR в форке

```bash
gh pr create \
  --repo wargoblin/Markpad \
  --head wargoblin:<рабочая-ветка> \
  --base master \
  --title "[Review] <заголовок>" \
  --body "Internal review PR before submitting upstream." \
  --draft
```

Запомни номер этого PR.

### 2. Вызови ВСЕХ ТРЁХ ревьюеров параллельно

```bash
# 1. Codex — через @mention в комментарии
gh pr comment <review-pr-номер> --repo wargoblin/Markpad --body "@codex /review focus: state machines, race conditions, cross-platform compat, error paths"

# 2. Copilot — формальный reviewer (требует Copilot App в форке)
gh pr edit <review-pr-номер> --repo wargoblin/Markpad --add-reviewer copilot-pull-request-reviewer 2>&1 || true

# 3. Claude Code Review — через @mention (требует Claude Code App в форке)
gh pr comment <review-pr-номер> --repo wargoblin/Markpad --body "@claude /review focus on architecture, edge cases, and semantic gaps between UI/labels and code behavior" 2>&1 || true
```

**Если какой-то ревьюер не настроен** — продолжай без ошибки. В отчёте отметь "not configured".

**Фокусирующая подсказка** для Codex обязательна — без фокуса он часто даёт `Didn't find any major issues` даже когда issues есть. Конкретный фокус (`state machines, race conditions, cross-platform`) направляет его на нетривиальные edges.

### 3. Ожидание и проверка

**Polling (не sleep):** опрашивай каждые 60 секунд, до 15 попыток (15 минут). Разные боты отвечают в разное время — Codex обычно 3-5 мин, Copilot 5-10 мин, Claude 7-12 мин. Жди пока **все настроенные боты** не дадут вердикт ИЛИ не пройдёт таймаут.

```bash
for attempt in $(seq 1 15); do
  sleep 60
  # проверь ВСЕ каналы ВСЕХ трёх ботов ниже
done
```

**Каждый бот пишет в несколько мест — проверяй ВСЕ:**

**Codex** (`chatgpt-codex-connector[bot]`):
- Reviews: `gh api "repos/wargoblin/Markpad/pulls/<n>/reviews" --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]")'`
- Inline: `gh api "repos/wargoblin/Markpad/pulls/<n>/comments" --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]")'`
- Issue comments: `gh api "repos/wargoblin/Markpad/issues/<n>/comments" --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]")'`

**Copilot** (имя пользователя — `Copilot` в inline и `copilot-pull-request-reviewer[bot]` в reviews):
- Reviews: `gh api "repos/wargoblin/Markpad/pulls/<n>/reviews" --jq '.[] | select(.user.login == "copilot-pull-request-reviewer[bot]")'`
- Inline: `gh api "repos/wargoblin/Markpad/pulls/<n>/comments" --jq '.[] | select(.user.login == "Copilot")'`

**Claude Code Review** (`claude[bot]` или `claude-code-action[bot]` — точное имя зависит от установленного App):
- Reviews: `gh api "repos/wargoblin/Markpad/pulls/<n>/reviews" --jq '.[] | select(.user.login | contains("claude"))'`
- Inline: `gh api "repos/wargoblin/Markpad/pulls/<n>/comments" --jq '.[] | select(.user.login | contains("claude"))'`
- Issue comments: `gh api "repos/wargoblin/Markpad/issues/<n>/comments" --jq '.[] | select(.user.login | contains("claude"))'`

**Определение результата (для каждого бота отдельно):**
- Review state "COMMENTED" + 0 inline → этот бот не нашёл проблем
- Issue comment "Didn't find any major issues" + 0 inline → этот бот не нашёл проблем
- Есть inline comments → этот бот нашёл проблемы — переходи к шагу 4 после того как **все** боты ответили или истёк таймаут

**Условие выхода из polling:** все настроенные боты вернули вердикт (любой), либо прошло 15 минут.

### 4. Обработка комментариев (до 10 итераций)

**Запомни ID обработанных комментариев** (для всех трёх ботов отдельно), чтобы не обрабатывать их повторно.

Объедини комментарии от всех трёх ботов в один список. Дубликаты возможны — иногда два бота укажут на одно и то же место. Группируй их перед обработкой и пиши общий fix-commit. Inline reply делай каждому боту отдельно (через `in_reply_to`).

Для каждого нового комментария от `chatgpt-codex-connector[bot]`, `Copilot` или `claude*`:

**Валидно → исправляем:**
- Баг, уязвимость, ошибка логики
- Реальный side effect в других частях кода
- Нарушение стиля проекта (паттерны Svelte 5 runes, tabs не пробелы, single quotes — см. `AGENTS.md`)
- Отсутствие обработки ошибок на системной границе (Tauri command boundary, fs ops)

```bash
# Исправить код, затем:
git add <файлы>
git commit -m "fix: <описание>"
git push origin <рабочая-ветка>
gh pr comment <review-pr-номер> --repo wargoblin/Markpad --body "Fixed: <что изменено и почему>"
```

Сообщи пользователю: что было → почему исправили.

**Не валидно → отклоняем:**
- Субъективное мнение без технического обоснования
- Over-engineering для задачи текущего масштаба
- Противоречит архитектуре проекта (например, попытка ввести Vue-style stores в Svelte 5 runes-проект)
- Предложение уже реализовано в кодовой базе иначе — намеренно

```bash
gh pr comment <review-pr-номер> --repo wargoblin/Markpad --body "Declined: <причина>"
```

Сообщи пользователю: что было → почему отклонили.

**После каждого push** — снова polling (60с × 15 попыток) и проверяй каналы **всех трёх ботов**. Force-push не нужен; дополнительные коммиты автоматически триггерят повторный review у Copilot, и Codex/Claude можно перезапустить через `@codex /review` / `@claude /review` если они не среагировали сами.

**Условия выхода из цикла:**
- Все настроенные боты вернули вердикт без новых inline comments
- Нет новых inline comments с ID, которых нет в списке обработанных
- Достигнут лимит 10 итераций

### 5. Отчёт и закрытие

```markdown
### Итог Code Review
- **Итераций:** N
- **Codex:** M комментариев (исправлено / отклонено / not configured)
- **Copilot:** M комментариев (исправлено / отклонено / not configured)
- **Claude Code Review:** M комментариев (исправлено / отклонено / not configured)
- **Статус:** готово к отправке в upstream / требует внимания
```

```bash
gh pr close <review-pr-номер> --repo wargoblin/Markpad --delete-branch=false
```

Следующий шаг: `/pr-submit` для отправки в upstream.

## Правила

- **Не делай force-push в ветке с открытым review PR** между итерациями — обычный `git push` достаточно, потому что коммиты комулятивные. Force-push только если действительно нужно переписать историю.
- **Не закрывай review PR пока есть необработанные комментарии** — иначе теряется контекст обсуждения.
