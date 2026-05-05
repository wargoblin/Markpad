---
description: "Позвать @alecdotdev на ревью upstream PR в alecdotdev/Markpad"
user-invocable: true
disable-model-invocation: true
---

# PR Notify — приглашение мейнтейнера на ревью (Markpad)

Запрашивает ревью у `@alecdotdev` и оставляет вежливый комментарий к upstream PR.

**Вызов:** `/pr-notify`

**Prerequisite:** upstream PR должен существовать и пройти автоматические проверки (Copilot review обработан, если был запущен).

## Шаги

### 1. Найди upstream PR

```bash
gh pr list --repo alecdotdev/Markpad --head wargoblin:<рабочая-ветка> --state open --json number,title,url
```

Если PR не найден — сообщи пользователю и **остановись**.

### 2. Pre-flight checklist

Перед вызовом мейнтейнера убедись что всё в порядке. Проверь каждый пункт и собери результат в чек-лист:

```bash
# 1. Ветка ребейзнута на свежий master?
git fetch upstream
git log --oneline upstream/master..<рабочая-ветка> | head -1  # должен быть ровно 1 коммит
git merge-base --is-ancestor upstream/master <рабочая-ветка> && echo "OK: rebased" || echo "FAIL: not rebased"

# 2. Frontend type-check чисто?
npm run check

# 3. Rust компилируется?
cd src-tauri && cargo check
cd ..

# 4. Rust тесты проходят?
cd src-tauri && cargo test
cd ..

# 5. CI прошёл (test.yml на upstream PR)?
gh pr checks <номер> --repo alecdotdev/Markpad

# 6. Нет необработанных комментариев Copilot? (если Copilot ревью был)
gh api "repos/alecdotdev/Markpad/pulls/<номер>/comments" \
  --jq '.[] | select(.user.login == "Copilot") | .id' > /tmp/copilot_ids.txt
gh api "repos/alecdotdev/Markpad/pulls/<номер>/comments" \
  --jq '.[] | select(.in_reply_to_id != null) | .in_reply_to_id' > /tmp/replied_ids.txt
comm -23 <(sort /tmp/copilot_ids.txt) <(sort /tmp/replied_ids.txt)  # должно быть пусто

# 7. PR body заполнен (Summary + Test plan)?
gh pr view <номер> --repo alecdotdev/Markpad --json body --jq '.body'
```

**Опциональные проверки** (не блокируют, но сильно повышают качество PR):
```bash
cd src-tauri && cargo clippy -- -D warnings
cd ..
```

**Собери результат в чек-лист и покажи пользователю:**

```
Pre-flight checklist:
  [x] Rebased on upstream/master (1 squashed commit)
  [x] svelte-check passed
  [x] cargo check passed
  [x] cargo test passed (N tests)
  [x] CI test.yml passed on upstream PR
  [x] No unresolved Copilot comments (or Copilot not configured)
  [x] PR body has Summary + Test plan
```

**Если любой пункт FAIL:**
- Сообщи пользователю что именно не прошло
- Предложи исправить (`/pr-prepare` для rebase/squash, ручные правки для остального)
- **Не зови мейнтейнера** пока все обязательные пункты не пройдут

### 3. Покажи чек-лист пользователю

Покажи собранный чек-лист пользователю. Если все пункты зелёные — переходи дальше без подтверждения. Останавливайся и спрашивай только если что-то FAIL.

### 4. Собери контекст для комментария

Перед написанием комментария изучи:

1. **Diff PR** — что именно меняется:
```bash
gh pr diff <номер> --repo alecdotdev/Markpad
```

2. **Контекст проекта** — прочитай `AGENTS.md` и ключевые файлы, которых касается PR, чтобы понять архитектурные решения мейнтейнера.

3. **Историю PR** — последние merged PRs в апстриме, паттерны кодовой базы:
```bash
gh pr list --repo alecdotdev/Markpad --state merged --limit 5 --json title,number
```
Найди что-то конкретное за что можно искренне похвалить (архитектура Tauri commands, чистые Svelte 5 runes, продуманные i18n-ключи, Rust-Frontend boundary и т.д.).

### 5. Запроси ревью у мейнтейнера

```bash
gh pr edit <номер> --repo alecdotdev/Markpad --add-reviewer alecdotdev
```

### 6. Оставь комментарий

Напиши комментарий к PR со следующей структурой:

```bash
gh pr comment <номер> --repo alecdotdev/Markpad --body "<комментарий>"
```

**Структура комментария:**

1. **Приветствие + похвала** (если есть за что) — короткое обращение с конкретным комплиментом про кодовую базу. Не общие фразы вроде "great project", а что-то конкретное (удачная абстракция, чистый API, продуманная структура и т.д.). Если не за что — просто приветствие.
2. **Краткое описание** — что делает PR и почему, в 1-2 предложениях (детали уже в body PR)
3. **Чек-лист проверок** — покажи что было сделано перед отправкой:

```markdown
**Pre-submit checklist:**
- [x] Rebased on latest `master`
- [x] svelte-check clean (`npm run check`)
- [x] Rust compiles cleanly (`cargo check`)
- [x] Rust tests passing (N tests)
- [x] Codex review — M comments addressed ([review PR](ссылка-на-draft-PR-в-форке))
- [x] Copilot review — K comments addressed (или: not configured in upstream)
```

**Ссылки:**
- **Codex review** — ссылка на закрытый draft PR в форке (`wargoblin/Markpad`), где проходил review. Найди его:
```bash
gh pr list --repo wargoblin/Markpad --head <рабочая-ветка> --state closed --json number,url --jq '.[0].url'
```
- **Copilot review** — комментарии Copilot видны прямо в текущем upstream PR, отдельная ссылка не нужна

Количество комментариев указывай реальное. Если комментариев не было — пиши "no issues found". Если какой-то review не запускался — пропусти этот пункт.

4. **Готовность к правкам** — что открыт к замечаниям и готов доработать

**Тон:**
- Уважительный, но не подобострастный
- Конкретный, не generic
- На английском (мейнтейнер общается на английском в GitHub)

**Пример хорошего комментария:**
> Hey! The Tauri command boundary in `src-tauri/src/lib.rs` is really clean — keeping the frontend ↔ backend contract small and explicit makes it easy to add new file ops without leaking implementation details.
>
> This PR adds auto-save with debounced writes and a self-write suppression in the file watcher.
>
> **Pre-submit checklist:**
> - [x] Rebased on latest `master`
> - [x] svelte-check clean
> - [x] Rust compiles cleanly
> - [x] Rust tests passing (N tests)
> - [x] Codex review — 3 comments addressed ([review PR](https://github.com/wargoblin/Markpad/pull/N))
> - [x] Copilot review — no issues found
>
> Happy to adjust anything based on your feedback!

**Пример плохого комментария (НЕ делай так):**
> Hello! Your project is amazing and wonderful! I made some small changes, hope you like them! Please review when you have time, thank you so much!!!

## Результат

Сообщи пользователю:
- URL PR
- Что ревью запрошен у мейнтейнера (`@alecdotdev`)
- Текст оставленного комментария

## Правила

- **Похвала должна быть искренней** — если не нашёл ничего конкретного, лучше пропустить, чем писать дежурный комплимент
- **Чек-лист — часть комментария** — мейнтейнер должен видеть что проверки пройдены
- **Комментарий на английском** — язык коммуникации в upstream
- **Один комментарий** — не спамь. Если комментарий уже оставлен ранее, не дублируй
