# MCP Debugger (HTTP) — Универсальный сценарий отладки с минимальными запросами

## Дополнительные инструкции (оперативные)
- Поменьше останавливайся, работай. Останавливайся только если явно запросил пользователь.
- Ставь breakpoints/logpoints, внимательно следуя подсказкам MCP tools (верные source/line/column, executable locations, verified=true).
- При перезапуске MCP HTTP сервера сразу запрашивай разрешение у AI agent sandbox (elevated permissions, on-request). Не спрашивай отдельно у пользователя.
- Всегда работай в каталоге запуска проекта.
- Если initialize вернул "Server already initialized" — сразу перезапусти MCP HTTP сервер на 3333 (останови процесс на порту 3333 и подними `http-server` заново).
- При проблемах с MCP (порт 3333), обязательно запроси перезапуск MCP HTTP сервера: если /health не OK, initialize/attach не проходят (например, «Server already initialized» без mcp-session-id), либо порт занят.
- В переписке не показывай код/скрипты взаимодействия с MCP и внутренние команды. Сообщай только действия и результаты по сценарию отладки: 
  - постановка logpoint с координатами, путь и контекст с зелёным маркером,
  - запуск запроса к сервису,
  - сбор попаданий,
  - сокращённый JSON результатов (включая поле hasAccess).
- При любых сбоях MCP — остановись и сообщи о проблеме.


Цель: подключиться к Node.js процессу на 9229 через MCP (HTTP Streamable), поставить logpoint в произвольной точке кода (TS/JS), запустить нужный сценарий приложения (HTTP-запрос, задача, скрипт и т.п.), собрать попадания и красиво вывести данные — с минимумом запросов к пользователю.

## Предварительные условия
- Node.js и npm установлены (сервер тестировался на Node 22.x).
- MCP сервер проекта: `~/devel/mcp-chrome-debugger-protocol` (собранный `dist/`).
- Отлаживаемый Node.js процесс запущен с инспектором на `:9229` (`node --inspect ...`).

## 1) Проверка и запуск MCP HTTP сервера (порт 3333)
```bash
# Проверить, свободен ли 3333
fuser 3333/tcp || true

# Если пусто — запустить сервер на 3333
cd ~/devel/mcp-chrome-debugger-protocol
nohup npm run http-server -- --port 3333 > .mcp-http.log 2>&1 & echo $!

# Проверить порт и health
fuser 3333/tcp || true
curl -fsS http://localhost:3333/health
```
Примечание: принудительно задаем `--port 3333`, чтобы избежать конфликтов.

## 2) Инициализация MCP сессии (HTTP Streamable)
```bash
# Инициализация (нужен Accept: application/json, text/event-stream)
curl -sS -D /tmp/mcp_init_headers.txt \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Content-Type: application/json' \
  --data '{
    "jsonrpc":"2.0","id":1,"method":"initialize",
    "params":{
      "protocolVersion":"2025-03-26",
      "capabilities":{},
      "clientInfo":{"name":"local-cli","version":"1.0.0"}
    }
  }' \
  http://localhost:3333/mcp | sed -n '1,1p'

# Извлечь mcp-session-id из заголовков
export MCP_SID=$(awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}' /tmp/mcp_init_headers.txt | tr -d '\r')
echo "MCP_SID=$MCP_SID"

# Завершить handshake: уведомление initialized
curl -sS -o /dev/null \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Content-Type: application/json' \
  -H "mcp-session-id: $MCP_SID" \
  -H 'mcp-protocol-version: 2025-03-26' \
  --data '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}' \
  http://localhost:3333/mcp
```
Если пропустить заголовки — будет 406/400.

## 3) Подключение к отладчику (9229)
```bash
curl -sS -N \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Content-Type: application/json' \
  -H "mcp-session-id: $MCP_SID" \
  -H 'mcp-protocol-version: 2025-03-26' \
  --data '{
    "jsonrpc":"2.0","id":2,
    "method":"tools/call",
    "params":{"name":"attach","arguments":{}}
  }' \
  http://localhost:3333/mcp | sed -n 's/^data: //p'
```
По умолчанию `attach` подключается к `localhost:9229`. Можно передать `{ "port": 9229 }` или `{ "url": "ws://127.0.0.1:9229/..." }`.

## 4) Постановка logpoint (произвольная точка)
Выберите исполняемую строку в вашем файле (после нужных присвоений/вычислений). Пример:
- Файл: `SRC=/abs/path/to/your/file.ts`
- Строка/колонка: `LINE=NN`, `COL=MM`
- Шаблон логирования: используйте безопасные подстановки `{expr}`. Для структур — оборачивайте в `JSON.stringify(...)`.

```bash
export SRC="/abs/path/to/your/file.ts"   # realpath к файлу
export LINE=NN                            # 1-based
export COL=MM                             # 1-based, на исполняемом фрагменте
export LOGMSG='payload={JSON.stringify(yourObjectOrArray)}'  # или 'x={x} y={y}'

jq -n --arg src "$SRC" --argjson line "$LINE" --argjson col "$COL" --arg msg "$LOGMSG" '{
  jsonrpc:"2.0", id:3, method:"tools/call",
  params:{ name:"setBreakpoints", arguments:{
    source:{ path:$src },
    breakpoints:[{ line:$line, column:$col, logMessage:$msg }]
  }}
}' | curl -sS -N \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Content-Type: application/json' \
  -H "mcp-session-id: $MCP_SID" \
  -H 'mcp-protocol-version: 2025-03-26' \
  --data-binary @- http://localhost:3333/mcp | sed -n 's/^data: //p'
```
Советы:
- Если нет попаданий — попробуйте соседние строки/колонки, главное — чтобы точка была на реально исполняемом участке.
- Можно ставить несколько logpoint'ов за один вызов `setBreakpoints`.

## 5) Показать logpoint с контекстом и маркером (локально)
```bash
# Контекст (3 строки до и 1 после)
nl -ba "$SRC" | sed -n "$((LINE-3)),$((LINE+1))p"

# Координаты logpoint
printf "
marker@%s:%d:%d
" "$SRC" "$LINE" "$COL"

# Inline-маркер: вставляем 🟢 прямо в код на нужной колонке
awk -v target="$LINE" -v col="$COL" 'NR>=target-3 && NR<=target+1{
  if (NR==target){
    pre=substr($0,1,col-1); post=substr($0,col);
    printf("%6d	%s%s%s
", NR, pre, "🟢", post);
  } else {
    printf("%6d	%s
", NR, $0);
  }
}' "$SRC"
```
## 6) Триггер сценария приложения и сбор попаданий
- Запустите то, что приводит к исполнению выбранной строки: HTTP-запрос, CLI-команда, воркер и т.п.
  - Пример HTTP: `curl -s -o /dev/null 'http://localhost:5000/your-endpoint'`
  - Или воспроизведите шаги в UI/скрипте.
- Подождите 1–2 секунды (или синхронизируйтесь иным способом).

Забрать попадания logpoint:
```bash
curl -sS -N \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Content-Type: application/json' \
  -H "mcp-session-id: $MCP_SID" \
  -H 'mcp-protocol-version: 2025-03-26' \
  --data '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"getLogpointHits","arguments":{}}}' \
  http://localhost:3333/mcp | sed -n 's/^data: //p' | tail -n1 | tee /tmp/hits.json

# Пример извлечения полезной нагрузки (если в logMessage использовался JSON.stringify(...))
jq -r '.result.content[0].text' /tmp/hits.json | jq -r . > /tmp/hits_parsed.json
jq -r '.hits[-1].payload.vars["JSON.stringify(yourObjectOrArray)"]' /tmp/hits_parsed.json | jq -r '.' > /tmp/log_payload.json

# По необходимости — сократить структуру до ключевых полей
jq '[ .[] | {id: (.id // ._id), title: (.title // .name), hasAccess: (.hasAccess)} ]' /tmp/log_payload.json | jq '.'
```

## 7) Полезные варианты
- Список/снятие брейкпоинтов: `tools/call getBreakpoints`, `tools/call removeBreakpoint`, повторный `setBreakpoints`.
- Очистка попаданий: `tools/call clearLogpointHits`.
- Отладочные события: `tools/call getDebuggerEvents`.
- Завершение MCP-сессии (опционально):
```bash
curl -sS -X DELETE \
  -H "mcp-session-id: $MCP_SID" \
  -H 'mcp-protocol-version: 2025-03-26' \
  http://localhost:3333/mcp -D -
```
- Останов MCP HTTP сервера: найдите PID, завершите процесс (если поднимали через `nohup`, PID печатался при запуске).

## Минимизация запросов разрешений
- Проверяйте порты через `fuser` и `/health`, перезапускайте только если нужно.
- Работайте через локальный HTTP (`localhost`) и одну MCP-сессию.
- Всегда отправляйте правильные заголовки (`Accept`, `mcp-session-id`, `mcp-protocol-version`).
- Ставьте logpoint на гарантированно исполняемые строки, чтобы избежать повторных попыток.
- Для парсинга используйте локальный `jq`, не требующий сети.

## Перезапуск MCP HTTP сервера (порт 3333)
Если при инициализации видите «Server already initialized», нет `mcp-session-id` в ответе, `/health` не OK, либо порт 3333 занят — выполните перезапуск.

### Шаги
```bash
# 1) Остановить текущий процесс (если есть)
cd ~/devel/mcp-chrome-debugger-protocol
if [ -f .mcp-http.pid ]; then
  kill "$(cat .mcp-http.pid)" 2>/dev/null || true
  sleep 0.3
  rm -f .mcp-http.pid
fi
# На всякий случай убить слушатель порта 3333
fuser -k 3333/tcp || true

# (Опционально) Завершить залипшую MCP-сессию, если сервер отвечает
curl -sS -X DELETE \
  -H "mcp-protocol-version: 2025-03-26" \
  http://localhost:3333/mcp -o /dev/null || true

# 2) Запустить заново на 3333 и сохранить PID
cd ~/devel/mcp-chrome-debugger-protocol
nohup npm run http-server -- --port 3333 > .mcp-http.log 2>&1 & echo $! | tee .mcp-http.pid

# 3) Проверить порт и /health
fuser 3333/tcp || true
curl -fsS http://localhost:3333/health

# 4) Повторить initialize с заголовками и взять новый mcp-session-id
#   - Accept: application/json, text/event-stream
#   - Content-Type: application/json
#   - mcp-protocol-version: 2025-03-26
```

Примечания:
- Всегда задавайте `--port 3333`, чтобы избегать конфликтов.
- Если перезапуск не помогает, проверьте, что Node-процесс отладки запущен с `--inspect=9229`.
- Для чистоты можно очистить попадания: `tools/call clearLogpointHits`.
