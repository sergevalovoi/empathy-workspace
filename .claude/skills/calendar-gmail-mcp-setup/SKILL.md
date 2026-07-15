---
name: calendar-gmail-mcp-setup
description: |
  Мастер подключения личного Google Calendar и Gmail к Claude Code через
  self-hosted MCP-пакеты (не встроенный коннектор Claude Code — он на момент
  написания сломан). Ведёт через весь процесс: GCP-проект, OAuth-клиент,
  credentials-файл, авторизация, регистрация MCP-сервера, проверка.

  Триггеры: "подключить календарь", "google calendar mcp", "подключить gmail",
  "почта mcp", "настроить календарь для claude"
license: MIT
compatibility: |
  Windows 11 — команды даны для PowerShell/терминала Claude Code. Node.js
  обязателен (проверка в Pre-flight). Схема проверена и работает на Windows;
  на macOS/Linux шаги те же, отличаются только пути (`~/.claude/...` вместо
  `C:\Users\...\.claude\...`).
metadata:
  version: "1.0.0"
  category: setup
  calendar-mcp-repo: https://github.com/nspady/google-calendar-mcp
  gmail-mcp-repo: https://github.com/GongRzhe/Gmail-MCP-Server
  source: составлено из реального troubleshooting-сеанса (см. references/troubleshooting.md) — встроенный коннектор Claude Code для Calendar оказался сломан на уровне сохранения OAuth-токена
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
---

# Подключение Google Calendar и Gmail к Claude Code

Даёт Claude доступ к твоему календарю (читать/создавать/удалять события, проверять занятость) и почте (читать, искать, отправлять, управлять письмами) — прямо из Claude Code, без переключения в браузер.

**Важно с самого начала:** если Claude Code предложит встроенный коннектор `google-calendar` (`calendarmcp.googleapis.com`) через `mcp__google-calendar__authenticate` — **не используй его**. На момент написания этого скилла он ломается на этапе сохранения токена: браузер показывает «Подключено», но сам токен никогда не сохраняется, и это повторяется независимо от типа OAuth-клиента, количества попыток или перезапуска приложения. Ниже — рабочий путь через self-hosted npm-пакеты, полностью в обход этого коннектора.

## Глоссарий

- **MCP (Model Context Protocol)** — способ подключения внешних инструментов к Claude Code.
- **OAuth-клиент (Desktop app)** — тип credentials в Google Cloud для приложений без сервера, работает через PKCE (без client_secret на стороне пользователя). Обязательно выбирать именно этот тип, не «Web application».
- **gcp-oauth.keys.json** — файл с client_id/client_secret, который скачивается/собирается из настроек OAuth-клиента. Это секрет, никогда не коммитить в git.
- **Publish App** — кнопка в GCP consent screen. Без неё приложение в режиме Testing, и refresh-токены Google **истекают каждые 7 дней** — без публикации через неделю всё отвалится само.

## Как ходят данные

1. **MCP-серверы (`gcal`, `gmail`) работают локально** на твоей машине, общаются с Google API напрямую по твоим credentials.
2. Когда Claude обрабатывает данные (читает письмо, смотрит календарь) — **содержимое уходит в Anthropic API**, как обычный запрос к Claude.
3. **Сами токены/credentials остаются локально** на диске — никуда, кроме Google, не отправляются.

## Pre-flight check

```powershell
Write-Host "Node: $(try { node -v } catch { 'NOT FOUND' })"
Write-Host "Claude CLI: $(try { claude --version } catch { 'NOT FOUND' })"
```

Если Node не установлен — https://nodejs.org (LTS).

**Время:** 15-25 минут на каждый сервис (календарь/почта отдельно, можно параллельно в двух окнах GCP-консоли).

## Шаг 1 — Google Cloud: проект и OAuth-клиент (~10 мин на сервис)

Рекомендуется **два отдельных GCP-проекта** — один для Calendar, один для Gmail (изоляция: если один клиент скомпрометирован, второй сервис не затронут). Можно и один проект на оба — работать будет, просто без этой изоляции.

1. https://console.cloud.google.com → создать проект (или выбрать существующий)
2. APIs & Services → Library → включить нужный API:
   - Календарь: **Google Calendar API**
   - Почта: **Gmail API**
3. APIs & Services → Credentials → **Create Credentials → OAuth client ID**
   - **Application type: Desktop app** (не Web application — это принципиально, см. Глоссарий)
   - Любое имя
4. Записать **Client ID** и **Client Secret** из созданного клиента
5. APIs & Services → OAuth consent screen → добавить свой email в **Test users** (иначе Google не даст пройти авторизацию)
6. **Сразу же (не откладывать!): OAuth consent screen → Publish App → Confirm.** Без верификации, просто снимает лимит Testing-режима. Иначе через 7 дней токен тихо умрёт и придётся авторизоваться заново.

## Шаг 2 — Credentials-файл (~2 мин на сервис)

Формат одинаковый для обоих сервисов:

```json
{"installed":{"client_id":"ТВОЙ_CLIENT_ID.apps.googleusercontent.com","project_id":"ТВОЙ_PROJECT_ID","auth_uri":"https://accounts.google.com/o/oauth2/auth","token_uri":"https://oauth2.googleapis.com/token","auth_provider_x509_cert_url":"https://www.googleapis.com/oauth2/v1/certs","client_secret":"ТВОЙ_CLIENT_SECRET","redirect_uris":["http://localhost"]}}
```

Сохранить как:
- Календарь: `C:\Users\<юзер>\.claude\google-calendar-mcp\gcp-oauth.keys.json`
- Почта: `C:\Users\<юзер>\.gmail-mcp\gcp-oauth.keys.json`

## Шаг 3 — Авторизация (~3-5 мин на сервис)

**Календарь:**
```powershell
$env:GOOGLE_OAUTH_CREDENTIALS = "C:\Users\<юзер>\.claude\google-calendar-mcp\gcp-oauth.keys.json"
npx -y @cocal/google-calendar-mcp@2.6.2 auth
```
Откроет браузер, попросит выбрать аккаунт и разрешить доступ. Токен сохранится в `%USERPROFILE%\.config\google-calendar-mcp\tokens.json`.

**Почта:**
```powershell
mkdir "$env:USERPROFILE\.gmail-mcp" -ErrorAction SilentlyContinue
npx -y @gongrzhe/server-gmail-autoauth-mcp@1.1.11 auth
```
(Файл `gcp-oauth.keys.json` должен уже лежать в `~\.gmail-mcp\` из Шага 2.) Выведет ссылку в консоль — открыть, авторизоваться. Токен сохранится в `~\.gmail-mcp\credentials.json`.

**Первый запуск может занять 20-40 секунд** — `npx` скачивает пакет. Это нормально, не признак ошибки.

## Шаг 4 — Регистрация MCP-серверов (~1 мин на сервис)

**Важно:** версии закреплены явно (`@2.6.2`, `@1.1.11`) — без версии `npx -y package` тянет `latest` при каждом старте, что означает доверие к тому, что мейнтейнер не сломает/не скомпрометирует пакет между твоими запусками. Проверить актуальные версии: `npm view @cocal/google-calendar-mcp version` / `npm view @gongrzhe/server-gmail-autoauth-mcp version`, обновлять вручную, не на автомате.

```powershell
claude mcp add gcal -s user -e GOOGLE_OAUTH_CREDENTIALS="C:/Users/<юзер>/.claude/google-calendar-mcp/gcp-oauth.keys.json" -- npx -y @cocal/google-calendar-mcp@2.6.2

claude mcp add gmail -s user -- npx -y @gongrzhe/server-gmail-autoauth-mcp@1.1.11
```

`-s user` — доступно из любого проекта/папки на этой машине. Убрать `-s user` (сделать `-s local`), если нужно только для текущего проекта.

Проверить: `claude mcp list` — оба должны быть `✔ Connected`. Если первая проверка после регистрации показала `✘ Failed to connect` — это тоже часто просто холодный старт `npx`, повторить `claude mcp list` через 20-30 секунд.

## Шаг 5 — Проверка

1. **Полностью перезапустить Claude Code** (не просто чат/вкладку — известная особенность: даже когда `claude mcp list` уже показывает `✔ Connected`, тулы вроде `mcp__gcal__list-calendars` не появляются в текущей сессии без полного перезапуска приложения).
2. Спросить: «покажи мои календари» / «прочитай последнее письмо во входящих»
3. Если не работает — `references/troubleshooting.md`

## Риски и осознанные компромиссы

- **Scope Gmail широкий по умолчанию** (`gmail.modify`) — Claude получает доступ читать всю почту, отправлять от твоего имени, управлять/удалять письма и лейблы. Письма — вектор prompt injection (текст письма теоретически может попытаться дать инструкцию агенту). Если нужен более узкий доступ — смотри опции пакета для сужения scope при первичной авторизации.
- **Community npm-пакеты, не официальные от Google/Anthropic.** Без пиннинга версии каждый запуск доверяет актуальному коду мейнтейнера. С пиннингом (см. Шаг 4) — доверие фиксируется на момент установки, обновление — осознанное действие, не автоматика.
- **Credentials на диске в открытом виде** (`gcp-oauth.keys.json`, `tokens.json`/`credentials.json`) — не зашифрованы. Стандартная практика для такого класса локальных интеграций, но не путать с «безопасно как в vault».

## Полный откат

1. `claude mcp remove gcal -s user` и/или `claude mcp remove gmail -s user`
2. Удалить файлы: `~/.claude/google-calendar-mcp/`, `~/.config/google-calendar-mcp/`, `~/.gmail-mcp/`
3. Отозвать доступ в Google-аккаунте: https://myaccount.google.com/permissions
4. Опционально удалить OAuth-клиент в GCP Console

## Справочник

- `references/troubleshooting.md` — все известные ошибки, включая ПОЧЕМУ встроенный коннектор Claude Code не работает
- Calendar: https://github.com/nspady/google-calendar-mcp
- Gmail: https://github.com/GongRzhe/Gmail-MCP-Server
