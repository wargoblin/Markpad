---
description: "Codex review в форке wargoblin/Markpad: создать draft PR, запустить review loop, закрыть PR"
user-invocable: true
disable-model-invocation: true
---

# PR Review — Codex review в форке (Markpad)

Создаёт draft PR в форке, вызывает @codex для review, обрабатывает комментарии в цикле, закрывает PR.

**Вызов:** `/pr-review`

**Prerequisite:** ветка должна быть запушена в форк (через `/pr-prepare` или вручную).

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

### 2. Вызови Codex

```bash
gh pr comment <review-pr-номер> --repo wargoblin/Markpad --body "@codex /review"
```

### 3. Ожидание и проверка

**Polling (не sleep):** опрашивай каждые 60 секунд, до 10 попыток. Если reviewer ответил на 1-й минуте — не жди оставшиеся. Если не ответил за 10 минут — продолжай без ошибки.

```bash
for attempt in $(seq 1 10); do
  sleep 60
  # проверь все три канала ниже
  # если есть ответ от chatgpt-codex-connector[bot] → break
done
```

**Codex использует ТРИ канала ответа — проверяй ВСЕ:**

1. **Reviews** — формальный вердикт (Codex постит основной review сюда):
```bash
gh api "repos/wargoblin/Markpad/pulls/<review-pr-номер>/reviews" \
  --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]") | {id: .id, state: .state, body: .body}'
```

2. **Inline PR comments** — конкретные замечания к коду:
```bash
gh api "repos/wargoblin/Markpad/pulls/<review-pr-номер>/comments" \
  --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]") | {id: .id, path: .path, line: .line, body: .body}'
```

3. **Issue comments** — fallback (некоторые версии Codex могут ответить сюда):
```bash
gh api "repos/wargoblin/Markpad/issues/<review-pr-номер>/comments" \
  --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]") | {id: .id, body: .body}'
```

**Определение результата:**
- Если review появился (state "COMMENTED") и inline comments пусты → **review passed**, переходи к шагу 5
- Если issue comment содержит "Didn't find any major issues" и inline comments пусты → **review passed**, переходи к шагу 5
- Если есть inline comments → **есть замечания**, переходи к шагу 4

### 4. Обработка комментариев (до 10 итераций)

**Запомни ID обработанных комментариев**, чтобы не обрабатывать их повторно.

Для каждого нового комментария от `chatgpt-codex-connector[bot]`:

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

**После каждого push** — снова polling (60с × 10 попыток) и проверяй ВСЕ ТРИ канала.

**Условия выхода из цикла:**
- Review state "COMMENTED" без inline comments
- Или issue comment содержит "Didn't find any major issues"
- Нет новых inline comments с ID, которых нет в списке обработанных
- Достигнут лимит 10 итераций

### 5. Отчёт и закрытие

```markdown
### Итог Code Review
- **Итераций:** N
- **Исправлено:** M комментариев
- **Отклонено:** K комментариев
- **Статус:** готово к отправке в upstream / требует внимания
```

```bash
gh pr close <review-pr-номер> --repo wargoblin/Markpad --delete-branch=false
```

Следующий шаг: `/pr-submit` для отправки в upstream.

## Правила

- **Не делай force-push в ветке с открытым review PR** между итерациями — обычный `git push` достаточно, потому что коммиты комулятивные. Force-push только если действительно нужно переписать историю.
- **Не закрывай review PR пока есть необработанные комментарии** — иначе теряется контекст обсуждения.
