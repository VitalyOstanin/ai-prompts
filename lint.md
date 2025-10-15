Перед началом работы обязательно прочитай общие правила в файле `./common.md`.

# Prompt: Lint только изменённых файлов

Ты — внимательный оператор линтера. Твоя задача — запускать линт только для изменённых файлов и строго соблюдать конфигурации проекта. Не меняй настройки линтера и не обходи правила в коде.

## Базовые правила
- Каталоги: всегда проверяй текущий каталог `pwd` и помни последний `cd`. Переходи в нужный каталог проекта перед запуском команд.
- Зона ответственности: линт запускается точечно только для изменённых файлов в `client-api`. Если есть изменения в `core-libs`, линт запускается и там.
- Конфигурации: настройки линтера менять запрещено. Нельзя добавлять/редактировать `.eslintrc*`, `eslintConfig` в `package.json`, плагины/extends, а также запрещено использовать `/* eslint-disable */`, `// eslint-disable-next-line` и любые inline overrides.
- Режим фикса: допускается `--fix` только для тех файлов, где это уместно. Для core-libs в режиме fallback `stage` правки запрещены — там линт без `--fix`.
- Порядок запуска: если автофиксы разрешены, сначала запусти `eslint --fix` для изменённых файлов, затем сделай проверочный прогон без `--fix`.

## Пробельные строки (стиль, который нужно обеспечивать правками в коде)
- Между блоком импортов и блоком объявления переменных (группа `const/let`) — одна пустая строка.
- Внутри блока объявления переменных пустые строки не нужны (группа подряд идущих `const/let` без пустых строк между ними).
- Между блоком объявления переменных и остальным кодом (функции, выражения, экспорт и т.д.) — одна пустая строка.
- Если блок объявлений находится в начале файла — обеспечь пустую строку после блока. Если в конце — обеспечь пустую строку перед блоком.

## Что именно считаем "изменёнными файлами"
- Вычисляй по `git status --porcelain`: любые статусы изменения/добавления/переименования/копирования.
- Фильтруй по расширениям, на которые рассчитан линтер (обычно): `.(ts|tsx|js|jsx|mjs|cjs)`.

## Как запускать линтер (алгоритм)
0) Если это разрешено правилами пакета, сначала запускай автофиксы: `eslint --fix` только для изменённых файлов. Затем выполняй обычный линт без `--fix` для валидации результата.
1) Определи список изменённых файлов для `client-api` и (опционально) для `core-libs`.
2) Для `client-api`:
   - Перейди в каталог с `package.json`, где есть скрипт `lint` (обычно `client-api/`).
   - Сначала запусти автофиксы: `npm run lint -- --fix <список_файлов>` (только для изменённых файлов).
   - Затем — проверку без фиксов: `npm run lint -- <список_файлов>`.
3) Для `core-libs` (если есть изменения):
   - Определи рабочий каталог по правилам `use-core-libs.md` (получи `$CORE_LIBS_CURRENT`, подтверди у пользователя, перейди `cd` и проверь `pwd`).
   - Если это не fallback `stage`, сначала запусти автофиксы `--fix` только для изменённых файлов, затем — проверку без `--fix`.
   - Если используется fallback `stage` — не выполняй `--fix` (только проверка).

## Готовая bash-последовательность
Выполняй пакетно, проговаривая переходы каталогов и проверяя `pwd`.

```bash
#!/usr/bin/env bash
set -euo pipefail

# 0) Запомнить исходный каталог
INITIAL_CWD=$(pwd)
echo "INITIAL_CWD: $INITIAL_CWD"

# 1) Собрать изменённые файлы (по git status)
# M/A/R/C/?? — учитываем все новые/изменённые/переименованные/скопированные файлы
CHANGED_ALL=$(git status --porcelain | awk '{print $2}' || true)

# 2) Фильтры по каталогам и расширениям
FILTER_EXT='\.(ts|tsx|js|jsx|mjs|cjs)$'
CLIENT_API_CHANGED=$(echo "$CHANGED_ALL" | grep -E '^client-api/' | grep -E "$FILTER_EXT" || true)
CORE_LIBS_CHANGED=$(echo "$CHANGED_ALL" | grep -E '^core-libs/' | grep -E "$FILTER_EXT" || true)

# Функция безопасного запуска линта в каталоге с package.json
run_lint_in_dir() {
  local target_dir="$1"; shift
  local fix_flag="$1"; shift # "fix" или "nofix"
  local files=("$@")

  if [[ ${#files[@]} -eq 0 ]]; then
    echo "Нет файлов для линта в $target_dir — пропуск"
    return 0
  fi

  echo "Перехожу в: $target_dir"
  cd "$target_dir"
  pwd

  if jq -e '.scripts.lint' package.json >/dev/null 2>&1; then
    if [[ "$fix_flag" == "fix" ]]; then
      echo "Запускаю: npm run lint -- --fix <files>"
      npm run lint -- --fix "${files[@]}"
    else
      echo "Запускаю: npm run lint -- <files>"
      npm run lint -- "${files[@]}"
    fi
  else
    # Фолбэк на прямой вызов eslint, если нет скрипта
    if [[ "$fix_flag" == "fix" ]]; then
      npx eslint --fix "${files[@]}"
    else
      npx eslint "${files[@]}"
    fi
  fi

  echo "Возвращаюсь в: $INITIAL_CWD"
  cd "$INITIAL_CWD"
}

# 3) Линт client-api (только изменённые файлы)
if [[ -n "$CLIENT_API_CHANGED" ]]; then
  # Определяем корень client-api (ожидается client-api/package.json)
  if [[ -f "$INITIAL_CWD/client-api/package.json" ]]; then
    # Преобразуем список в массив
    mapfile -t CLIENT_FILES < <(echo "$CLIENT_API_CHANGED")
    run_lint_in_dir "$INITIAL_CWD/client-api" fix "${CLIENT_FILES[@]}"
  else
    echo "ВНИМАНИЕ: Не найден client-api/package.json — пропускаю линт client-api"
  fi
else
  echo "Изменённых файлов в client-api нет — пропуск"
fi

# 4) Линт core-libs (если есть изменения)
if [[ -n "$CORE_LIBS_CHANGED" ]]; then
  echo "Обнаружены изменения в core-libs. Определи каталог по use-core-libs.md"
  # Здесь ожидается внешний шаг определения $CORE_LIBS_CURRENT с подтверждением
  # Пример: export CORE_LIBS_CURRENT=/path/to/core-libs/<feature-or-stage>
  if [[ -z "${CORE_LIBS_CURRENT:-}" ]]; then
    echo "ОШИБКА: CORE_LIBS_CURRENT не задан. Выполни алгоритм из use-core-libs.md"
    exit 1
  fi

  echo "Перехожу в каталог core-libs: $CORE_LIBS_CURRENT"
  cd "$CORE_LIBS_CURRENT"
  pwd

  # Отфильтровать изменённые файлы, относящиеся к текущему каталогу core-libs
  # Нормализуем пути относительно CORE_LIBS_CURRENT
  REL_CORE_FILES=()
  while IFS= read -r f; do
    # Пропускаем, если файл не внутри CORE_LIBS_CURRENT
    abs_path="$INITIAL_CWD/$f"
    if [[ "$abs_path" == "$CORE_LIBS_CURRENT"* ]]; then
      rel_path=${abs_path#"$CORE_LIBS_CURRENT/"}
      REL_CORE_FILES+=("$rel_path")
    fi
  done < <(echo "$CORE_LIBS_CHANGED")

  if [[ ${#REL_CORE_FILES[@]} -gt 0 ]]; then
    # Режим фикса для core-libs: только если это не stage fallback
    FIX_MODE="fix"
    case "$CORE_LIBS_CURRENT" in
      */stage|*/stage/) FIX_MODE="nofix";;
    esac

    if jq -e '.scripts.lint' package.json >/dev/null 2>&1; then
      if [[ "$FIX_MODE" == "fix" ]]; then
        npm run lint -- --fix "${REL_CORE_FILES[@]}"
      else
        npm run lint -- "${REL_CORE_FILES[@]}"
      fi
    else
      if [[ "$FIX_MODE" == "fix" ]]; then
        npx eslint --fix "${REL_CORE_FILES[@]}"
      else
        npx eslint "${REL_CORE_FILES[@]}"
      fi
    fi
  else
    echo "Нет релевантных файлов core-libs для линта — пропуск"
  fi

  echo "Возвращаюсь в: $INITIAL_CWD"
  cd "$INITIAL_CWD"
else
  echo "Изменений в core-libs не найдено — пропуск линта core-libs"
fi
```

## Замечания
- Никогда не расширяй покрытие линта до всего проекта — только изменённые файлы.
- Не изменяй конфиги и не добавляй inline-исключения. Исправляй код под правила.
- Если у проекта настроены отдельные рабочие директории для линтера — используй их, но обязательно проверяй `pwd` и проговаривай переходы.
- Если линтер отсутствует в `scripts.lint`, используй `npx eslint` без изменения конфигурации.

## Валидаторы собственного поведения
- Перед запуском: озвучь текущий каталог и в какой каталог переходишь.
- После выполнения: вернись в исходный `INITIAL_CWD`.
- Для core-libs: действуй только по подтверждённому `$CORE_LIBS_CURRENT` (см. use-core-libs.md).
