Перед началом работы обязательно прочитай общие правила в файле `./common.md`.

# Промпт: Синхронизация core-libs с client-api

## Инструкции для AI ассистента

### Задача
Определи рабочий каталог core-libs для текущего контекста и подключи его. При необходимости анализа или пересборки core-libs используй определённый каталог.

### Алгоритм выполнения

**ВАЖНО: Выполняй все шаги пакетно в одном цикле, минимизируя количество запросов к AI модели. Останавливайся только для подтверждения рабочего каталога у пользователя.**

#### Пакетное выполнение шагов 1-3 (без остановок):

**ИСПОЛЬЗУЙ ОДНУ bash команду для выполнения всего скрипта:**

```bash
#!/bin/bash

# 0. Сохранение изначального рабочего каталога
INITIAL_CWD=$(pwd)
SERVICE_NAME=$(basename "$INITIAL_CWD")
echo "Рабочий каталог $SERVICE_NAME: $INITIAL_CWD"

# 1. Определение FEATURE_NAME
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
if [[ -z "$CURRENT_BRANCH" ]]; then
  echo "ОШИБКА: Не удалось определить текущую ветку Git"
  exit 1
fi

FEATURE_NAME=$(basename "$CURRENT_BRANCH")
echo "Определен FEATURE_NAME: $FEATURE_NAME"

# 2. Поиск базового каталога core-libs
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

if [[ -z "$CORE_LIBS_BASE" ]]; then
  echo "ОШИБКА: Базовый каталог core-libs не найден"
  exit 1
fi

# 3. Определение рабочих путей и логика fallback
CORE_LIBS="${CORE_LIBS_BASE}/${FEATURE_NAME}"
echo "Проверяем фича-каталог: $CORE_LIBS"

if [[ -d "$CORE_LIBS" ]]; then
  CORE_LIBS_CURRENT="$CORE_LIBS"
  echo "Найден фича-каталог: $CORE_LIBS_CURRENT"
else
  CORE_LIBS_CURRENT="${CORE_LIBS_BASE}/stage"
  echo "Фича-каталог не найден, используется fallback: $CORE_LIBS_CURRENT"
fi

if [[ ! -d "$CORE_LIBS_CURRENT" ]]; then
  echo "ОШИБКА: Каталог $CORE_LIBS_CURRENT не существует"
  exit 1
fi

echo "Будет использован рабочий каталог core-libs: $CORE_LIBS_CURRENT"
```

#### 4. ЕДИНСТВЕННАЯ ОСТАНОВКА - Подтверждение рабочего каталога
- **ОБЯЗАТЕЛЬНО** покажи пользователю определённый рабочий каталог core-libs из вывода скрипта
- Запроси подтверждение: "Будет использован рабочий каталог core-libs: $CORE_LIBS_CURRENT. Продолжить? (y/n)"
- Если пользователь не подтвердил - **остановись**
- Только после подтверждения продолжай выполнение

#### Шаг 5. Системное разрешение и подключение контекста:

**ОБЯЗАТЕЛЬНАЯ ОСТАНОВКА** для получения системного разрешения AI агента

**Подключение в контекст:**
- **ВАЖНО:** Инициируй системное разрешение AI агента (Claude Code CLI) на доступ к каталогу `$CORE_LIBS_CURRENT`
- Запроси у системы безопасности: "Требуется доступ к рабочему каталогу core-libs для анализа и работы с кодом"
- **Только после получения системного разрешения** подключай определённый каталог `$CORE_LIBS_CURRENT` в текущий рабочий контекст
- Используй этот каталог для всех операций с core-libs (чтение файлов, анализ кода, сборка)
- Если система безопасности отказала в доступе - **остановись**, но сообщи что каталог определён как `$CORE_LIBS_CURRENT`

#### Шаг 6. Выполнение операций (при необходимости):

   **Анализ и чтение кода:**
   - Используй `$CORE_LIBS_CURRENT` для всех операций чтения файлов
   - Анализируй код в определённом каталоге

   **Пересборка (только если требуется):**
   1. **Проверка возможности изменений:**
      - Если используется stage fallback - **остановись с ошибкой**
      - Продолжай только если `$CORE_LIBS_CURRENT` это фича-каталог

   2. **Переход в каталог core-libs:**
      ```bash
      cd $CORE_LIBS_CURRENT
      ```
      - Если переход не удался - **остановись с ошибкой**
      - Проверь успешность: `pwd` должен показать `$CORE_LIBS_CURRENT`

   3. **Сборка (только после успешного перехода):**
      ```bash
      npm run build
      ```
      - Если сборка не удалась - **остановись с ошибкой**

   **Синхронизация с client-api (только если требуется):**
   1. **Проверка SERVICE каталога:**
      - Используй сохраненный `$INITIAL_CWD` как SERVICE каталог
      - Проверь существование `$INITIAL_CWD/node_modules/@bk/core-libs/`
      - Если отсутствует - **остановись с ошибкой**

   2. **Копирование:**
      ```bash
      rsync -a --exclude='.git' --exclude='node_modules' --quiet ${CORE_LIBS_CURRENT}/ ${INITIAL_CWD}/node_modules/@bk/core-libs/
      ```
      - Используй rsync с исключением `.git` и `node_modules` в silent режиме
      - Если копирование не удалось - **остановись с ошибкой**

#### 7. Правила обработки ошибок
- **Чтение данных:** можно использовать stage как fallback
- **Изменения в коде:** только в фича-каталогах, иначе ошибка
- При любой ошибке - немедленная остановка с четким сообщением
- Всегда указывай, какой каталог используется (`$CORE_LIBS_CURRENT`)

#### 8. Сообщения
- Четко сообщай о найденном базовом каталоге core-libs и запрашивай подтверждение
- Показывай итоговый используемый каталог core-libs
- При fallback к stage - предупреди об этом
- При ошибках - укажи конкретную причину и что не удалось выполнить
- Всегда выводи путь `$CORE_LIBS_CURRENT` который подключен в контекст

### Примеры использования

#### Успешное определение и подтверждение:
```
Рабочий каталог client-api: /home/user/projects/bitkogan/code/client-api
Исходное название ветки: feature/stage-BC-8952-packages-isBought
FEATURE_NAME определен как: stage-BC-8952-packages-isBought
Поиск базового каталога core-libs...
Найден базовый каталог core-libs: /home/user/projects/bitkogan/code/core-libs
Будет использован рабочий каталог core-libs: /home/user/projects/bitkogan/code/core-libs/stage-BC-8952-packages-isBought/
Продолжить? (y/n): y
Фича-каталог core-libs подключен в контекст
```

#### Fallback к stage с подтверждением:
```
Рабочий каталог client-api: /home/user/projects/bitkogan/code/client-api
Исходное название ветки: feature/stage-BC-8952-packages-isBought
FEATURE_NAME определен как: stage-BC-8952-packages-isBought
Поиск базового каталога core-libs...
Найден базовый каталог core-libs: /home/user/projects/bitkogan/code/core-libs
Фича-каталог не найден, используется fallback к stage
Будет использован рабочий каталог core-libs: /home/user/projects/bitkogan/code/core-libs/stage/
Продолжить? (y/n): y
Stage каталог core-libs подключен в контекст для операций чтения
```

#### Ошибка - базовый каталог не найден:
```
Рабочий каталог client-api: /home/user/projects/bitkogan/code/client-api
FEATURE_NAME определен как: stage-BC-8942-portfolio-yield
Поиск базового каталога core-libs...
ОШИБКА: Базовый каталог core-libs не найден в файловой структуре
```

#### Отказ пользователя от подтверждения:
```
Рабочий каталог client-api: /home/user/projects/bitkogan/code/client-api
Исходное название ветки: feature/stage-BC-8952-packages-isBought
FEATURE_NAME определен как: stage-BC-8952-packages-isBought
Поиск базового каталога core-libs...
Найден базовый каталог core-libs: /home/user/projects/bitkogan/code/core-libs
Будет использован рабочий каталог core-libs: /wrong/path/core-libs/stage-BC-8952-packages-isBought/
Продолжить? (y/n): n
Операция отменена пользователем
```

#### Успешная пересборка:
```
Рабочий каталог client-api: /home/user/projects/bitkogan/code/client-api
CORE_LIBS_CURRENT: /home/user/projects/bitkogan/code/core-libs/stage-BC-8952-packages-isBought/
Сборка core-libs: ✓
Синхронизация с client-api: ✓
```
