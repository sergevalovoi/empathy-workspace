# Troubleshooting — Calendar/Gmail MCP

Ошибки по этапам, в порядке возникновения. Основано на реальном многочасовом troubleshooting-сеансе, не на теории.

## Почему не использовать встроенный коннектор Claude Code

Claude Code умеет сам предложить `mcp__google-calendar__authenticate` (встроенный HTTP-коннектор на `calendarmcp.googleapis.com`). **Не используй его** — на момент написания:

- OAuth физически завершается: браузер показывает «Подключено, аутентификация прошла успешно»
- Колбэк реально доходит на локальный порт (проверено через `netstat` — свежий `TIME_WAIT` после каждой попытки)
- Но токен **никогда не сохраняется** — `accessToken` в `.credentials.json` остаётся пустым, `refreshToken` вообще отсутствует
- Воспроизведено с 3 разными OAuth-клиентами (Web application, Desktop app), с обоими типами клиента, после полного перезапуска приложения, с одним и с двумя процессами Claude Code — во всех случаях одинаковый результат
- Ручной `mcp__google-calendar__complete_authentication` с полным callback URL отвечает `"No OAuth flow is in progress"` — то есть встроенный листенер УЖЕ обработал колбэк (это его страница "Подключено"), но всё равно не записал токен

**Вывод:** это баг persistence-слоя в самом Claude Code, не в GCP-настройках, не в типе OAuth-клиента, не в сети. Не тратить время на ретраи — сразу использовать self-hosted путь из основного SKILL.md.

## GCP Console

### "The caller does not have permission" при первых попытках (устаревшая находка, не для self-hosted пути)
Если это всплывает при работе через встроенный коннектор — не относится к self-hosted пакетам из этого скилла, они делают собственный OAuth и не зависят от того же кода.

### Consent screen не пропускает дальше формы
Email не добавлен в **Test users** (OAuth consent screen → Audience). Добавление может распространяться несколько минут.

### Забыли Publish App
Симптом проявится не сразу, а через 7 дней: инструменты вдруг требуют повторной авторизации без видимой причины. Если это случилось — просто повторить `npx ... auth` для нужного сервиса, но лучше сразу зайти в GCP Console → OAuth consent screen → Publish App, чтобы не повторялось еженедельно.

## npx / установка пакета

### `claude mcp list` показывает `✘ Failed to connect` сразу после регистрации
Почти всегда — холодный старт `npx` (первая загрузка пакета с npm registry). Не признак реальной проблемы. Подождать 20-30 секунд, повторить `claude mcp list`. Если не помогло — прогреть кэш вручную:
```powershell
npx -y @cocal/google-calendar-mcp@2.6.2 --version
npx -y @gongrzhe/server-gmail-autoauth-mcp@1.1.11 --version
```

### `npm error enoent Could not read package.json` из `_npx` кэша
Повреждённый кэш конкретной версии пакета. Удалить и заново прогреть:
```powershell
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\npm-cache\_npx\<хэш-из-ошибки>"
npx -y @cocal/google-calendar-mcp@2.6.2 --version
```

### Новые тулы (`mcp__gcal__*`, `mcp__gmail__*`) не появляются в текущей сессии
`claude mcp list` показывает `✔ Connected`, но сама открытая сессия Claude Code всё равно не видит инструменты. Это не баг конкретно этого MCP — сессия кэширует список тулов на момент своего старта. **Нужен полный перезапуск приложения** (не просто новый чат/вкладка). После рестарта тулы появляются сразу.

## Авторизация (Шаг 3 основного скилла)

### Браузер не открылся автоматически
В выводе команды `auth` печатается URL — скопировать и открыть вручную.

### "redirect_uri_mismatch"
Обычно означает, что OAuth-клиент создан не как **Desktop app** (см. Шаг 1 основного скилла) — у Desktop-клиентов Google сам разрешает произвольный localhost-порт, у Web application нужно вручную регистрировать redirect URI, и это не тот путь, что описан здесь.

### "Access blocked: app has not completed Google verification"
Email не добавлен в Test users, или добрались до этого экрана до того как Test user успел распространиться (подождать 1-2 минуты, повторить).

## После установки

### `delete_email` (Gmail) падает с `Insufficient Permission`
Scope `gmail.modify` (дефолтный при авторизации через `@gongrzhe/server-gmail-autoauth-mcp`) не покрывает permanent delete — это ограничение самого scope, не баг. Решение — переместить письмо в корзину вместо удаления:
```
modify_email({ messageId, addLabelIds: ["TRASH"], removeLabelIds: ["INBOX"] })
```
(`SENT` как `removeLabelIds` тоже не сработает — это системный лейбл, не убирается через API.)

### Календарь создаёт события со сдвигом по времени
Явно указывать `timeZone` в каждом событии (например `Asia/Makassar` для Бали), не полагаться на дефолтную таймзону календаря — она может быть выставлена на что угодно (встречался случай с Europe/Minsk на аккаунте владельца из Индонезии, унаследовано из старых настроек).

### "Valid tokens found" при ручном запуске, но `claude mcp list` всё равно ругается
Прогнать `claude mcp list` ещё раз через 10-15 секунд — первый прогон мог совпасть с ещё идущей установкой npm-зависимостей.

## Если ничего не помогло

1. Проверить `node -v` — Node.js обязателен, без него `npx` не работает вообще
2. Удалить токен-файлы и передобавить с нуля (Шаг 3 основного скилла)
3. Issues upstream-инструментов:
   - https://github.com/nspady/google-calendar-mcp/issues
   - https://github.com/GongRzhe/Gmail-MCP-Server/issues
