
```mermaid
flowchart LR
    planner["Person<br/><b>WFM-планировщик</b><br/>Проверяет сценарий и утверждает смены"]
    tel["Software System<br/><b>Телефония / Omnichannel</b><br/>Offered, answered, abandoned, AHT"]
    calendar["Software System<br/><b>Календарь событий</b><br/>Праздники, кампании, релизы"]
    wfm["Software System<br/><b>WFM / HR</b><br/>Навыки, смены, трудовые ограничения"]

    subgraph bank["Контур банка / одобренное private cloud"]
      subgraph forecast["Software System: Contact Center Forecasting"]
        ingest["Container<br/><b>Batch Ingestion & Data Quality</b><br/>Topology, dedup, completeness"]
        store["Container<br/><b>Time-series Feature Store</b><br/>Intervals, lags, known-future features"]
        model["Container<br/><b>Forecast Service</b><br/>Модель; median и quantiles"]
        optimizer["Container<br/><b>Staffing Optimizer</b><br/>Обычный код; SLA, Erlang, constraints"]
        api["Container<br/><b>Planning API / UI</b><br/>Сценарии, explanations, override"]
        registry["Container<br/><b>Model Registry</b><br/>Champion, baseline, signed artifact"]
        monitor["Container<br/><b>Monitoring & Backtest</b><br/>WAPE, MASE, coverage, SLA, cost"]
      end
    end

    tel -. "CB-1: operational data boundary" .-> ingest
    calendar -. "CB-1: known-future boundary" .-> ingest
    ingest --> store
    store --> model
    registry -. "CB-2: model supply chain" .-> model
    model --> optimizer
    wfm --> optimizer
    optimizer --> api
    api --> planner
    planner -. "CB-3: override/approval" .-> api
    api --> wfm
    ingest --> monitor
    model --> monitor
    optimizer --> monitor
    wfm --> monitor
```

## Режим и состав

- Batch-прогноз строится на недели вперёд после закрытия интервалов; intraday
  пересчёт запускается по расписанию или событию. Результат версионируется и
  не изменяет уже утверждённую смену без подтверждения планировщика
- Модель прогнозирует только распределение calls/workload. Перевод квантиля в
  численность и допустимое расписание выполняет детерминированный оптимизатор
- Аудио, текст разговоров и идентификаторы клиентов не нужны и не поступают в
  систему. Retrieval и LLM отсутствуют
- При недоступности модели используется последний валидный прогноз с TTL, а
  затем seasonal baseline; при сбое оптимизатора планировщик получает прогноз
  и применяет текущую процедуру WFM

Границы `CB-n` раскрыты в [таблице угроз](operations-and-security.md#границы-доверия-прогноз)
