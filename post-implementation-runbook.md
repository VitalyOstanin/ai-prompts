# Post-Implementation Runbook

Короткий регламент действий после реализации: мониторинг деплоя на stage и оформление задачи в YouTrack.

## Stage Deploy Monitoring

Что отслеживать
- Успешное завершение джоб `build stage` и `deploy stage` в последнем pipeline ветки `stage`.
- Итоговый статус pipeline = `success`.

Как проверять (пример цикла каждые 30 сек, до 15 минут)
1) Получить последний pipeline для `stage` и его `id`/`webUrl`.
2) Проверять статус джоб:
   - `build stage` → должен стать `success`;
   - `deploy stage` → должен перейти в `pending/running`, затем `success`.
3) Критерий готовности: обе джобы `success`, pipeline `success`.
4) Для отчёта сохранить ссылки:
   - Pipeline URL, Deploy Job URL.

Пример формулировки статуса
- Pipeline: #<iid> — success
  - Pipeline: <https://gitlab.example.com/group/project/-/pipelines/<id>>
  - Deploy job: <https://gitlab.example.com/group/project/-/jobs/<jobId>>

## YouTrack: Комментарий и статус

Что обязательно указать в комментарии
- “Выложено на stage”.
- Кликабельные коммиты:
  - client-api: `[<hash7>](https://gitlab.example.com/group/client-api/-/commit/<hash7>)`
  - @bk/core-libs: `[<hash7>](https://gitlab.example.com/group/core-libs/-/commit/<hash7>)`
- Ссылки на документацию:
  - stage: путь к файлу в коде в кодовых кавычках + прямая ссылка на GitLab.
  - master: аналогично (может появиться позже — указать это).

Как избежать поломки ссылок в YouTrack
- Пути к файлам оборачивать в кодовые кавычки (`` `path/to/file.md` ``), чтобы YouTrack не подставлял автоссылки внутрь.
- В заголовке использовать неразрывный дефис в идентификаторе задачи (пример: `BC‑9228`), чтобы не срабатывала автоссылка там, где это мешает форматированию.

Шаблон комментария (markdown)
```
BC‑XXXX: <краткое описание> (выложено на stage)

- Ветка: stage; commit: [<hash7>](https://gitlab.example.com/group/client-api/-/commit/<hash7>); @bk/core-libs: [<hash7>](https://gitlab.example.com/group/core-libs/-/commit/<hash7>).
- Кратко: <одно предложение о сути изменений>.
- Полное описание:
  - stage: `docs/<path>/BC-XXXX-<name>.md`
    https://gitlab.example.com/group/client-api/-/blob/stage/docs/<path>/BC-XXXX-<name>.md
  - master (после merge): `docs/<path>/BC-XXXX-<name>.md`
    https://gitlab.example.com/group/client-api/-/blob/master/docs/<path>/BC-XXXX-<name>.md
```

Статус задачи
- После успешного деплоя на stage перевести задачу в `Ready for Test`.

