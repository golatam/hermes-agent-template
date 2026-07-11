# Clavito / Hermes — карта текущей конфигурации

Дата среза: 2026-07-11  
Источник: `/data/.hermes/config.yaml` + активные cron-доставки + текущая Railway-среда.  
Секреты, токены и ключи намеренно не включены.

## Короткий вывод

Конфиг сейчас рабочий для текущей схемы: Telegram `CLAVITO CENTER` — основной интерфейс, Discord выключен, runtime footer выключен, cron доставляет отчёты в Telegram-топики. Для VPS-миграции главный принцип — переносить `/data` layout и `HERMES_HOME=/data/.hermes`, а не Railway template.

## 1. Модель и агент

- `model.provider`: `openai-codex`
  - Основной провайдер модели.
  - Для миграции: нужен перенос OAuth/auth state, не только `.env`.

- `model.default`: `gpt-5.5`
  - Основная модель по конфигу.
  - Риск: если в будущем нужен жёсткий pin новой модели, фиксировать явно тут.

- `agent.max_turns`: `60`
  - Максимум итераций/tool-call циклов на один запрос.

- `agent.reasoning_effort`: `medium`
  - Глубина рассуждений по умолчанию.

- `agent.gateway_timeout`: `1800`
  - Таймаут долгой gateway-задачи: 30 минут.

- `agent.restart_drain_timeout`: `180`
  - 3 минуты на корректное завершение активных задач при рестарте.

- `agent.api_max_retries`: `3`
  - Ретраи API-вызовов.

## 2. Контекст, компрессия, память

- `compression.enabled`: `true`
  - Автосжатие контекста включено.

- `compression.threshold`: `0.85`
  - Сжатие позднее, чем дефолтные ~50%; удобно для длинных Telegram-сессий.

- `compression.target_ratio`: `0.35`
  - После сжатия оставляется примерно 35% целевого окна.

- `compression.protect_last_n`: `20`
  - Последние 20 сообщений не сжимаются.

- `compression.protect_first_n`: `3`
  - Первые 3 не-system сообщения защищены.

- `memory.provider`: `holographic`
  - Активна Holographic memory.

- `memory.memory_enabled`: `true`
- `memory.user_profile_enabled`: `true`
- `memory.memory_char_limit`: `2200`
- `memory.user_char_limit`: `1375`
  - Эти лимиты почти заполнены; новые устойчивые факты лучше хранить в `fact_store`, а не раздувать всегда-включаемую память.

- `memory.nudge_interval`: `10`
  - Авто-напоминание/проверка памяти каждые 10 ходов.

## 3. Сессии и reset

- `session_reset.mode`: `idle`
  - Сброс только по простою, не по фиксированному часу.
  - Это правильно для Railway: фиксированный hourly reset раньше мог конфликтовать с supervisor/restart loop.

- `session_reset.idle_minutes`: `1440`
  - Сброс после 24 часов простоя.

- `session_reset.at_hour`: отсутствует
  - Хорошо: нет фиксированного daily reset.

## 4. Telegram gateway

- `platforms.telegram.enabled`: `true`
  - Telegram — основной интерфейс.

- `telegram.require_mention`: `true`
  - По умолчанию в группах нужен mention/reply/wakeword.

- `telegram.free_response_chats`: `['-1004470067691']`
  - Без mention разрешено только в `CLAVITO CENTER`.

- `telegram.allowed_chats`: `['-537675176', '-1004470067691', '-1003804792291']`
  - Эти чаты вообще разрешены. Но без mention свободный режим есть только для `-1004470067691`.

- `telegram.group_allowed_chats`: `['-1004470067691']`
  - Групповой no-mention/gated policy ужат до `CLAVITO CENTER`.

- `telegram.allowed_topics`: отсутствует
  - Значит в `CLAVITO CENTER` можно отвечать во всех топиках, не только Inbox.

- Runtime env:
  - `TELEGRAM_REQUIRE_MENTION=true`
  - `TELEGRAM_FREE_RESPONSE_CHATS=-1004470067691`
  - `TELEGRAM_ALLOWED_CHATS=-537675176,-1004470067691,-1003804792291`

### Telegram topic routing

- Inbox: `1`
- Go Latam: `24`
- Dolphin Anty: `27`
- Health: `31`
- Crypto: `34`
- Events: `39`
- Marketplaces: `42`
- Sr. Extranjero: `46`
- FGT Stone: `49`
- Paryajpam: `54`
- Personal finances: `97`

## 5. Discord / legacy platforms

- `platforms.discord.enabled`: `false`
- `plugins.disabled` includes:
  - `discord-platform`
  - `raft-platform`

Discord gateway сейчас должен быть выключен. В Railway env `DISCORD_BOT_TOKEN` всё ещё установлен, поэтому при будущих изменениях кода/конфига важно не дать env-token auto-enable снова включить Discord.

## 6. Display / output

- `display.runtime_footer.enabled`: `false`
- `display.runtime_footer.fields`: `[]`
  - Показ `% контекста` отключён.

- `display.tool_progress`: `all`
  - Внутренний прогресс tools включён на уровне конфига. В Telegram часть прогресса может фильтроваться платформой/режимом.

- `display.show_reasoning`: `false`
  - Reasoning не показывается пользователю.

- `display.streaming`: `true`
  - Стриминг включён глобально.

## 7. Terminal / tools

- `terminal.backend`: `local`
  - Команды исполняются в текущем Railway/container окружении.

- `terminal.cwd`: `/tmp`
  - Рабочая директория по умолчанию.

- `terminal.timeout`: `180`
  - 3 минуты на обычную команду.

- `delegation.max_concurrent_children`: `3`
- `delegation.max_spawn_depth`: `1`
- `delegation.max_iterations`: `50`
  - Делегирование плоское, до 3 параллельных детей.

## 8. Voice / audio

- `stt.enabled`: `true`
- `stt.provider`: `local`
  - Голосовые сообщения транскрибируются локальным провайдером, если зависимости доступны.

- `tts.provider`: `edge`
  - Озвучка через Edge TTS.

## 9. Cron jobs: текущая карта доставок

Активные:

- Health check Garmin → `telegram:-1004470067691:31`
- Hermes text backup to GitHub → `telegram:-1004470067691:1`
- Hermes R2 restic backup → `telegram:-1004470067691:1`
- TRON work USDT wallet watcher → `telegram:-1004470067691:34`
- Weekly Valencia events digest → `telegram:-1004470067691:39`
- Crypto portfolio monitor → `telegram:-1004470067691:34`
- Sr. Extranjero radar digest → `telegram:-1004470067691:46`
- Daily Telegram Inbox Review → `telegram:-1004470067691:1`

Paused / legacy:

- Discord project task capture → paused
- Hermes profile drift audit → paused
- Gateway watchdog jobs → paused

## 10. Mail routing

Registry:

- `/data/obsidian-vault/email-accounts.yaml`
- script: `/data/.hermes/scripts/mail_accounts_mcp.py`

Important schema:

```yaml
email_accounts:
  ...
  provider: gmail
```

Not:

```yaml
accounts:
  ...
  provider: google_workspace
```

Project routes:

- personal / inbox / health / events → `kirill@sirenko.ru`
- Go Latam → `kirill@golatam.digital`
- marketplaces / FGT Stone → `info@smartonlinesupplies.com`
- Dolphin → `ksirenko@dolphin-software.online`
- Paryajpam → `ciriloparyajpam@gmail.com`
- Sr. Extranjero / Flexify → disabled/null

External sending still needs preview + explicit confirmation.

## 11. VPS migration notes

Recommended target:

- Hetzner CX43
- 16 GB RAM
- keep `/data` path on VPS
- set `HERMES_HOME=/data/.hermes`
- native Hermes install is enough if only Clavito/Hermes runs there

Move these paths:

- `/data/.hermes/config.yaml`
- `/data/.hermes/.env`
- `/data/.hermes/auth.json`
- `/data/.hermes/state.db`
- `/data/.hermes/skills/`
- `/data/.hermes/cron/`
- `/data/.hermes/memories/`
- `/data/.hermes/profiles/`
- `/data/.hermes/scripts/`
- `/data/obsidian-vault/`
- backup/restic config and scripts

Do not migrate Railway template as the source of truth. On VPS use systemd/native Hermes, or Docker Compose only if we decide we want isolation/reproducibility.

## 12. Risks / cleanup candidates

- `DISCORD_BOT_TOKEN` exists in env while Discord is disabled. Safe now because plugin/platform disabled, but risky after code/config changes.
- `allowed_chats` still includes other Telegram groups; they are mention-gated, but if `require_mention` is accidentally flipped false, bot could answer there.
- `memory` always-on char limits are near full; durable project facts should go to `fact_store` and project vault, not MEMORY.md.
- Railway restart method is still supervisor-specific; after VPS migration replace with systemd service lifecycle.
- Telegram replay/backlog protection is not fixed in code; after restarts old updates may still replay if Telegram polling offset/dedup misbehaves.

## 13. Practical use of this file

Use this as:

1. pre-migration checklist;
2. rollback reference;
3. config drift audit baseline;
4. human-readable explanation of what Clavito currently depends on;
5. source for a future interactive HTML config map.
