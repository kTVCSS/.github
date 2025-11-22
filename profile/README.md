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
  - **`kTVCSS`** – сервис‑нода для игровых серверов:
    - мониторинг и управление игровыми серверами CS:S через RCON;
    - обработка событий матчей (раунды, урон, убийства, подключения, чат);
    - обработчики событий для статистики, модерации, резервных копий матчей;
    - интеграция с основной платформой через БД и HTTP API.
  - **`kTVCSS.GameserversManager`** – автоматическое управление игровыми серверами:
    - мониторинг состояния серверов в БД (колонка `BUSY`);
    - автоматический запуск/остановка серверов через `make` команды;
    - отложенная остановка серверов с проверкой состояния.
- **Десктоп‑приложение**
  - **`kTVCSS.Desktop`** – Electron‑приложение:
    - нативная обертка над веб‑сайтом ktvcss.com;
    - системные уведомления, блокировка засыпания, автозагрузка медиа;
    - автоматические обновления через GitHub Releases.

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
- **`kTVCSS`**
  - Сервис‑нода, работающий на каждом игровом сервере как отдельный процесс.
  - `NodeHostedService` – BackgroundService для управления жизненным циклом ноды.
  - `NodeOrchestrator` – оркестратор подключения к серверу через RCON и обработки событий.
  - Обработчики событий: подключение/отключение игроков, раунды, чат, урон, убийства, резервные копии матчей и др.
  - Логирование в файлы формата `{ServerId}_{dd_MM_yy}.log`.
  - Требует ID сервера в аргументах командной строки при запуске.
- **`kTVCSS.GameserversManager`**
  - Фоновый сервис для автоматического управления игровыми серверами.
  - `GameServerMonitorService` – опрос БД и мониторинг изменений `BUSY` в таблице `GameServers`.
  - Автоматический запуск серверов при `BUSY = 1` через `make up server=ktvcss-{ID}`.
  - Отложенная остановка серверов при `BUSY = 0` (настраиваемая задержка, по умолчанию 2 минуты).
  - Перезапуск node‑сервисов после запуска/остановки игровых серверов.
- **`kTVCSS.Desktop`**
  - Electron‑приложение для Windows, macOS и Linux.
  - Главное окно с размером 1720x900, загружает сайт https://ktvcss.com.
  - Системные уведомления через перехват веб‑уведомлений сайта.
  - Блокировка засыпания системы (`prevent-display-sleep`).
  - Поддержка кастомных звуков для уведомлений о найденной игре.
  - Автоматические обновления через GitHub Releases (electron-updater).
  - NSIS‑установщик для Windows с установкой для всех пользователей.
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

4. **kTVCSS (сервис‑нода)**
   - Требуется ID сервера в аргументах командной строки:
     ```bash
     dotnet run --project kTVCSS/kTVCSS.csproj 0
     ```
     Где `0` – индекс сервера в списке.
   - Конфигурация в `appsettings.json`:
     - `Configuration:SQLConnectionString` – подключение к БД.
     - `Configuration:SSHHost`, `SSHLogin`, `SSHPassword` – SSH доступ.
     - `Application:WordsFilterPath` – путь к файлу с запрещенными словами.
   - Каждая нода работает на отдельном игровом сервере как отдельный процесс.
   - Поддерживает Docker (см. `Dockerfile`).

5. **kTVCSS.GameserversManager**
   - Фоновый сервис для автоматического управления игровыми серверами.
   - Конфигурация в `appsettings.json`:
     - `ConnectionStrings:db` – подключение к БД.
     - `Monitoring:PollIntervalSeconds` – интервал опроса БД (по умолчанию 5 секунд).
     - `Monitoring:StopDelayMinutes` – задержка перед остановкой сервера (по умолчанию 2 минуты).
   - Запуск:
     ```bash
     dotnet run --project kTVCSS.GameserversManager/kTVCSS.GameserversManager.csproj
     ```
   - Для продакшена можно запускать как systemd service (см. `kTVCSS.GameserversManager/README.md`).
   - Требования: Linux, утилита `make` в PATH, доступ к БД и права на управление серверами.

6. **kTVCSS.Desktop**
   - Установка зависимостей:
     ```bash
     cd kTVCSS.Desktop
     npm install
     ```
   - Запуск в режиме разработки:
     ```bash
     npm start
     ```
   - Сборка установщика:
     ```bash
     npm run build
     ```
     Установщик создаётся в папке `dist/`.
   - Автообновления: настроены через GitHub Releases (см. `kTVCSS.Desktop/UPDATE_GUIDE.md`).

---

## Детальная документация по проектам

### kTVCSS (сервис‑нода)

Сервис‑нода для мониторинга и управления игровыми серверами Counter-Strike: Source. Работает на каждом игровом сервере как отдельный процесс и отвечает за обработку событий матчей, подключений игроков, RCON‑команд и интеграцию с основной платформой.

#### Архитектура

- **`NodeHostedService`** – основной BackgroundService:
  - запускает ноду для указанного сервера (ID передается через аргументы командной строки);
  - загружает конфигурацию серверов из БД и фильтр запрещенных слов;
  - управляет жизненным циклом `NodeOrchestrator`;
  - ведет логи в файлы формата `{ServerId}_{dd_MM_yy}.log`.

- **`NodeOrchestrator`** – оркестратор работы ноды:
  - подключается к игровому серверу через RCON;
  - слушает события сервера (подключения, отключения, раунды, матчи, чат и т.д.);
  - обрабатывает события через систему обработчиков.

- **Обработчики событий** (`Services/EventHandlers/`):
  - `PlayerConnectedEventHandler` – обработка подключения игроков;
  - `PlayerDisconnectedEventHandler` – обработка отключения игроков;
  - `RoundStartEventHandler` / `RoundEndEventHandler` – начало/конец раундов;
  - `ChatMessageEventHandler` – обработка сообщений в чате (модерация, команды);
  - `DamageEventHandler` / `KillFeedEventHandler` – статистика урона и убийств;
  - `MatchBackupEventHandler` – резервные копии матчей;
  - и другие обработчики для различных событий.

- **Основные сервисы**:
  - `MatchService` – управление матчами;
  - `PlayerService` – управление игроками;
  - `RconService` – взаимодействие с игровым сервером через RCON;
  - `ServerService` – управление серверами;
  - `MixService` – интеграция с системой Mix;
  - `ChatCommandService` – обработка команд в чате;
  - `MapSelectionService` – выбор карт.

#### Запуск

1. Требуется ID сервера в аргументах командной строки:
   ```bash
   dotnet run --project kTVCSS/kTVCSS.csproj 0
   ```
   Где `0` – индекс сервера в списке загруженных серверов.

2. При запуске программа:
   - загружает список серверов из БД;
   - находит сервер с указанным индексом;
   - подключается к нему через RCON;
   - начинает обработку событий.

3. Деплой:
   - запускается как отдельный процесс на каждом игровом сервере;
   - можно запускать как systemd service (Linux) или Windows Service;
   - поддерживает Docker (см. `Dockerfile`).

#### Особенности

- Логирование всех операций в файлы формата `{ServerId}_{dd_MM_yy}.log`.
- Обработка глобальных исключений для предотвращения краша.
- Автоматическая загрузка фильтра запрещенных слов из файла.
- Асинхронная обработка событий через систему обработчиков.
- Поддержка множественных серверов (каждая нода обслуживает один сервер).

---

### kTVCSS.Desktop

Десктопное приложение на базе Electron, которое является оберткой над веб‑сайтом ktvcss.com. Предоставляет нативный опыт работы с платформой на Windows, macOS и Linux.

#### Архитектура

- **`main.js`** – главный процесс Electron:
  - создает и управляет главным окном приложения;
  - обрабатывает системные уведомления через IPC;
  - предотвращает засыпание системы (`prevent-display-sleep`);
  - настраивает автоматическое обновление через `electron-updater` (GitHub Releases).

- **`preload.js`** – preload‑скрипт:
  - предоставляет безопасный API для рендерера (`electronAPI`);
  - перехватывает веб‑уведомления и преобразует их в системные;
  - нормализует пути к звукам для корректного воспроизведения;
  - поддерживает кастомные звуки уведомлений из localStorage.

- **Автообновление**:
  - проверка обновлений при запуске (через 5 секунд после старта);
  - периодическая проверка обновлений каждые 4 часа;
  - автоматическая загрузка и установка обновлений через GitHub Releases;
  - подробное логирование в DevTools и консоль.

#### Функциональность

1. **Системные уведомления**:
   - перехват веб‑уведомлений из сайта;
   - показ нативных уведомлений ОС;
   - поддержка кастомных звуков для уведомлений о найденной игре.

2. **Блокировка засыпания**:
   - предотвращает засыпание дисплея во время работы приложения.

3. **Автозагрузка медиа**:
   - предзагрузка аудио для звуков уведомлений.

4. **Автообновление**:
   - автоматические обновления через GitHub Releases.

#### Запуск и сборка

**Разработка:**
```bash
cd kTVCSS.Desktop
npm install
npm start
```

**Сборка установщика:**
```bash
npm run build
# Установщик создается в папке dist/
```

**Публикация релиза (требуется GH_TOKEN):**
```bash
export GH_TOKEN="ваш_github_token"
npm run build
```

#### Особенности

- Интеграция с веб‑сайтом без модификации кода сайта.
- Агрессивный перехват всех уведомлений через переопределение API.
- Поддержка кастомных звуков для уведомлений о найденной игре.
- Установка для всех пользователей системы (`perMachine: true`).

---

### kTVCSS.GameserversManager

Фоновый сервис для автоматического управления игровыми серверами на основе их состояния в базе данных. Мониторит колонку `BUSY` в таблице `GameServers` и автоматически запускает/останавливает серверы.

#### Архитектура

- **`GameServerMonitorService`** – BackgroundService:
  - опрашивает БД каждые N секунд (настраивается через `PollIntervalSeconds`);
  - отслеживает изменения `BUSY` для серверов с `TYPE != 3`;
  - автоматически запускает/останавливает серверы через команды `make`.

- **Логика работы**:
  - При `BUSY = 1`: выполняется команда `make up server=ktvcss-{ID}`.
  - При `BUSY = 0`: планируется остановка через N минут (настраивается через `StopDelayMinutes`), затем выполняется `make stop server=ktvcss-{ID}` после повторной проверки.

- **Защита от ложных срабатываний**:
  - отмена запланированной остановки, если `BUSY` изменился обратно на 1;
  - повторная проверка `BUSY` перед остановкой сервера.

#### Запуск

**Локально:**
```bash
dotnet run --project kTVCSS.GameserversManager/kTVCSS.GameserversManager.csproj
```

**Публикация для Linux:**
```bash
dotnet publish -c Release -r linux-x64 --self-contained
cd bin/Release/net10.0/linux-x64/publish
./kTVCSS.GameserversManager
```

**Как systemd service:**

Создайте файл `/etc/systemd/system/ktvcss-gameservers-manager.service`:

```ini
[Unit]
Description=kTVCSS Game Servers Manager
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/path/to/kTVCSS.GameserversManager
ExecStart=/path/to/kTVCSS.GameserversManager/kTVCSS.GameserversManager
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Затем:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ktvcss-gameservers-manager
sudo systemctl start ktvcss-gameservers-manager
```

#### Особенности

- Периодический опрос БД (polling) вместо real-time событий.
- Отслеживание состояния каждого сервера в памяти.
- Автоматический перезапуск node‑сервисов после запуска/остановки игровых серверов (`systemctl restart node-ktvcss{ID}`).
- Инициализация при старте: остановка всех серверов с `BUSY = 0`.
- Проверка состояния серверов через `make ps` перед запуском.

#### Требования

- Linux (для работы с `make` и `systemctl`).
- Утилита `make` в PATH.
- Доступ к базе данных SQL Server.
- Права на выполнение команд `make up/stop` в `/srv/cssold`.
- Права на управление systemd‑сервисами `node-ktvcss*`.

#### Интеграция

- Взаимодействует с таблицей `kTVCSS.dbo.GameServers` в БД.
- Команды `make` должны быть настроены для управления контейнерами/процессами серверов.
- После запуска/остановки сервера автоматически перезапускается соответствующий node‑сервис для синхронизации состояния.

---
