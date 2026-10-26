# dsh-guikit-remote

**EN.** Windows desktop control for [DeepSeek Harness (dsh)](https://github.com/deepseek-ai/deepseek-harness) when dsh itself runs on *another* machine (e.g. a Linux server). The 15 `gui_*` tools execute on the remote Windows box through the [windows-mcp](https://github.com/CursorTouch/Windows-MCP) PowerShell channel (streamable-http MCP), screenshots are pulled back over scp, and `gui_vision` answers questions about the screen using the GPU of the **host** (the controlled Windows machine) or the **server** (the machine running dsh). For near-real-time speed the plugin also runs a **resident micro-server on Windows** reached through a persistent SSH tunnel — no per-call PowerShell spawn, screenshots inline as base64, and click/key + auto-screenshot in a single round trip (see *Скорость*); any daemon hiccup transparently falls back to windows-mcp.

**RU.** Плагин удалённого управления рабочим столом Windows для dsh, когда сам dsh живёт на другой машине (например на Linux-сервере). Скрипты тулзов выполняются на Windows через windows-mcp (единственный канал, сидящий в интерактивной сессии), скриншоты забираются по scp, а `gui_vision` отвечает на вопросы о экране видеокартой хоста (управляемая Windows) или сервера (машина с dsh) — по переключателю в конфиге.

## Зачем

Родной [dsh-guikit](https://github.com/gezi-wen/dsh-guikit) запускает PowerShell **локально на хосте dsh**. Если хост — Linux, это бессмысленно: UIA/SendInput/скриншоты рабочего стола существуют только внутри интерактивной Windows-сессии. Даже SSH на Windows-машину не помогает: порождённые SSH-процессы попадают в session 0 и рабочий стол не видят (`CopyFromScreen` → «The handle is invalid»).

Этот форк сохраняет весь PowerShell-слой dsh-guikit (UIA-обходчики, P/Invoke, линейки координат) и меняет только **транспорт**: скрипт улетает в `windows-mcp` (streamable-http MCP, работает в интерактивной сессии), PNG скриншота возвращается по scp, результат парсится обратно.

## Архитектура

```
dsh (Linux/сервер)                         Windows-машина (управляемая)
┌──────────────────────┐    streamable-http MCP (:8000)   ┌──────────────────────────┐
│ dsh-guikit-remote    │ ──── PowerShell tool ──────────► │ windows-mcp (интерактив. │
│  gui_* тулзы         │                                  │  сессия): SendInput, UIA,│
│                      │ ◄─── stdout JSON / exit code ─── │  PrintWindow, скриншоты  │
│                      │                                  └──────────────────────────┘
│  gui_screenshot      │ ──── scp (SSH, файлы) ─────────►  %TEMP%\dsh-guikit\*.png
│  gui_vision          │ ──── OpenAI-compatible ────────►  ollama (GPU хоста) или
└──────────────────────┘        /chat/completions          ollama+strata (GPU сервера)
```

- **Канал исполнения** — только windows-mcp: он сидит в интерактивной сессии и реально видит рабочий стол.
- **Скриншоты** — PNG пишется на Windows, забирается на сервер по scp (SSH для файлов работает, для рабочего стола — нет).
- **Длинные скрипты** — канал windows-mcp режет команды ~10 КБ, плагин сам бьёт скрипт на base64-чанки 7 КБ, собирает `.ps1` (UTF-8 BOM) и исполняет.
- **desktop-gate** — пока человек трогает мышь/клавиату управляемой машины, windows-mcp отвечает `USER_CONTROL`; плагин ждёт (`gate.maxWaitMs`) и честно падает по таймауту, а не мешает пользователю.

## Тулзы (15)

| Тулза | Назначение |
|---|---|
| `gui_screen` | раскладка мониторов, виртуальный десктоп, позиция курсора |
| `gui_screenshot` | снимок региона/экрана в PNG (сетка координат, крестик курсора, авто-даунскейл) |
| `gui_window_shot` | снимок одного окна через PrintWindow (работает, даже когда окно под другими) |
| `gui_window` | список/прямоугольник/активация/перемещение окон |
| `gui_uia` | UI Automation: tree/find/invoke/value по одному окну |
| `gui_locate` | поиск одного элемента по имени → координата для клика |
| `gui_verify` | стабильная (multi-sample) проверка состояния: элемент/окно/границы |
| `gui_wait` | ожидание: пиксель изменился / окно появилось |
| `gui_click` | реальный клик мышью (left/right/middle, 1–3 раза, window-local координаты) |
| `gui_drag` | реальный drag с интерполяцией (окна, слайдеры, drag-and-drop) |
| `gui_scroll` | колесо мыши в точке (вертикаль/горизонталь) |
| `gui_type` | ввод текста, включая Unicode/CJK (SendInput, мимо IME) |
| `gui_key` | клавиши и комбинации (ctrl+s, alt+f4, win, printscreen) |
| `gui_vision` | **новая**: снимок экрана → vision-модель → ответ; `gpu: host\|server` разово переключает видеокарту |
| `gui_lr_folder` | **бонус**: навигация по дереву папок Adobe Lightroom Classic (фильтр «Фильтровать папки») |

Координаты везде — **физические пиксели виртуального десктопа** (второй монитор слева — отрицательные X). Снимки больше `maxDim` (2000 px) авто-ужимаются; в ответе есть `scale`: физический пиксель = `origin + image_px / scale`, клики всегда физические.

## Конфигурация (кратко)

| Ключ | По умолчанию | Смысл |
|---|---|---|
| `mcp.url` | — | windows-mcp streamable-http endpoint (`http://<win>:8000/mcp`, без слэша на конце!) |
| `mcp.token` | `process.env.WIN_MCP_KEY` | auth-ключ windows-mcp |
| `ssh.host` / `.user` / `.port` | — / admin / 22 | scp-канал для скриншотов |
| `gate.maxWaitMs` | 120000 | сколько ждать desktop-gate'а, пока пользователь освободит мышь/клавиату |
| `daemon.enabled` | `true` | резидентный микро-сервер на Windows (см. «Скорость»); `false` — всё через windows-mcp |
| `daemon.port` | 18742 | порт демона на `127.0.0.1` (доступ только через SSH-туннель, файрвол не трогается) |
| `daemon.gateIdleMs` | 1500 | простой мыши/клавиатуры, после которого демон принимает ввод |
| `daemon.idleExitMin` | 15 | минут простоя до самовыхода демона (следующий вызов поднимет его заново) |
| `daemon.jpegQuality` | 80 | качество JPEG для inline-скринов `/act` и `gui_vision` |
| `vision.gpu` | `host` | какая видеокарта крутит `gui_vision`: `host` (Windows) или `server` (машина с dsh) |
| `vision.host.baseUrl/model` | `http://<win>:11434/v1` / `qwen3-vl:2b-instruct` | OpenAI-compatible endpoint ollama на Windows |
| `vision.server.baseUrl/model` | `http://127.0.0.1:11434/v1` / `qwen3-vl:2b-instruct` | endpoint ollama на сервере |
| `vision.server.manage` | `true` | загружать/выгружать модель по требованию |
| `vision.server.strataUrl` | `http://127.0.0.1:8193` | llama.cpp-сервер, у которого резервируется VRAM перед загрузкой |
| `vision.server.reserveMib` | 5120 | сколько VRAM резервировать (иначе ollama частично грузится в CPU) |
| `vision.server.unloadMs` | 120000 | простой до выгрузки модели и возврата резерва |
| `scale` / `maxDim` | 0 (авто) / 2000 | даунскейл скриншотов для vision-контекста |

Полное руководство по установке и настройке — **[docs/INSTALL.md](docs/INSTALL.md)**.

## Скорость (резидентный демон)

Каждый вызов через windows-mcp — это спавн PowerShell (~0.4 с) + scp-возврат скриншота
(~1 с): на скриншот уходит ~2 с, на клик со скриншотом — два таких往返. Чтобы приблизиться
к реальному времени, плагин поднимает на Windows **резидентный микро-HTTP-сервер** (демон):

- демон — PowerShell-скрипт, который плагин один раз загружает и запускает **через сам
  windows-mcp** (чтобы процесс попал в интерактивную сессию и видел рабочий стол), а дальше
  ходит к нему по **постоянному `ssh -N -L` туннелю** на `127.0.0.1:18742`;
- маршруты `GET /ping`, `POST /shot` (full/region/window, PNG|JPEG, даунскейл, base64 в
  ответе — scp не нужен), `POST /act` (click/drag/scroll/type/key/move/activate + `settleMs`
  + **inline-скриншот в том же ответе**), `POST /wait` (пиксель/окно, поллинг внутри демона),
  `POST /quit`;
- desktop-gate перенесён в демона (`GetLastInputInfo`): пока пользователь у машины — `423
  USER_CONTROL`, хост ждёт `wait_seconds` и повторяет;
- ключ доступа — GUID, лежит в `%TEMP%\dsh-guikit-remote\dshgkd.key`; демон привязан к
  `127.0.0.1`, файрвол не менялся; при простое `idleExitMin` минут выходит, следующий вызов
  поднимает его заново;
- **любой сбой демона прозрачно откатывает вызов на windows-mcp-путь** — пользователь не
  замечает разницы, кроме скорости.

Замерено на живой машине (RTX 3060, 4K-десктоп, туннель через gbit): `/shot` всего экрана
с даунскейлом до 2000px и JPEG-кодированием — **0.45 с** сквозной; `/act` клик с inline-скрином —
**0.41 с** одним往返 (было ~2 с и ~4 с соответственно). У `gui_click`/`gui_key`/`gui_type`/
`gui_scroll`/`gui_drag` появились параметры `settleMs` и `shot:true` — действие и скриншот
приходят в одном вызове.

## Быстрый старт

```bash
git clone <этот репозиторий> && cd dsh-guikit-remote
# установка в профиль dsh (или через Settings → Plugins → Add plugin → локальная папка):
dsh plugin --profile web add "$(pwd)"
# ключ windows-mcp:
echo 'WIN_MCP_KEY=ваш-ключ' >> ~/.dsh/.env
systemctl restart dsh-web   # или перезапуск вашего профиля
```

⚠️ Записи `include:dsh-guikit` (локальный guikit из коробки) быть не должно — тулзы конфликтуют именами (`tool "gui_screen" is already registered`). Новые тулзы появляются в **новых** сессиях.

## Проверка

```
gui_screen            → список мониторов управляемой машины
gui_screenshot        → PNG рабочего стола, доступен через read_image
gui_vision "что на экране?" → текстовый ответ vision-модели
```

## Реальный пример: конвейер Adobe Lightroom Classic

Плагин обкатан на полностью автоматизированном конвейере фотографа: запуск Lightroom → импорт папки CR3 → AI-обработка и кадрирование плагином AlbumLab (очередь задач на сервере) → экспорт JPEG. Пошаговые процедуры с проверенными координатами и ловушками — в [examples/skills/](examples/skills/):

- [lightroom-library-folder-nav](examples/skills/lightroom-library-folder-nav.md) — модуль Library, выбор папки по пути
- [lightroom-import](examples/skills/lightroom-import.md) — импорт папки с диска в каталог
- [lightroom-albumlab-ai](examples/skills/lightroom-albumlab-ai.md) — запуск AI-очереди и ожидание завершения
- [lightroom-export-jpeg](examples/skills/lightroom-export-jpeg.md) — массовый экспорт JPEG

## Ограничения

- Управляемая Windows должна иметь **залогиненную разблокированную интерактивную сессию** (RDP-дисконнект ломает SendInput).
- Пока пользователь активно работает мышью за машиной, тулзы ждут до `gate.maxWaitMs`.
- Канал windows-mcp: команды >~10 КБ не проходят (плагин сам режет на чанки), формат ответа `Response: …\r\n\nStatus Code: N`.
- Lightroom-специфичный `gui_lr_folder` — единственная тулза, знающая про конкретное приложение.

## Лицензия и благодарности

Apache-2.0 (см. [LICENSE](LICENSE), [NOTICE](NOTICE)). Производная от [dsh-guikit](https://github.com/gezi-wen/dsh-guikit) v0.6.2 (© gezi-wen, Apache-2.0); транспортный слой построен на [Windows-MCP](https://github.com/CursorTouch/Windows-MCP). Иконка — собственная.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
