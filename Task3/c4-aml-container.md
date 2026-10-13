
```mermaid
flowchart LR
    analyst["Person<br/><b>Специалист финмониторинга</b><br/>Проверяет кейс и фиксирует решение"]
    owner["Person<br/><b>Model/Data Risk</b><br/>Утверждает релиз и контролирует качество"]

    core["Software System<br/><b>Core Banking / Event Bus</b><br/>Операции и статусы"]
    refs["Software System<br/><b>KYC и справочники банка</b><br/>Клиент, ОКВЭД, лимиты"]
    ext["External Software System<br/><b>Внешний KYC-сигнал</b><br/>Вспомогательный уровень риска"]
    cases["Software System<br/><b>Case Management</b><br/>Рабочее место и исходы расследований"]

    subgraph bank["Контур банка"]
      subgraph aml["Software System: AML Prioritization"]
        ingest["Container<br/><b>Event Ingestion & Validation</b><br/>Обычный код; schema, idempotency, DLQ"]
        features["Container<br/><b>Feature Pipeline / Store</b><br/>Point-in-time признаки и freshness"]
        scoring["Container<br/><b>Scoring Service</b><br/>Модель, OOD, калибровка, reason codes"]
        policy["Container<br/><b>Policy & Queue Orchestrator</b><br/>Правила, top-K, abstain, контрольная выборка"]
        queue["Container<br/><b>Review Queue</b><br/>Приоритеты, SLA, retries"]
        feedback["Container<br/><b>Outcome Collector</b><br/>Решения, причины, поздние исходы"]
        train["Container<br/><b>Training Pipeline</b><br/>Offline build, backtest, champion/challenger"]
        registry["Container<br/><b>Internal Model Registry</b><br/>Подписанные модели, schema, lineage"]
        monitor["Container<br/><b>Monitoring & Audit</b><br/>SLO, drift, queue, business guardrails"]
      end
    end

    core -. "TB-1: события" .-> ingest
    refs -. "TB-1: внутренние источники" .-> features
    ext -. "TB-2: внешний сигнал" .-> ingest
    ingest --> features
    features --> scoring
    scoring --> policy
    ingest --> policy
    policy --> queue
    queue -. "TB-4: рекомендация в процесс" .-> cases
    cases --> analyst
    analyst -. "TB-5: решение человека" .-> cases
    cases --> feedback
    feedback --> train
    features --> train
    train -. "TB-3: артефакт модели" .-> registry
    registry -. "TB-3: подписанный артефакт" .-> scoring
    ingest --> monitor
    scoring --> monitor
    queue --> monitor
    feedback --> monitor
    monitor --> owner
    owner --> registry
```

## Режим

- Скоринг асинхронный, event-driven, с семантикой at-least-once
- Идемпотентность: `(operation_id, event_version, policy_version)`
- Пилот обрабатывает операции текущей очереди; сервис не находится в
  синхронном пути проведения платежа
- При временном сбое выполняются ограниченные повторы с backoff, затем событие
  попадает в DLQ и очередь строится по действующим правилам
- Batch используется для тяжёлых агрегатов, обучения и переоценки качества;
  он не заменяет near-real-time постановку кейса

Пунктирные связи `TB-n` - границы доверия, раскрытые в [таблице угроз](operations-and-security.md#границы-доверия-aml)
