
## 1. Версионируемая единица релиза

Релизом считается атомарный bundle:

`raw schema + feature code/schema + segment mapping + text encoder + model +
calibration + reason-code mapping + policy + output schema`

Нельзя откатить только weights, оставив новую семантику refund sign,
counterparty mapping или policy. Каждый bundle подписан, имеет lineage и
совместимую предыдущую версию

В текущей архитектуре нет prompt/LLM. Поэтому prompt regression не добавляется
формально. Если text-звено позже заменят LLM/prompt, prompt и retrieval corpus
войдут в тот же bundle и пройдут набор ниже плюс тесты prompt injection,
source grounding и schema validity

## 2. Регрессионный набор

| pack              | состав                                                                                                | что защищает ?                             |
| ----------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Core immutable    | Стабильный временной holdout старого МСБ, разные суммы, каналы, ОКВЭД и rule/no-rule                  | Не сломать прежние сегменты                |
| Seller legitimate | Marketplace payouts, регулярные мелкие поступления, refunds, self-employed settlements, сезонные пики | False positives нового легитимного профиля |
| Seller suspicious | Подтверждённые экспертами необычные отклонения внутри seller profile                                  | Не «выключить» риск для всего сегмента     |
| Cold-start/OOD    | Новый клиент, неизвестный marketplace, короткая история, unseen category/text                         | Правильное воздержание и fallback          |
| Rule conflict     | High rule + low model, low rule + high model, внешний сигнал missing/stale                            | Mandatory rules и ручной conflict route    |
| Data quality      | Missing/stale, duplicates, cancellation, refund sign, time zone, changed counterparty mapping         | Feature/pipeline defects                   |
| Text variants     | Новые назначения, сокращения, транслит, опечатки, шаблоны разных площадок                             | Text encoder drift                         |
| Temporal/leakage  | Events на границе окна, поздние outcomes, backfill и изменённый справочник                            | Point-in-time корректность                 |
| Contract/security | NaN/out-of-range, unknown reason, invalid source, replay, oversized text                              | Output validation и безопасный отказ       |
| Operational       | Peak throughput, feature/model timeout, registry unavailable, queue backpressure                      | SLO, circuit breaker, DLQ и baseline       |

Каждый пример хранит source/time/segment/guideline/label lineage. Regression
set не используется для обучения. Seller pack пополняется из вероятностной
выборки и ошибок canary после adjudication; старые версии остаются immutable

## 3. Gates перед релизом

### Data/contract

- schema compatibility, point-in-time and replay checksum tests проходят
- ни один mandatory feature не изменил семантику без новой major version
- seller coverage и sample weights документированы
- output contract и fallback tests проходят на 100% regression examples

### Offline quality

- на core lower 95% CI изменения `precision@K` не хуже утверждённой
  non-inferiority margin
- на seller pack новый challenger статистически лучше текущего
  seller fallback/champion и не достигает успеха только снижением coverage
- OOD и abstain измеряются вместе с quality: нельзя «улучшить» precision,
  отправив почти всех человеку
- reason-code review не выявляет запрещённый proxy или нелогичную зависимость
- calibration и ranking считаются отдельно по core/seller/cold-start

### Operations/security

- нагрузочный тест выдерживает peak с запасом, fallback и replay проверены
- подписаны SBOM, model card, data sheet, threat-model delta и rollback plan
- Model Risk, финмониторинг, Data Owner и ИБ одобрили bundle

## 4. Схема rollout

1. **Offline replay:** old/new bundles на одном immutable наборе и raw replay
2. **Shadow 100% seller traffic, минимум 2 недели:** challenger не влияет на
   очередь; blind sample получает outcomes
3. **Canary 5% seller-eligible queue:** model route меняет только порядок;
   mandatory rules и human decision сохраняются
4. **Canary 10% -> 25% -> 50%:** переход только после минимального числа
   экспертных outcomes, а не по календарю
5. **Seller champion:** отдельный feature flag/route. Старые сегменты остаются
   на прежнем champion
6. **Объединение моделей** рассматривается позднее, если общий вариант не хуже
   segment-specific по качеству, TCO и поддержке

На каждом шаге сравнение выполняется с одновременным control bucket. Состав
traffic, K и экспертная мощность фиксируются, чтобы изменение очереди не
маскировалось capacity

## 5. Автоматические и ручные триггеры rollback

### Автоматический routing rollback

- invalid output, signature/schema mismatch
- critical feature stale/missing или OOD вне допустимой области
- p99 latency/timeout/fallback нарушают утверждённый SLO
- mandatory rule conflict обработан не по policy
- critical security alert

Трафик seller slice атомарно переключается на `rules + manual + exploration`.
Платёжный процесс не останавливается

### Rollback по качеству

Решение принимает on-call Model Risk/финмониторинг, если:
- нижняя 95% граница seller uplift ≤0 после достижения pre-registered n
- core или критичный seller slice хуже non-inferiority margin
- false-high-priority, снятые приостановки или жалобы нарушили guardrail
- abstain/coverage существенно отклонились от одобренной рабочей точки
- random control и queue metrics показывают selection feedback loop

Point estimate клиентского вреда запускает немедленный review даже до
статистической мощности

## 6. Технический rollback

1. Release Controller переключает один bundle pointer на предыдущую подписанную
   версию; policy и calibration откатываются вместе с моделью/features
2. Новые seller events направляются на предыдущий champion либо
   `rule_only/manual`; уже принятые человеком решения не переписываются
3. Необработанные idempotent events replay из журнала; дубли не создают новый
   кейс
4. Feature writes новой версии сохраняются для forensic analysis, но не
   читаются старым bundle
5. Incident snapshot фиксирует версии, metrics, sample IDs и audit trail
6. После root-cause и regression fix начинается новый shadow; перепрыгнуть
   сразу в canary нельзя

Ежеквартальный game day проверяет переключение champion->baseline, восстановление
registry, replay DLQ и доступность предыдущего bundle

## 7. Владельцы решения

| решение                         | владелец                    | участники             |
| ------------------------------- | --------------------------- | --------------------- |
| Segment definition/change event | Product/Data Owner          | CRM, Data Platform    |
| Guideline и label adjudication  | Руководитель финмониторинга | Эксперты, Model Risk  |
| Data/feature bundle             | Data Owner                  | Data Engineering, ИБ  |
| Model release                   | Model Risk                  | ML, финмониторинг, ИБ |
| Client guardrail stop           | Финмониторинг/Compliance    | Operations, Product   |
| Security rollback               | ИБ on-call                  | Platform, Model Risk  |
| Возврат после rollback          | Change authority            | Все владельцы gates   |

## 8. Критерий закрытия инцидента

Инцидент закрывается не после переобучения, а когда:
- root cause подтверждён различающим тестом и воспроизводится replay
- seller route имеет валидный regression pack и контрольную выборку
- shadow/canary проходят quality, coverage, queue и client guardrails
- старые сегменты не деградировали
- rollback/game-day успешен
- владельцы приняли residual risk и обновили model/data cards
