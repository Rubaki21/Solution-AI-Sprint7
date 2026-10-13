
## 1. Формулировка

Наблюдение: через три месяца после запуска банк начал обслуживать селлеров
маркетплейсов; качество ranking снизилось. У селлеров много мелких поступлений
от площадок, возвратов и расчётов с самозанятыми

Сначала требуется определить, что именно означает `качество просело`:
- снизился `precision@K` экспертной цели `требует расследования`
- выросли снятые приостановки/жалобы
- упал yield расследований или только поздних подтверждений
- вырос override специалистов
- изменилась средняя метрика из-за доли нового сегмента, хотя старые сегменты
  не изменились

Эскалация и подтверждённая схема остаются разными outcomes. Диагностика
проводится по версии модели, policy, feature pipeline, segment и источнику
выборки

## 2. Конкурирующие гипотезы

| ID  | объяснение                                                   | наблюдение, которое его поддержит                                                                                                                             | что опровергнет ?                                                                     |
| --- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| H1  | В обучении нет селлеров: coverage/OOD gap                    | Seller slice плох, OOD/abstain и distance от train высоки; старые сегменты стабильны                                                                          | Селлеры были репрезентативны в train, OOD низок, matched slice также стабилен         |
| H2  | Сдвиг сумм, частот и связей                                  | Ошибки концентрируются на high-frequency small payments, refunds, marketplace concentration; matched по профилю старые клиенты ведут себя аналогично          | При одинаковых числовых/временных признаках seller label остаётся главным объяснением |
| H3  | Незнакомый язык назначений                                   | Рост unknown-token/embedding distance и новых шаблонов; отключение text features восстанавливает ranking                                                      | Text-only метрики стабильны, tabular ablation объясняет просадку                      |
| H4  | Легитимный профиль похож на транзит/дробление: concept shift | Эксперты согласованно называют model top-K нормальными seller flows; false positives привязаны к payout/refund/self-employed цепочкам                         | Независимая разметка подтверждает подозрительность тех же паттернов                   |
| H5  | Изменилось поведение нарушителей                             | Просадка есть и в старых сегментах; новые подтверждённые кейсы образуют кластеры вне прежних reason codes                                                     | Ошибка ограничена легитимными селлерами, старые segment outcomes стабильны            |
| H6  | Ошибка данных/feature pipeline                               | Replay одной операции даёт разные признаки; marketplace payouts задублированы, refund sign/контрагент/время неверны; деградируют обе модели на новой pipeline | Raw-to-feature reconciliation проходит, old pipeline и current pipeline эквивалентны  |
| H7  | Сменились правила, очередь или разметка                      | Queue yield просел, но вероятностная выборка стабильна; inclusion probability/гайдлайн/состав экспертов изменились                                            | Независимая blind-разметка подтверждает реальную model error                          |
| H8  | Агрегат скрывает composition effect                          | Метрики каждого старого сегмента стабильны, общая упала из-за высокой доли cold-start sellers                                                                 | Seller slice также стабилен после поправки на sampling                                |
| H9  | Capacity/latency degradation                                 | Features stale, timeouts и `rule_only` выросли при увеличении потока                                                                                          | SLO/freshness/fallback стабильны                                                      |

H1, H2 и H4 могут быть истинны одновременно: отсутствие сегмента в train
проявляется через новые распределения, а прежняя связь `быстрый оборот ->
высокий риск` перестаёт быть верной для легитимных селлеров

## 3. Дешёвый различающий эксперимент

### Выборка

Зафиксировать 600 операций из одного временного окна:

- 300 seller operations: marketplace payouts, возвраты, выплаты самозанятым,
  новые/зрелые селлеры, model top/middle/low score
- 300 old-SMB операций, matched по сумме, частоте, возрасту клиента, каналу и
  rule signals

Выборка внутри групп вероятностная; inclusion probability сохраняется. 20%
размечаются двумя специалистами вслепую, все расхождения проходят
adjudication. При 12 минутах это 120 экспертных часов; с 20% двойной разметкой
- около 144 часов или **131 тыс. ₽ прямого ФОТ** по ставке 909 ₽/ч, плюс 3–5 дней аналитика

600 наблюдений недостаточно для оценки редкого формального fraud outcome, но
достаточно как дешёвый discriminating test операционной цели и крупных
segment-specific ошибок. При неопределённом результате размер пересчитывается,
а не делается сильный вывод

### Матрица replay

На неизменных raw events получить четыре комбинации:
1. old model + old feature pipeline
2. old model + current feature pipeline
3. current model + old feature pipeline
4. current model + current feature pipeline

Все вычисления point-in-time. Сравнить feature checksum, missing/freshness,
score, reason codes и policy route. Это отделяет defect данных/кода от
свойства модели

### Срезы и ablation

1. Посчитать weighted `precision@K`, override, OOD и abstain отдельно для
   sellers и matched old-SMB
2. Повторить ranking без text features и отдельно без temporal/network
   features; модель не выкатывать - это диагностический offline replay
3. Сравнить amount/frequency/refund/counterparty distributions и расстояние до
   train по каждому сегменту
4. Провести error review по reason code: rapid turnover, concentration,
   splitting, counterparty type и text pattern
5. Сравнить очередь с постоянной вероятностной выборкой, применив inclusion
   weights. Если расходятся только queue metrics, проверить selection policy
6. Проверить старые сегменты до/после запуска селлеров на той же версии:
   исключить Simpson's paradox и capacity effects

## 4. Как различаются причины ?

| результат теста                                                          | вывод                           | действие                                                                               |
| ------------------------------------------------------------------------ | ------------------------------- | -------------------------------------------------------------------------------------- |
| Old segments stable; sellers OOD; matched profile объясняет ошибки       | H1+H2: coverage/covariate shift | Seller abstain, собрать labels, добавить segment-aware features/model                  |
| Sellers in-distribution, но эксперты отвергают прежнюю связь risk->label | H4: concept shift               | Пересмотреть target/features, отдельная калибровка или policy сегмента                 |
| Ошибка исчезает без text features                                        | H3: text shift                  | Обновить text encoder/dictionary на seller corpus; до этого text weight=0 для сегмента |
| Old model портится только на current pipeline                            | H6: pipeline defect             | Rollback feature/pipeline version, replay affected events; модель не переобучать       |
| Random sample stable, queue sample degraded                              | H7: selection/policy shift      | Исправить sampling/policy и оценку; не объявлять model drift                           |
| Old segments тоже падают, новые suspicious clusters подтверждаются       | H5: adversarial/concept drift   | Срочный expert labeling, challenger и rule update с ограниченным rollout               |
| Score/SLO нормальны, но stale/fallback выросли                           | H9: operational degradation     | Capacity/freshness fix и replay, не менять weights                                     |

## 5. Предварительный и окончательный диагноз

**Предварительный диагноз по сценарию:** segment coverage gap с
covariate/concept shift. Новые легитимные seller flows похожи на признаки
транзита и дробления, которых модель научилась опасаться на старом МСБ

**Окончательный диагноз разрешено зафиксировать только после теста.** Он
подтверждён, если:
- replay исключил H6/H9
- старые сегменты не деградировали
- seller OOD/feature shift значим относительно train
- blind experts согласованно подтверждают рост false high-priority
- matched/ablation локализуют вклад transaction profile и/или текста

Если эти условия не выполнены, нельзя `лечить` проблему добавлением seller
примеров: это может замаскировать pipeline defect или сломанный sampling

## 6. Карточка эксперимента

| поле                     | решение                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Где именно теряем ?      | Seller slice: payout/refund/self-employed flows; проверяется против старых сегментов и control sample              |
| Гипотеза причины         | Train не покрывает новый сегмент, а прежние risk patterns не разделяют легитимный seller flow и транзит            |
| Конкурирующие объяснения | H2–H9 выше                                                                                                         |
| Рычаг                    | Segment coverage, признаки, text encoder, calibration/policy, pipeline fix или sampling - только после локализации |
| Дешёвый эксперимент      | 600 blind labels + 2×2 replay + matched cohort + feature ablation                                                  |
| Ожидаемый эффект         | Установить слой причины до изменения prod и получить seller regression pack                                        |
| Стоимость проверки       | ≈131 тыс. ₽ прямого экспертного ФОТ + 3–5 дней аналитика/ML engineer                                               |
| Критерий остановки       | Причина не локализована или IAA <0,67 - расширить/исправить разметку, не выпускать новую модель                    |
