# Service Debug Prompt

## Отладка сервиса - пошаговая инструкция

### 1. Проверка и подготовка окружения

**Важно**: Всегда запоминай текущий каталог и возвращайся в правильные каталоги для каждого действия!

```bash
# Запомнить изначальный рабочий каталог
INITIAL_CWD=$(pwd)
SERVICE_NAME=$(basename "$INITIAL_CWD")
echo "Рабочий каталог $SERVICE_NAME: $INITIAL_CWD"

# Проверить, что находимся в правильном каталоге сервиса
ls package.json src/ || echo "ОШИБКА: Неправильный каталог для сервиса!"
```

### 2. Освобождение порта 5000

**Обязательно перед каждым запуском сервиса:**

```bash
# Освободить порт 5000 перед запуском
fuser -k 5000/tcp 2>/dev/null || true
```

### 3. Обновление core-libs (при необходимости)

Если были изменения в core-libs, нужно их скопировать:

```bash
# Определение переменных окружения
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
FEATURE_NAME=$(basename "$CURRENT_BRANCH")

# Поиск базового каталога core-libs
CURRENT_DIR=$(pwd)
CORE_LIBS_BASE=""
while [[ "$CURRENT_DIR" != "/" ]]; do
  if [[ -d "$CURRENT_DIR/core-libs" ]] && [[ -d "$CURRENT_DIR/core-libs/stage" ]]; then
    CORE_LIBS_BASE="$CURRENT_DIR/core-libs"
    echo "Найден базовый каталог core-libs: $CORE_LIBS_BASE"
    break
  fi
  CURRENT_DIR=$(dirname "$CURRENT_DIR")
done

# Определение рабочих путей
CORE_LIBS="${CORE_LIBS_BASE}/${FEATURE_NAME}"
if [[ -d "$CORE_LIBS" ]]; then
  CORE_LIBS_CURRENT="$CORE_LIBS"
  echo "Найден фича-каталог: $CORE_LIBS_CURRENT"
else
  CORE_LIBS_CURRENT="${CORE_LIBS_BASE}/stage"
  echo "Фича-каталог не найден, используется fallback: $CORE_LIBS_CURRENT"
fi

# Копирование core-libs
if [[ -d "$CORE_LIBS_CURRENT" ]]; then
  echo "Копирование core-libs из $CORE_LIBS_CURRENT"
  rsync -av --delete "$CORE_LIBS_CURRENT/" node_modules/@bk/core-libs/
fi

# Вернуться в каталог сервиса
cd "$INITIAL_CWD"
```

### 4. Сборка сервиса

**Убедиться что находимся в каталоге сервиса:**

```bash
# Проверить правильность каталога
pwd
ls package.json src/ || echo "ОШИБКА: Неправильный каталог!"

# Сборка
npm run build
```

### 5. Запуск сервиса

```bash
# Убедиться что порт свободен
fuser -k 5000/tcp 2>/dev/null || true

# Запуск в фоновом режиме
npm run start:dev &

# Дождаться запуска сервиса (обычно 15-30 секунд)
echo "Ожидание запуска сервиса..."
sleep 20

# Проверить что сервис запустился на порту 5000
netstat -tlnp | grep :5000 || echo "Сервис еще не запущен, ожидаем еще..."
sleep 10
```

### 6. Проверка через curl

**Тестовые curl запросы:**

```bash
# Тест без авторизации (должен вернуть 401)
curl -X GET 'http://localhost:5000/api/v2/portfolios/yield/6306ae66-c541d6b0-f4dc2172' \
  -H 'accept: */*' \
  -s

# Тест с авторизацией (подставьте актуальные cookie из браузера; значения ниже — пример)
curl -X GET 'http://localhost:5000/api/v2/portfolios/yield/6306ae66-c541d6b0-f4dc2172' \
  -H 'accept: */*' \
  -H 'cookie: _bdk_token=<YOUR_TOKEN>; _bdk_session_id=<YOUR_SESSION_ID>; _bdk_id=<YOUR_DEVICE_ID>' \
  -s

# Тест с фильтрацией по датам (подставьте актуальные cookie)
curl -X GET 'http://localhost:5000/api/v2/portfolios/yield/6306ae66-c541d6b0-f4dc2172?startDate=2025-07-01&endDate=2025-07-15' \
  -H 'accept: */*' \
  -H 'cookie: _bdk_token=<YOUR_TOKEN>; _bdk_session_id=<YOUR_SESSION_ID>; _bdk_id=<YOUR_DEVICE_ID>' \
  -s

# Тест несуществующего портфеля (должен вернуть пустой массив []) — подставьте актуальные cookie
curl -X GET 'http://localhost:5000/api/v2/portfolios/yield/nonexistent-portfolio-id' \
  -H 'accept: */*' \
  -H 'cookie: _bdk_token=<YOUR_TOKEN>; _bdk_session_id=<YOUR_SESSION_ID>; _bdk_id=<YOUR_DEVICE_ID>' \
  -s
```

## Ожидаемые результаты

### Успешный ответ с данными:
```json
[
["2024-12-23",0.0306],
["2024-12-24",0.0334],
["2025-01-15",0.0833],
["2025-01-16",0.1049]
]
```

### Ошибка авторизации:
```json
{"message":"Invalid token, session or user","error":"Unauthorized","statusCode":401}
```

### Пустой результат:
```json
[]
```

## Типичные проблемы и решения

1. **Порт занят**: Выполнить `fuser -k 5000/tcp`
2. **Сервис не запускается**: Проверить каталог, выполнить `npm install`
3. **Ошибки сборки**: Проверить копирование core-libs
4. **401 ошибки**: Обновить cookie в curl запросах
5. **Неправильный каталог**: Всегда проверять `pwd` и `ls package.json`

### 7. Проверка тестов после изменений

**ВАЖНО**: После любых правок в коде обязательно запускать тесты!

```bash
# Вернуться в каталог сервиса (если вышли из него)
cd "$INITIAL_CWD"

# Проверить, что находимся в правильном каталоге
pwd
ls package.json src/ || echo "ОШИБКА: Неправильный каталог!"

# Запустить все тесты
npm test

# Или запустить тесты для конкретного сервиса
npm test -- --testNamePattern="PortfolioYieldService"

# Запустить тесты в silent режиме (меньше вывода)
npm test -- --testNamePattern="PortfolioYieldService" --silent
```

**Ожидаемый результат тестов:**
```
PASS src/portfolio-yield/portfolio-yield.service.spec.ts
Test Suites: X passed, Y total
Tests: Z passed, W total
```

**Если тесты падают:**
1. Внимательно прочитать ошибки
2. Проверить изменения в коде
3. Обновить тесты при изменении API
4. Убедиться что моки настроены правильно

## Последовательность действий

1. Запомнить текущий каталог
2. Освободить порт 5000
3. Обновить core-libs (если нужно)
4. Вернуться в каталог сервиса
5. **Запустить тесты** (после любых изменений кода)
6. Собрать сервис
7. Запустить сервис
8. Дождаться запуска
9. Проверить curl запросами

**ЗОЛОТОЕ ПРАВИЛО**: Всегда проверяй тесты после изменений в коде!
