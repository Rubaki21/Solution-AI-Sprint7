
## AML: выходной контракт

Модель не публикует результат напрямую. `Output Contract Validator` и
`Prioritization Policy` формируют следующий версионированный объект:

```json
{
  "schema_version": "1.0",
  "operation_id": "opaque-id",
  "event_version": 3,
  "scored_at": "2026-10-13T15:00:00Z",
  "expires_at": "2026-10-13T15:15:00Z",
  "trace_id": "uuid",
  "model": {
    "name": "aml-ranking",
    "version": "signed-model-version",
    "feature_set_version": "point-in-time-v1",
    "scope": "flagged-operations"
  },
  "recommendation": "prioritize",
  "score": 0.73,
  "score_band": "review",
  "rank_bucket": "top-k",
  "reason_codes": ["rapid_turnover", "counterparty_concentration"],
  "rule_signals": ["rule-id:version"],
  "source_refs": [
    {"type": "operation", "id": "opaque-id", "as_of": "timestamp"}
  ],
  "data_quality": {
    "status": "ok",
    "missing": [],
    "stale": [],
    "cold_start": false,
    "ood": false
  },
  "abstention": {"abstained": false, "reasons": []}
}
```

### Инварианты и валидация

- `recommendation` ∈ `{prioritize, standard, rule_only, manual_required, abstain}` значения `fraud`, `block` и `reject` схемой не допускаются
- `score` либо в `[0,1]`, либо `null` при `rule_only/manual_required/abstain`
- `source_refs`, версии, timestamps, data-quality и trace обязательны;
  reason code берётся только из утверждённого справочника
- Ссылка на источник - непрозрачный ID и `as_of`, а не копия чувствительного
  payload. Case Management получает данные по собственным правам
- Просроченный результат, неизвестная версия, невалидная схема, NaN/Infinity,
  отсутствующий источник или несовместимая версия признаков блокируют model
  route до публикации
- Обязательные правила исполняются независимо. Конфликт модели и правила не
  разрешается моделью: кейс получает `manual_required` и обе причины
- Даже валидный контракт создаёт только элемент очереди. Любое действие,
  затрагивающее клиента, проходит текущие проверки и подтверждение сотрудника

## AML: fallback

| наблюдаемый признак                                          | маршрут                                                                                                     | возврат в основной путь                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Пропущен обязательный или устарел критичный признак          | `manual_required` либо `rule_only`; модель не вызывается                                                    | После восстановления freshness и replay события         |
| OOD выше утверждённой границы                                | Воздержание, ручная очередь и попадание в exploration sample                                                | После экспертной оценки и проверки новой области        |
| Новый клиент/сегмент без истории                             | Cold-start score не используется; правила + отдельная ручная квота                                          | После минимальной истории либо одобренной pooled-модели |
| Score в зоне воздержания около текущего capacity cutoff      | Стандартная ручная очередь; не повышать и не понижать только по score                                       | При изменении K/калибровки по утверждённой политике     |
| Модель расходится с обязательным/сильным внешним сигналом    | `manual_required`, показать оба источника                                                                   | Только решением специалиста; конфликт идёт в мониторинг |
| Невалидный контракт или нет source reference                 | Результат отбрасывается, baseline route, security event при повторении                                      | После исправления версии/schema                         |
| Model/Feature Service недоступен или превысил latency budget | Circuit breaker -> действующие правила и приоритет суммы; событие в retry/DLQ                                | После health check и контролируемого replay             |
| Внешний KYC недоступен                                       | Использовать последнее допустимое значение до TTL либо признак `unknown`; не считать низким риском          | После восстановления; переоценка ещё актуальных кейсов  |
| Очередь достигла лимита                                      | Backpressure; обязательные rules проходят, model-only кейсы сохраняются без клиента-затрагивающего действия | После снижения depth; порядок по SLA/policy             |

Fallback baseline должен регулярно тестироваться. «Ошибка модели» не должна
останавливать проведение платежей или отключать обязательные правила

## AML: мониторинг и откат

| слой             | что измеряется ?                                                                                                         | реакция                                                                                                           |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Вход             | Schema/version rejects, lag, duplicate rate, missing/stale, диапазоны, доля cold-start, распределение сегментов и текста | Несовместимая схема/критичная freshness -> circuit breaker и baseline; drift -> review, не немедленное переобучение |
| Признаки         | Offline/online parity, point-in-time tests, feature availability, PSI/дистанции по утверждённым полям                    | Нарушение parity/утечка -> остановка версии и расследование                                                        |
| Сервис модели    | Availability, p50/p95/p99 latency, error/timeout, score/reason distribution, OOD/abstain, resource saturation            | SLO breach -> canary stop/rollback; рост OOD -> ручная квота и review области                                       |
| Политика/очередь | Depth, oldest age, доля top-K/rule/control sample, retries/DLQ, конфликты, K относительно мощности                       | Нарушена контрольная квота или SLA -> заморозка релиза, capacity fallback                                          |
| Человек          | Handling time, override/disagreement, согласованность, причины, нагрузка по специалистам                                 | Рост расхождений -> проверка UI, guideline, калибровки; не считать override ошибкой человека автоматически         |
| Качество         | Weighted `precision@K`, uplift и CI, yield по сегментам, delayed outcomes, calibration                                   | Нижняя 95% граница uplift ≤0 после достаточного набора либо сегмент хуже margin -> champion rollback               |
| Бизнес           | Эскалации, обоснованные/снятые приостановки, жалобы, отток, стоимость кейса, предписания                                 | Guardrail хуже утверждённой границы -> остановка расширения и совместный review комплаенса/риска                   |

Каждый релиз идёт shadow -> canary с ограниченной долей очереди -> champion.
Rollback меняет только policy/model version; события повторно обрабатываются
идемпотентно. Пороговые значения SLO, drift и business guardrails утверждаются
владельцами до запуска и хранятся как versioned policy, а не зашиваются в код

## Документы: контракт, fallback и мониторинг

Контракт результата содержит `job_id`, hash/версию файла, тип и версию схемы,
по каждому полю - исходное и нормализованное значение, confidence,
page/bounding box, validation results и статус подтверждения. Допустимые
статусы: `extracted_unverified`, `confirmed`, `corrected`, `manual_required`,
`unsupported`. Только `confirmed/corrected` передаются в кредитный конвейер

Fallback:
- не прошли AV/MIME/размер/декомпрессия -> quarantine/reject без model call
- JSON Schema не прошла, нет page/bbox, поле не подтверждается источником,
  документы противоречат друг другу -> андеррайтер
- неизвестный тип, плохое качество, confidence ниже полевого порога или
  запрос вне пилотного scope -> полный ручной ввод
- timeout/недоступность -> retry с лимитом, затем `manual_required`
- превышение страниц/пикселей/tokens -> безопасное разбиение либо ручной путь,
  но не усечение без отметки

Мониторинг: доля quarantined/unsupported, качество изображения, drift
типов/форм, latency и DLQ, exact match/correction rate по каждому полю и
сценарию, false-high-confidence, время проверки, возврат в ручной ввод и
ошибки после подтверждения. Любая галлюцинация отсутствующего критичного поля
с высокой уверенностью блокирует автопринятие и запускает review версии

## Прогноз: контракт, fallback и мониторинг

Контракт содержит очередь/канал, интервал, `generated_at`, horizon,
data-cutoff, model/data versions, p10/p50/p80/p90 (с монотонностью квантилей),
ожидаемый workload, interval coverage target, OOD/data-quality flags и список
известных событий. Оптимизатор принимает только непросроченный прогноз с
полными интервалами и валидной topology

Fallback: при пропуске/смене topology - ручное сопоставление; при OOD/сбое
модели - последний валидный прогноз до TTL, затем seasonal naive; при
недоступности календаря - baseline без кампании с явным предупреждением; при
сбое оптимизатора - квантили модели и текущая ручная процедура WFM

Мониторинг покрывает freshness и полноту рядов, WAPE/MASE/pinball/coverage,
ошибку по пикам и очередям, model latency, overrides, изменение смен после
рекомендации, SLA, abandon rate, overtime и стоимость. Хороший forecast при
плохом SLA направляет расследование в AHT/shrinkage/optimizer, а не на
автоматическое переобучение модели

## Границы доверия: AML

| граница                                  | угроза                                                                                             | что делаем ?                                                                                                                                                                       |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TB-1 - Core/KYC -> ingestion/features    | Подмена/replay, несовместимая schema, лишние ПДн, ошибочный timestamp                              | mTLS/service identity, allowlist topics, schema registry, signature/checksum, idempotency, data minimization, point-in-time validation, quarantine                                 |
| TB-2 - внешний KYC -> банк               | Компрометация/poisoning сигнала, отказ, изменение семантики; утечка при обратном запросе           | Односторонний минимизированный adapter, договор/region review, mTLS, versioned mapping, TTL, rate limit, signal treated as auxiliary, fallback `unknown`, no final decision        |
| TB-3 - build/registry -> runtime         | Poisoned weights/dependencies, artifact tampering, уязвимый pickle/container, неразрешённая модель | Внутренний registry, SBOM, malware/CVE/license scan, reproducible build, hash/signature verification, immutable artifact, isolated non-root runtime, approval and provenance       |
| TB-4 - scoring/policy -> Case Management | Невалидный/инъекционный текст, превышение полномочий, прямой вызов блокировки, повтор кейса        | Строгая JSON Schema и enum reason codes, output encoding, least-privilege API только `create/update case`, idempotency, no payment-action credential, human confirmation and audit |
| TB-5 - analyst -> feedback               | Захват учётной записи, insider misuse, label poisoning, массовая выгрузка данных                   | MFA/RBAC/SoD, case-scoped access, immutable audit, rate/export controls, dual review/adjudication, anomaly monitoring; feedback не попадает в обучение без validation              |

## Границы доверия: документы

| граница                             | угроза                                                                                               | что делаем ?                                                                                                                                                         |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DB-1 - загрузка -> DMS/quarantine   | Malware, polyglot, zip/decompression bomb, parser exploit, prompt injection/скрытый текст, DoS       | Quarantine, AV/CDR, magic bytes, sandboxed parsers, page/pixel/archive limits, no macros/network, content is data not instruction, OCR/model has no tools or secrets |
| DB-2 - registry -> extractor        | Подменённые веса, backdoor, уязвимые зависимости/модельный формат                                    | Подпись/hash, SBOM/CVE/license scan, provenance, safe serialization, isolated runtime, benchmark на adversarial/poisoned samples                                     |
| DB-3 - result -> кредитный конвейер | Галлюцинация, неверная schema/единица, источник отсутствует, неподтверждённое поле влияет на решение | JSON Schema, field allowlist, units/arithmetic checks, mandatory page/bbox, status gate `confirmed/corrected`, least privilege, immutable lineage                    |
| DB-4 - андеррайтер -> corrections   | Ошибка/злоупотребление, poisoning будущего обучения, раскрытие чужого документа                      | MFA/RBAC, application-scoped access, old/new/reason audit, dual label gold-set, separation of training approval, screen/export controls                              |

## Границы доверия: прогноз

| граница                                 | угроза                                                                                                    | что делаем ?                                                                                                                                  |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| CB-1 - operational systems -> analytics | Неполные offered calls, смена topology, поздние исправления, лишние client/employee IDs, подмена кампаний | Data contracts, cutoff/version, reconciliation, topology history, known-at-time checks, aggregation/minimization, RBAC and quality quarantine |
| CB-2 - registry -> forecast runtime     | Supply-chain compromise и неутверждённый champion                                                         | Те же provenance/signature/SBOM controls, approval gate, immutable champion и tested seasonal fallback                                        |
| CB-3 - planner/UI -> WFM                | Несанкционированный override, нарушение трудовых ограничений, автоматическая публикация ошибочной смены   | MFA/RBAC/SoD, optimizer revalidation of hard constraints, explicit approval, versioned diff/audit, rollback to previous schedule              |
