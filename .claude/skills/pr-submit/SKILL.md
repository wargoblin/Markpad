---
description: "Создать cross-fork PR в alecdotdev/Markpad + Copilot review loop"
user-invocable: true
disable-model-invocation: true
---

# PR Submit — отправка в upstream (Markpad)

Создаёт cross-fork PR из `wargoblin/Markpad` в `alecdotdev/Markpad`, запрашивает Copilot review (если поддерживается) и обрабатывает комментарии.

**Вызов:** `/pr-submit` или `/pr-submit wait`

**Prerequisite:** ветка должна быть запушена в форк (через `/pr-prepare` или вручную).

## Режимы

| Режим | Вызов | Поведение |
|-------|-------|-----------|
| **Auto (default)** | `/pr-submit` | Создаёт PR сразу |
| **Wait** | `/pr-submit wait` | Спрашивает подтверждение перед созданием |

**Arguments:** `$ARGUMENTS`
- Если `$ARGUMENTS` содержит `"wait"` → показать summary и спросить пользователя
- Иначе → создать PR сразу

## Шаги

### 1. Проверь наличие открытых upstream PR для этой ветки

```bash
gh pr list --repo alecdotdev/Markpad --head wargoblin:<рабочая-ветка> --state open --json number,title,url
```

**Если PR уже существует:**
1. **Не создавай новый** — используй существующий
2. **Запроси свежий Copilot review** (шаг 3) — даже если предыдущий review уже есть, после push нужен новый
3. **Жди и обработай** комментарии (шаги 4-5) — включая старые необработанные
4. Переходи к шагу 3, пропуская шаг 2

**Как найти необработанные комментарии:** НЕ фильтруй по дате — это ненадёжно (комментарии между push'ами теряются). Вместо этого ищи комментарии Copilot, на которые нет ответа:

```bash
# Получи все комментарии Copilot
gh api "repos/alecdotdev/Markpad/pulls/<номер>/comments" \
  --jq '.[] | select(.user.login == "Copilot") | {id: .id, path: .path, line: .line, body: .body, in_reply_to_id: .in_reply_to_id}'

# Получи все ответы (от любого пользователя)
gh api "repos/alecdotdev/Markpad/pulls/<номер>/comments" \
  --jq '.[] | select(.in_reply_to_id != null) | .in_reply_to_id'
```

Комментарий Copilot считается **необработанным**, если его `id` НЕ встречается как `in_reply_to_id` ни в одном другом комментарии. Обрабатывай ВСЕ такие комментарии.

### 2. Создай cross-fork PR

**Если wait mode ON:** покажи diff summary и спроси: «Отправить PR в upstream alecdotdev/Markpad?»

```bash
gh pr create \
  --repo alecdotdev/Markpad \
  --head wargoblin:<рабочая-ветка> \
  --base master \
  --title "<краткий-заголовок до 70 символов>" \
  --body "$(cat <<'EOF'
## Summary
- <bullet 1>
- <bullet 2>

## Test plan
- [ ] <шаг проверки 1 — например, открыть .md файл в dev-сборке>
- [ ] <шаг проверки 2 — проверить конкретное поведение>

Reviewed with Codex before submission.
EOF
)"
```

**Стиль заголовка PR:** смотри последние merged PRs в апстриме для tone matching:
```bash
gh pr list --repo alecdotdev/Markpad --state merged --limit 5 --json title
```
Краткие, описательные. `[codex]`-префикс используется когда сам Codex генерил коммит. `feat:`/`fix:` — не обязательны, но допустимы.

### 3. Запроси Copilot review

```bash
gh pr edit <upstream-pr-номер> --repo alecdotdev/Markpad --add-reviewer copilot-pull-request-reviewer 2>&1 || true
```

Если не сработало — попробуй через API:

```bash
gh api repos/alecdotdev/Markpad/pulls/<upstream-pr-номер>/requested_reviewers \
  -f "reviewers[]=copilot-pull-request-reviewer" 2>&1 || true
```

**Если Copilot не настроен в upstream** — продолжай без ошибки и пропускай шаги 4-5. Сообщи пользователю что Copilot review недоступен.

### 4. Дождись Copilot review

**Polling (не sleep):** опрашивай каждые 60 секунд, до 10 попыток. Если reviewer ответил на 1-й минуте — не жди оставшиеся. Если не ответил за 10 минут — продолжай без ошибки.

```bash
for attempt in $(seq 1 10); do
  sleep 60
  # проверь оба канала (reviews + inline comments) — см. ниже
  # если есть новый review от copilot-pull-request-reviewer[bot] → break
done
```

**Copilot использует два username — проверяй ОБА:**

1. **Reviews** (вердикт):
```bash
gh api "repos/alecdotdev/Markpad/pulls/<номер>/reviews" \
  --jq '.[] | select(.user.login == "copilot-pull-request-reviewer[bot]") | {id: .id, state: .state, body: .body}'
```

2. **Inline PR comments** (замечания к коду):
```bash
gh api "repos/alecdotdev/Markpad/pulls/<номер>/comments" \
  --jq '.[] | select(.user.login == "Copilot") | {id: .id, path: .path, line: .line, body: .body}'
```

**Определение результата:**
- review state = "COMMENTED" и inline comments пусты → Copilot не нашёл проблем
- Есть inline comments от `Copilot` → есть замечания

### 5. Обработка комментариев Copilot

**Запомни ID обработанных комментариев**, чтобы не обрабатывать их повторно.

Для каждого inline comment от `Copilot`:
- **Валидно** → исправить, push, отчитаться пользователю
- **Не валидно** → объяснить пользователю почему отклонено
- **Уже исправлено** (stale comment на старый код) → пропустить, сообщить пользователю

**После каждого push** — перезапроси Copilot review (шаг 3) и polling заново (шаг 4).

## Результат

Покажи URL созданного upstream PR.

## Правила

- **Никогда не мержи** upstream PR — решение за мейнтейнером (`@alecdotdev`)
- **Не создавай дубликат PR** — если PR для ветки уже открыт, работай с ним
- **После любого push** — всегда перезапрашивай Copilot review (если поддерживается)
- **Запрос ревью != завершение задачи** — всегда жди результат и обрабатывай
- **Если в upstream Copilot не настроен** — пропусти 3-5, переходи к `/pr-notify`
