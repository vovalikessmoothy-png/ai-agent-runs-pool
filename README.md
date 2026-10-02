# ai-agent-runs-pool

Пул ранов GitHub Actions для [ai-agent-runner](https://github.com/trained-assist/ai-agent-runner) (экспериментальный аккаунт).

## Модель

```
триггер (issue label / repository_dispatch / ручной)
   → endpoint-приёмник (хвост CF-воркера, только JSON ≤ КБ)
      → dispatch СЮДА (этот репозиторий)
         → джоба: submit в Serverless API → SSE events в лог → result
            → большие данные НИКОГДА не идут через endpoint/API:
              прямая загрузка клиент⇄object storage, в API — только
              ссылка с TTL-токеном и sha256 (share-by-link, slice D1)
```

## Правила

- **Никаких гигабайтов через API/endpoint** — только метаданные и подписанные ссылки (GCS presigned / token-URL).
- Секреты — только в GitHub Secrets / GCP Secret Manager; в код и логи — никогда.
- Продуктовый код живёт в trained-assist/ai-agent-runner; здесь — только оркестрация джоб.

## Статус

Бутстрап. Легаси-эксперимент этого аккаунта (`gha-cluster-*`, `gha-worker-*`) — воркфлоу отключены 01.10.2026.

Входная точка: `POST https://llm-ladder.trainedassist.store/pool/trigger` (Bearer
`POOL_TRIGGER_TOKEN`) → `repository_dispatch` сюда → `.github/workflows/agent-task.yml`.
Пока секреты `RUNNER_API_URL`/`RUNNER_API_KEY` не заданы, джоба честно отвечает «принято»
и выходит 0; remote-режим (submit `RunSpec` с engine `fake` → poll status → result) включится
после деплоя VM-сервиса (runner-jobs#12).

## location (эпик ai-agent-run-api#1, Ф1)

Поле `location` приходит в `client_payload` от `POST /pool/trigger` и валидируется в двух местах:
на лестнице (400 с именем поля до dispatch'а) и здесь (защита в глубину для ручного
`workflow_dispatch`).

| `location` | Поведение receiver-джобы |
|---|---|
| `""` / не указано | полный запуск: submit в Serverless API → poll → result (наш пул) |
| `ru` / `eu` / `us` | **regional pool не подключён**: `location_reserved` → `::notice` + step summary + артефакт `location-reserved.json` (только метаданные: location, sha256 и длина task, время) → выход 0, submit **не выполняется** |
| прочее | отказ шага `location: expected one of …` (job red) |

Задача с зарезервированным регионом **не теряется молча**: запись видна в step summary,
`::notice`-аннотации и артефакте запуска workflow, лог несёт `location=<значение>`.
Логика самих регионов не строится — вне скоупа.
