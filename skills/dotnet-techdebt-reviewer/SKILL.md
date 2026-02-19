---
name: dotnet-techdebt-reviewer
description: Senior .NET reviewer and architect workflow for deep technical debt audits in .NET projects. Use when user asks for hack report, bad practices review, dirty code audit, architecture/code smell analysis, or full issue inventory with evidence and fixes.
---

# .NET TechDebt Reviewer

## Goal
Сделать полный аудит .NET проекта и выдать детальный markdown-отчёт по техдолгу/костылям/рискам.

## Workflow
1. Определи корень проекта (`*.sln`, `*slnxmc`, `src/`, `lib/`, `tests/`).
2. Собери факты через поиск по коду (`rg`) и точечное чтение файлов (`nl -ba`, `sed`).
3. Обязательно проверь:
   - EF Core: `AsNoTracking`, `Include/ThenInclude`, `HasPrecision`, `HasColumnType`, `HasConversion`, `FromSql*`, `ExecuteSql*`.
   - DB/type mismatch: `Guid.Parse`, `DateTime`/timezone, decimal conversions, raw SQL.
   - Redis: `ConnectionMultiplexer`, TTL/expiration, serialization, stampede, fire-and-forget.
   - Metrics/Prometheus: наличие инструментирования, high-cardinality labels.
   - API governance:
     - если разрабатывается API, должна быть генерация OpenAPI-спецификации (swagger/openapi contract как артефакт сборки или runtime endpoint);
     - контракт OpenAPI должен быть актуальным относительно реально доступных endpoint'ов.
   - Health endpoints:
     - в приложении должны быть ручки `health` и `liveness` (и отдельно `readiness`, если применимо);
     - проверки должны отражать критичные зависимости (БД, брокеры, внешние API) для readiness.
   - Config: hardcoded URL/timeout/retry/backoff/batch/page-size, значения в коде вместо options.
   - 12-Factor App соответствие (принципы манифеста):
     - I. Codebase: один codebase, много deploy.
     - II. Dependencies: явное объявление и изоляция зависимостей.
     - III. Config: конфигурация в environment, не в коде.
     - IV. Backing services: внешние ресурсы как attached resources.
     - V. Build, release, run: строгое разделение стадий.
     - VI. Processes: stateless processes.
     - VII. Port binding: экспорт сервиса через binding порта.
     - VIII. Concurrency: масштабирование через процессы.
     - IX. Disposability: быстрый старт и graceful shutdown.
     - X. Dev/prod parity: минимальный разрыв окружений.
     - XI. Logs: логи как event stream.
     - XII. Admin processes: одноразовые админ-задачи как отдельные процессы.
   - Логирование:
     - отсутствие логов в критических местах (external calls, DB операции, retries, ошибки, таймауты, circuit-breaker события, запуск/остановка background consumer);
     - отсутствие корреляционных идентификаторов (`ActivityId`, `TraceId`, `SpanId`) в ключевых логах;
     - слишком «тихие» `catch`/ошибки без структурированного лога и контекста.
   - Kafka:
     - использование `consumer group` (обязательно для production consumers);
     - корректная стратегия offset commit (manual/auto, at-least-once/at-most-once);
     - idempotent producer и `acks`/retries/delivery timeout;
     - DLQ/retry-topic стратегия для poison messages;
     - обработка ребалансов и graceful shutdown consumers;
     - keying/partitioning стратегия (порядок событий, hot partitions);
     - schema/versioning контрактов сообщений (backward compatibility);
     - security (SASL/SSL, секреты не в коде);
     - observability (lag, rebalance count, error rate, throughput, latency).
4. Дедуплицируй одинаковые замечания.
5. Для каждого замечания укажи:
   - `Область`: `db | efcore | redis | metrics | config | kafka | other`
   - `Категория`: `security | reliability | performance | architecture | maintainability | testability`
   - `Влияние`: `high | medium | low` + причина
   - `Evidence`: путь + snippet (1-10 строк)
   - `Quick fix` и `Proper fix`
6. Сформируй итоговый отчёт строго в Markdown (без JSON/таблиц-дампов).

## Reporting Rules
- Не ограничиваться top-10: включить все найденные уникальные замечания.
- Если интеграция (например Redis/Prometheus/Kafka) отсутствует, явно это зафиксировать как observation и/или риск.
- Для high-priority добавить короткий shortlist (10-20 пунктов).
- Указывать конкретные файлы и строки.
- Пользователь может не передавать шаблон отчёта: в этом случае использовать встроенную структуру отчёта из этого скилла.
- В отчёте явно выделять нарушения 12-factor и пробелы логирования как отдельные замечания (с evidence и fix-планом).
- Для API-проектов отсутствие OpenAPI и/или `health`/`liveness` ручек фиксировать как отдельные замечания.

## Output Template
Если пользователь передал шаблон, следуй ему. Если шаблон не передан, используй эту структуру:
- Метаданные
- Итоги
- High priority shortlist
- Полный список замечаний по зонам (`db`, `efcore`, `redis`, `metrics`, `config`, `kafka`, `other`)
- Приложение/сырой вывод (опционально)
