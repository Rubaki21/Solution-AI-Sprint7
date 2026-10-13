

```mermaid
flowchart LR
    applicant["Person<br/><b>Клиент / сотрудник</b><br/>Загружает пакет"]
    underwriter["Person<br/><b>Андеррайтер</b><br/>Подтверждает и исправляет поля"]
    portal["Software System<br/><b>Кредитный портал / DMS</b>"]
    loan["Software System<br/><b>Кредитный конвейер</b>"]

    subgraph bank["Одобренный контур обработки"]
      subgraph docai["Software System: Document Digitization"]
        quarantine["Container<br/><b>Upload Quarantine</b><br/>AV, MIME, limits, CDR"]
        preprocess["Container<br/><b>Preprocessor</b><br/>PDF split, rotate, image quality"]
        schema["Container<br/><b>Document Schema Registry</b><br/>Типы, поля, версии, правила"]
        extract["Container<br/><b>OCR / Layout Extraction</b><br/>Модель; text, tables, fields, confidence"]
        validate["Container<br/><b>Validation & Routing</b><br/>JSON Schema, formats, arithmetic, conflicts"]
        review["Container<br/><b>Review UI</b><br/>Источник, координаты, low-confidence queue"]
        result["Container<br/><b>Result API</b><br/>Idempotent versioned write"]
        feedback["Container<br/><b>Correction & Audit Store</b><br/>old/new, reason, lineage"]
        registry["Container<br/><b>Internal Model Registry</b><br/>Подписанные веса и dependencies"]
        monitor["Container<br/><b>Monitoring</b><br/>quality, latency, field drift"]
      end
    end

    applicant -. "DB-1: недоверенный файл" .-> portal
    portal -. "DB-1: недоверенный контент" .-> quarantine
    quarantine --> preprocess
    preprocess --> extract
    schema --> extract
    registry -. "DB-2: модель и зависимости" .-> extract
    extract --> validate
    schema --> validate
    validate --> review
    review --> underwriter
    underwriter -. "DB-4: подтверждение/исправление" .-> review
    review --> result
    result -. "DB-3: валидированные поля" .-> loan
    review --> feedback
    quarantine --> monitor
    extract --> monitor
    validate --> monitor
    feedback --> monitor
```

## Режим и состав

- Обработка асинхронная: пакет получает `job_id`, состояние и прогресс
  Большие документы не удерживают HTTP-соединение; шаги идемпотентны по
  хэшу файла, версии схемы и модели
- Повторы ограничены по типу ошибки; повреждённый/зашифрованный файл,
  превышение лимита и исчерпание retry направляются в ручной путь
- Контекст модели - только очищенные страницы и версия схемы документа.
  Подтверждающий источник - координаты на конкретной странице оригинала
- Инструменты - детерминированные валидаторы формата, арифметики и справочников.
  Свободного agent/tool execution и внешнего retrieval нет
- Модель извлекает; обычный код проверяет и маршрутизирует; андеррайтер
  подтверждает критичные поля. Неподтверждённые значения не участвуют в
  кредитном решении

Границы `DB-n` раскрыты в [таблице угроз](operations-and-security.md#границы-доверия-документы)
