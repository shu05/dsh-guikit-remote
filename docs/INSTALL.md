# Установка и настройка dsh-guikit-remote

Два узла: **сервер** — машина с dsh (в примере Linux), **хост** — управляемая Windows 10/11 (в примере `192.168.0.240`, юзер `admin`).

## Предварительные требования

| Компонент | Где | Зачем |
|---|---|---|
| dsh (`@deepseek-ai/dsh`) + Node ≥ 18 | сервер | сам агент |
| [Windows-MCP](https://github.com/CursorTouch/Windows-MCP) (streamable-http) | хост | единственный канал, видящий рабочий стол (интерактивная сессия) |
| OpenSSH-сервер на Windows + ключевой вход с сервера | хост | scp для скриншотов (`ssh-copy-id admin@<win>`) |
| [ollama](https://ollama.com) (по желанию) | сервер и/или хост | vision-модель для `gui_vision` |
| llama.cpp/strata с API `/v1/vram` (по желанию) | сервер | резервирование VRAM под vision-модель на общей карте |

На хосте должна быть **залогиненная и разблокированная** интерактивная сессия. RDP-подключение с дисконнектом ломает SendInput (рабочий стол «уезжает» в session 0).

## Шаг 1 — Windows-MCP на хосте

```powershell
# установка (winget по SSH часто не работает — проверен путь через uv):
uv tool install windows-mcp
```

Конфиг `C:\Users\<user>\.windows-mcp\config.toml`:

```toml
[server]
transport = "streamable-http"
host = "0.0.0.0"
port = 8000
auth_key = "ВАШ_СЕКРЕТНЫЙ_КЛЮЧ"
ip_allowlist = ["192.168.0.110"]   # IP сервера с dsh
```

Автозапуск — задача по логину (например `windows-mcp-server`); сервер пишет логи в `C:\Users\<user>\.windows-mcp\server.log`. Перезапуск с сервера:

```bash
ssh admin@<win> "schtasks /end /tn windows-mcp-server & schtasks /run /tn windows-mcp-server"
```

Проверка с сервера (URL **без** завершающего слэша, иначе 307):

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://<win>:8000/mcp   # 4xx с ключом = жив
```

## Шаг 2 — SSH/scp до хоста

```bash
ssh-copy-id admin@<win>
ssh -o BatchMode=yes admin@<win> "echo ok"
scp admin@<win>:"C:/Windows/System32/drivers/etc/hosts" /tmp/t && rm /tmp/t   # scp работает
```

SSH нужен **только** для файлов (скриншотов). Команды на рабочий стол по SSH гонять нельзя — session 0.

## Шаг 3 — установка плагина в dsh

```bash
git clone <этот репозиторий> && cd dsh-guikit-remote
dsh plugin --profile web add "$(pwd)"
```

или в Web UI: **Settings → Plugins → Add plugin → локальная папка** (пакет объявляет `dsh.bundle.patch`, плагин монтируется автоматически из `cordis.patch.yml` пакета).

Обязательно:

1. **Отключите локальный guikit**, если он включён (`include:dsh-guikit`) — иначе `tool "gui_screen" is already registered`.
2. Положите ключ windows-mcp в `$DSH_HOME/.env` (имя не должно начинаться с `DSH_`):
   ```bash
   echo 'WIN_MCP_KEY=ВАШ_СЕКРЕТНЫЙ_КЛЮЧ' >> ~/.dsh/.env
   ```
3. Перезапустите профиль: `systemctl restart dsh-web` (или как у вас запущен dsh). Тулзы появятся в **новых** сессиях.

## Шаг 4 — конфигурация

Дефолты пакета лежат в `cordis.patch.yml` (пример для `192.168.0.240`). Переопределяется в профиле (`~/.dsh/profiles/web/cordis.patch.yml`) — override заменяет config записи целиком:

```yaml
- id: dsh-guikit-remote
  config:
    mcp:
      url: http://192.168.0.240:8000/mcp
      token: !!js process.env.WIN_MCP_KEY || ''
    ssh:
      host: 192.168.0.240
      user: admin
      port: 22
    gate:
      maxWaitMs: 120000
    vision:
      gpu: server            # host = GPU Windows-машины, server = GPU машины с dsh
      host:
        baseUrl: http://192.168.0.240:11434/v1
        model: qwen3-vl:2b-instruct
        apiKey: ollama
        timeoutMs: 120000
      server:
        baseUrl: http://127.0.0.1:11434/v1
        model: qwen3-vl:2b-instruct
        apiKey: local
        timeoutMs: 120000
        manage: true         # загрузка/выгрузка модели по требованию
        strataUrl: http://127.0.0.1:8193
        reserveMib: 5120     # VRAM-резерв под модель (меньше — часть уедет в CPU)
        unloadMs: 120000     # простой до выгрузки
```

То же самое можно править галочками/полями в **Settings → Plugins**.

### vision.gpu = server

На машине с dsh нужен ollama (`OLLAMA_HOST=127.0.0.1:11434`, для systemd — обязательно `HOME=/root` и `OLLAMA_CONTEXT_LENGTH=4096`) и, если карта общая с LLM-сервером (strata/llama.cpp), `manage: true`: перед первым vision-запросом плагин резервирует `reserveMib` через `POST {strataUrl}/v1/vram {"reserve_mib":N}`, грузит модель, после `unloadMs` простоя выгружает (`keep_alive:0`) и возвращает резерв. Замерено на RTX 5090: холодный 6 с, тёплый 2 с.

### vision.gpu = host (по умолчанию)

На Windows ставится ollama (`OLLAMA_HOST=0.0.0.0:11434`, правило firewall на 11434, `OLLAMA_KEEP_ALIVE=-1` для резидентной модели). Проверенные модели для OCR интерфейсов: `qwen3-vl:2b-instruct` (2 ГБ VRAM, быстрый дефолт), `qwen3-vl:8b-instruct` (лучший OCR, ~8 ГБ). `gemma3:4b` и `moondream` галлюцинируют на текстах UI — не берите.

### Масштаб скриншотов

`scale: 0` (авто): снимок больше `maxDim` (2000 px) ужимается (4K → 50 %), вдвое экономя vision-контекст. `scale: 0.25..0.75` — фиксированная пропорция. Клики всегда в физических пикселях; связь `физ = origin + image_px / scale` плагин отдаёт в каждом снимке.

### Резидентный демон (скорость)

По умолчанию (`daemon.enabled: true`) плагин ускоряет горячий путь резидентным микро-сервером
на Windows: при первом же вызове он загружает `dshgkd.ps1` в `%TEMP%\dsh-guikit-remote\` и
запускает его **через windows-mcp** (чтобы процесс попал в интерактивную сессию и видел рабочий
стол), после чего ходит к нему по постоянному `ssh -N -L <port>:127.0.0.1:<port>` туннелю.
Никакой отдельной установки не нужно — достаточно уже требуемого SSH-ключа (шаг 2). Демон:

- отдаёт скриншоты base64 прямо в HTTP-ответе (scp не нужен) и умеет прикладывать скриншот к
  действию (`/act` с `settleMs` + `shot`) — клик и скриншот одним往返;
- держит desktop-gate сам (`GetLastInputInfo`), привязан к `127.0.0.1` (файрвол не трогается),
  ключ — GUID в `dshgkd.key`, при простое `idleExitMin` минут выходит и поднимается заново;
- при любом сбое прозрачно откатывается на windows-mcp-путь.

Отключить: `daemon.enabled: false`. Сменить порт: `daemon.port` (туннель и ключ подстроятся).
Замер (RTX 3060, 4K): `/shot` 0.45 с, клик+скриншот 0.41 с — против ~2 с и ~4 с на MCP-пути.

## Шаг 5 — проверка

В новой сессии dsh:

1. `gui_screen` → список мониторов хоста, виртуальный десктоп, курсор.
2. `gui_screenshot` → путь к PNG; откройте — это рабочий стол хоста.
3. `gui_vision { question: "что на экране?" }` → текстовый ответ.
4. `gui_window { action: list }` → список окон.

## Troubleshooting

| Симптом | Причина / решение |
|---|---|
| `USER_CONTROL {wait_seconds}` / тулза висит до таймаута | пользователь за машиной (desktop-gate). Попросите отойти или поднимите `gate.maxWaitMs`; принудительный отбор интерфейса — рестарт задачи windows-mcp |
| `The handle is invalid` при скриншоте по SSH | SSH-процесс в session 0 — за скриншоты отвечает плагин (windows-mcp + scp), не гоняйте PowerShell-скрипты по SSH |
| HTTP 307 от windows-mcp | слэш на конце `mcp.url` — уберите |
| `tool "gui_screen" is already registered` | включён локальный `dsh-guikit` — отключите |
| Тулзы не появились | перезапуск профиля + **новая** сессия чата |
| `dsh: sets "DSH_..." only the launching environment may set` | в `.env` нельзя переменные с префиксом `DSH_` — переименуйте (например `WIN_MCP_KEY`) |
| ollama грузит модель в CPU (медленно) | мало VRAM-резерва: поднимите `reserveMib` (замер: 4096 мало, 5120 — 100 % GPU) |
| PowerShell-скрипт не исполняется, `WinError 206` | команда >~10 КБ; плагин сам режет на чанки — если видите, значит кривой кастомный вызов |
| RDP-сессия отвалилась, клики «не работают» | разблокируйте/переподключите консоль хоста — SendInput живёт только в активной интерактивной сессии |
| тулзы медленные, в `via` нет `daemon` | демон не поднялся (нет SSH-ключа / порт занят / интерактивная сессия не активна). Проверьте `%TEMP%\dsh-guikit-remote\dshgkd.log` на хосте; это не ошибка — плагин сам работает через windows-mcp. `DSH_GUIKIT_DEBUG=1` печатает причину на сервере |
