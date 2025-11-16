## kTVCSS

Фуллстек‑платформа для киберспортивного проекта kTVCSS: сайт, матчмейкинг, статистика, турниры, чат, уведомления и интеграции. Репозиторий представляет собой монорепу из нескольких связанных сервисов и библиотек на .NET.

---

### Общая архитектура

- **Фронтенд**
  - **`kTVCSSBlazor`** – Blazor WebAssembly + ASP.NET Core Server:
    - клиент (`Client`) – основной UI (профили, матчи, команды, турниры, чат, фид, тикеты);
    - сервер (`Server`) – хостинг WASM, статика, небольшие вспомогательные API.
- **Backend‑сервисы**
  - **`kTVCSSBlazor.API`** – основной HTTP API:
    - авторизация (JWT), профили, игроки, матчи, BattleCup, команды, тикеты, контент, админ‑панель, GitHub‑интеграция и т.д.;
    - Telegram‑боты, web push‑уведомления, управление игровыми серверами, RCON.
  - **`kTVCSSBlazor.MatchManager`** – матчмейкинг:
    - очередь игроков, подбор 5v5 (Mix, HighRanked, Captains Mode);
    - Confirm Ready, тайм‑ауты и штрафы, распределение по серверам, запуск матчей через SSH.
  - **`kTVCSSBlazor.ChatHub`** – чат:
    - личные и групповые диалоги, онлайн‑статусы, typing‑индикации;
    - web push для новых сообщений.
- **Библиотеки**
  - **`kTVCSS.Db`** – доступ к данным:
    - Dapper‑репозитории и `EFContext` на SQL Server;
    - фасад `IRepository` для доменов (Players, Matches, Teams, Vips, Notifications и т.д.).
  - **`kTVCSS.Models`** – общие модели и DTO:
    - игроки, матчи, команды, тикеты, чат, BattleCup, push‑модели, статусы операций и др.
  - **`kTVCSS.Notifications`** – Web Push:
    - `PushNotificationService` (WebPush + VAPID), `PushNotificationOptions`.
  - **`kTVCSS.DomainRestrictionMiddleware`** – защита API:
    - middleware по Origin (`DomainRestrictionMiddleware`);
    - IP‑blacklist (`IIpBlacklistProvider`, `InMemoryIpBlacklistProvider`, `BlacklistOptions`).

---

### Технологии

- **Платформа**: .NET 10, C#, ASP.NET Core, Blazor WebAssembly.
- **БД**: Microsoft SQL Server (EF Core + Dapper).
- **Реальное время**: SignalR (чат, матчмейкинг).
- **Аутентификация**: JWT + куки (API), клиентская авторизация в Blazor.
- **Логирование**: NLog (файлы и база).
- **Уведомления**: WebPush, Telegram‑боты.
- **Интеграции**: RCON (CoreRCON), SSH (SSH.NET/Renci.SshNet), GitHub API, HtmlAgilityPack.
- **UI**: Radzen.Blazor, Blazored.LocalStorage.

---

### Репозитории и их роли

- **`kTVCSSBlazor`**
  - **Client**: SPA на Blazor WASM.
  - **Server**: хостинг клиента, статика, вспомогательные API (`/api/ping`, прокси в MatchManager и т.д.).
- **`kTVCSSBlazor.API`**
  - Контроллеры по доменам: авторизация, игроки, матчи, команды, BattleCup, тикеты, контент, уведомления, админ‑операции.
  - DI: `EFContext`, `IRepository`, сервисы (Tickets, Friends, MapPool, Notifications, GPT и др.).
  - Мидлвары: CORS, DomainRestriction, JWT, TokenRefresh, rate limiting.
- **`kTVCSSBlazor.ChatHub`**
  - Хаб `/chathub` + API‑контроллеры.
  - Сервис `ChatService` поверх `EFContext` + `IRepository`.
  - Push‑уведомления о новых сообщениях и фильтрация по чёрному списку (IP/PlayerId).
- **`kTVCSSBlazor.MatchManager`**
  - Хаб `/gamehub` (очередь игроков, Captains Mode, Mix‑чаты).
  - `Selection` (BackgroundService) – основной цикл матчмейкинга:
    - набор игроков, запуск pending‑матчей, штрафы и баны, Telegram‑оповещения.
  - Взаимодействие с БД (таблицы Players, GameServers, Mixes, MixesMaps, MixesAllowedPlayers и др.).
- **`kTVCSS.Db`**
  - `Context` (Dapper) и `EFContext` (EF Core).
  - Репозитории для Players, Matches, Teams, Admins, Vips, Notifications, UserFeed, MapPool и т.д.
  - `kTVCSSRepository` – фасад над всеми репозиториями.
- **`kTVCSS.Models`**
  - `Db/Models/*` – EF‑сущности и модели для Dapper (Players, Matches, BattleCup, Chat, Tickets, UserFeed и др.).
  - `Models/*` – API‑модели (регистрация, баны, Mix/MMR, GitHub, push и т.п.).
- **`kTVCSS.Notifications`**
  - `PushNotificationService` – отправка Web Push по подпискам (через `IPushSubscriptions` из `kTVCSS.Db`).
  - Поддержка VAPID‑ключей и автоматическое удаление устаревших подписок.
- **`kTVCSS.DomainRestrictionMiddleware`**
  - Middleware для фильтрации запросов по заголовку `Origin` и чёрному списку IP;
  - Extension `UseDomainRestriction(...)` и in‑memory реализация blacklist’а.

---

### Локальный запуск (dev)

1. **Предварительно**
   - Установить .NET 10 SDK.
   - Поднять SQL Server и накатить схему/данные (миграции + SP, см. `kTVCSS.Db` и `kTVCSSBlazor.API`).
   - Заполнить `appsettings.Development.json`:
     - `ConnectionStrings:db` и `ConnectionStrings:ef`.
     - `Jwt` (Issuer, Audience, Key).
     - `PushNotifications`, `Blacklist`, `api_white_paths`, токены Telegram‑ботов, SSH‑доступ и пр.

2. **Запуск сервисов**
   - `kTVCSSBlazor.API` – основной бэкенд.
   - `kTVCSSBlazor.MatchManager` – `/gamehub` + фоновые матчмейкинг‑воркеры.
   - `kTVCSSBlazor.ChatHub` – `/chathub`.
   - `kTVCSSBlazor.Server` – фронтенд:
     ```bash
     dotnet run --project kTVCSSBlazor/Server/kTVCSSBlazor.Server.csproj
     ```
   - Открыть сайт (порт берётся из `launchSettings.json` или вывода `dotnet run`).

3. **Фронтенд‑клиент**
   - В DEBUG клиент обращается к API по `http://localhost:3000` (можно изменить в `Client/Program.cs`).
   - Подключения к SignalR‑хабам:
     - чат – `https://chathub-host/chathub`;
     - матчмейкер – `https://mm-host/gamehub` (JWT в `access_token` query string).

---