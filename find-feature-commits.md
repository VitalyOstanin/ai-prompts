# Cherry-pick найденных фича-коммитов (PROJECT)

Команды для переноса немерджевых коммитов (старые → новые). Выполняйте из целевого репозитория и нужной ветки. Перед запуском убедитесь, что рабочее дерево чистое (без незакоммиченных изменений).

```bash
git cherry-pick <commit-1>
git cherry-pick <commit-2>
git cherry-pick <commit-3>
git cherry-pick <commit-4>
git cherry-pick <commit-5>
```

Примечания
- Источник: соседние фича-каталоги проекта (уровень выше текущего каталога).
- В выборке только коммиты с изменениями кода/документации; merge-коммиты исключены.
- Фича-каталоги (обезличенные):
  - PROJ-NNNN: /path/to/<project>/stage-PROJ-NNNN-<slug>
  - PROJ-NNNN: /path/to/<project>/stage-PROJ-NNNN-<slug> (подходящих коммитов не найдено)
  - PROJ-NNNN: /path/to/<project>/stage-PROJ-NNNN-<slug>
  - PROJ-NNNN, PROJ-NNNN: каталоги не найдены на уровне /path/to/<project>

---

## Как найти коммиты внутри фича‑каталогов

Ниже — универсальный скрипт, который:
- Находит фича‑каталоги по шаблону `stage-PROJ-NNNN-<slug>` на один уровень выше указанного пути.
- Берёт активную ветку (HEAD) в каждом каталоге.
- Собирает немерджевые коммиты, где менялись файлы кода/документации.
- (Опционально) Оставляет только коммиты, уникальные относительно базовой ветки (например, `stage`).
- Печатает готовый список команд `git cherry-pick` в хронологическом порядке (старые → новые).

```bash
#!/usr/bin/env bash
set -euo pipefail

# Входные данные
PARENT_DIR="/path/to/<project>"   # каталог, в котором лежат фича-каталоги (обычно: уровень выше текущего)
KEYS=(PROJ-9131 PROJ-9156 PROJ-9169 PROJ-9205 PROJ-9228)  # какие задачи искать
BASE_REF="stage"                  # базовая ветка для уникальных коммитов; оставьте пусто, чтобы не фильтровать

# Что считаем кодом/документацией
CODE_DOC_RE='(\.((kt|kts|java|go|rs|ts|tsx|js|jsx|py|rb|php|scala|swift|dart|c|cc|cpp|h|hpp|cs|proto|graphql|gql|sql|sh|bash|zsh|ps1|md|rst|adoc|txt|yml|yaml|json))$)|(^|/)docs?/|(^|/)README(\.|$))'

TMP_CMDS=$(mktemp)
TMP_META=$(mktemp)
trap 'rm -f "$TMP_CMDS" "$TMP_META"' EXIT

for key in "${KEYS[@]}"; do
  # Ищем каталоги только одного уровня глубины, имя содержит ключ
  while IFS= read -r DIR; do
    # Проверим, что это git-репозиторий
    git -C "$DIR" rev-parse --is-inside-work-tree >/dev/null 2>&1 || continue
    HEAD_REF=$(git -C "$DIR" rev-parse --abbrev-ref HEAD 2>/dev/null || echo HEAD)

    # Диапазон ревизий: HEAD минус BASE_REF (если задана и существует)
    REV_ARGS=("$HEAD_REF")
    if [[ -n "$BASE_REF" ]] && git -C "$DIR" rev-parse --verify "$BASE_REF" >/dev/null 2>&1; then
      REV_ARGS+=("--not" "$BASE_REF")
    fi

    # Обходим коммиты без merge
    while IFS= read -r C; do
      # Файлы в коммите
      if git -C "$DIR" show --name-only --pretty=format: "$C" \
           | sed '/^$/d' | grep -E "$CODE_DOC_RE" -q; then
        TS=$(git -C "$DIR" show -s --format='%ct' "$C")
        SH=$(git -C "$DIR" show -s --format='%h'  "$C")
        SUBJ=$(git -C "$DIR" show -s --format='%s'  "$C" | tr -d '\r\n')
        printf '%s\t%s\t%s\t%s\n' "$TS" "$SH" "$DIR" "$SUBJ" >> "$TMP_META"
      fi
    done < <(git -C "$DIR" rev-list --no-merges "${REV_ARGS[@]}")
  done < <(find "$PARENT_DIR" -maxdepth 1 -mindepth 1 -type d -iname "*${key}*" | sort)

done

# Печатаем команды cherry-pick по возрастанию времени
sort -n "$TMP_META" | awk -F '\t' '{print $2}' | uniq \
  | awk '{printf("git cherry-pick %s\n", $1)}' | tee "$TMP_CMDS"

# Подсказка: если объект коммита не найден в целевом репо,
# можно подтянуть его прямо из каталога-источника:
#   git fetch /path/to/source/worktree <sha>
# затем повторить cherry-pick.
```

Подсказки
- Если нужно увидеть, из какого каталога пришёл каждый коммит, посмотрите файл с метаданными (`$TMP_META` в скрипте) до сортировки.
- Чтобы получить только «последние N» коммитов, добавьте `-n N` к `rev-list`.
- Если базовой ветки `stage` нет, задайте `BASE_REF=""` — тогда выводятся все немерджевые коммиты HEAD.
