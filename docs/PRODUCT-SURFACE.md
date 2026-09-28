# > **Bundle routing note (2026-08-25):** this master specification is preserved

> **Текущая редакция документа: r15.11, 28 сентября 2026.** Добавлен ранний external-data scope; исходные исторические срезы ниже датированы отдельно. Изменение спецификации не является выполнением runtime/источниковой приёмки.

> **Основа и reuse текущей редакции:** [локальная карта внешних компонентов и адаптеров](DEVELOPMENT.md#implementation-basis). Там указаны источник, режим использования, наши модули/пути, задачи и ограничения. Кандидат не установленная зависимость; этот предметный документ не требует реализации библиотечной механики с нуля.


> Текущий пакет r15.11 · 28.09.2026. Независимая версия предметной спецификации и датированное происхождение указаны ниже. Нормативные gates — локальный CONTEXT; старые source measurements не текущие замеры.

**Уточнение scope Research: r12.1 · 2026-09-18.** Пользовательские примеры не определяют встроенную стратегию или обязательный dataset; статус реализации не меняется.

**Редакция:** 1.1.1 · **Дата:** 2026-09-28 · **Статус:** PROPOSED / implementation status requires evidence.

## 3. Полный каталог стратегий и конструктор

### 3.1 Каталог — семейства конструкций, а не закрытый список сделок

| Семейство | Основные конструкции | Что обязательно рассчитывать | Особые ограничения |
|---|---|---|---|
| Направленный рост | Long call, bull call debit spread, bull put credit spread | Премия/кредит, breakeven, payoff, Greeks, максимальный убыток при корректной модели | Credit spread требует обеспечения и защиты от разрыва ног |
| Направленное падение | Long put, bear put debit spread, bear call credit spread | Те же величины, риск и доступность выхода | Long put не страхует автоматически весь портфель |
| Защита | Protective put, put spread hedge, collar | Отдельно риск хеджа и остаточный риск базового портфеля | Collar ограничивает upside; put spread покрывает только участок падения |
| Большое движение | Long straddle, long strangle | Два breakeven, премия, theta/vega, сценарии движения и IV | Угаданное направление не гарантирует окупаемость премии |
| Диапазон | Long butterfly, condor, iron butterfly, iron condor | Экстремумы кусочного payoff, совместные fees, обеспечение коротких ног | Нет обещания стабильного дохода; проверка пакетного исполнения |
| Покрытая продажа | Covered call, cash-secured put | Риск всего пакета с underlying/cash, упущенный upside, assignment | Covered call сохраняет риск падения underlying; collateral/settlement должны соответствовать |
| Сроки и относительная IV | Calendar, diagonal, double calendar | Стоимость во времени, поверхности по срокам, ранняя экспирация, margin path | Одной payoff-кривой на общий expiry их не описать |
| Skew и относительная стоимость | Risk reversal, ratio/backspread, broken-wing butterfly | Неограниченные хвосты, ratios, collateral, режимы IV | Некоторые варианты имеют неограниченный убыток; исследование допускается, retail live по умолчанию нет |
| Динамическая волатильность | Delta-hedged options, gamma scalping, volatility carry | Путь хеджа, turnover, funding, fees, latency, inventory | Нужны частые котировки и реалистичный hedge execution; не эквивалент статическому payoff |
| Относительная стоимость между рынками | Basis/volatility spreads, cross-venue combinations | Различия индексов, валют, экспираций, расчётов, funding и доступности капитала | Межплощадочная атомарность не предполагается |
| Событийные/сигнальные | Пользовательские событийные и многофакторные правила, order-flow-triggered конструкции, event hedges | Точное правило события, задержка, выбор инструмента, вход/выход, статистика | Сигнал не подменяет цену опциона или модель исполнения |
| Собственные | Произвольная разрешённая комбинация перечисленных primitives и пользовательских правил | Семантика каждой ноги, временной логики, риска, данных и исполнения | Неподдерживаемая модель возвращает явный отказ, не приблизительное «всё проверено» |

В первом вычислительном релизе покрываются long call/put, четыре vertical spread, protective put, collar, straddle/strangle и стандартные butterfly/condor при единой поддержанной спецификации. Calendar/diagonal и динамические стратегии имеют отдельные valuation/execution модели; их карточки и требования видны в каталоге до допуска к расчёту. Это ограничение зрелости конкретного движка, а не исключение класса из продукта.

### 3.2 Независимые статусы доступности

Для каждой стратегии и версии данных система показывает `definitionSupported`, `valuationSupported`, `historicalTestEligible`, `paperEligible`, `liveEligible`, а также причины отказа. Наличие шаблона в каталоге не даёт пяти разрешений сразу.

- **SYNTHETIC:** проверяется арифметика или UX на созданных данных.
- **MODELLED:** сценарий использует модельные значения; это не наблюдавшиеся сделки.
- **HISTORICAL:** фиксированные реальные данные с раскрытой моделью заполнения.
- **PAPER:** наблюдение и симулированные действия в отдельной среде.
- **LIVE:** фактические операции на конкретном разрешённом маршруте.

Статусы не смешиваются в графике доходности. Даже реалистичный HISTORICAL-бэктест не является историей реальных fills.

### 3.3 Три входа в один конструктор

**Намерение:** пользователь описывает цель, горизонт, актив и ограничения. AI уточняет недостающие параметры и формирует draft. **Шаблон:** пользователь выбирает семейство и меняет параметры. **Профессиональный builder:** редактирует legs, selectors, triggers, sizing и exits напрямую. Все три пути компилируются в один typed `StrategySpec`.

Конструктор состоит из независимых частей: observation universe, signal/features, instrument selector, leg composition, position sizing, entry, exit/roll, execution policy, portfolio constraints. Условия — переиспользуемые predicates с units, параметрами, versioned dependency plan и требованиями данных. Entry и exit редактируются независимо; ни формула, ни порог, ни источник из примера не зашиваются в core.

Пользователь может подключить свои лицензированные данные. Для ряда нужны источник, временная семантика, units, schema, known gaps и права использования. Загруженный CSV без времени доступности не становится PIT-доказательством. Пользовательский код выполняется в изолированном исследовательском runtime с лимитами CPU/RAM/времени, без торговых ключей и с запрещённой сетью по умолчанию. Версия кода и dependencies входят в manifest.

## 4. От собственной идеи до работающей стратегии

### 4.1 Предметные состояния

`IDEA → SPEC_DRAFT → DATA_CHECKED → EXPERIMENT_REGISTERED → TESTED → VALIDATED_VERSION → PAPER_DEPLOYED → LIVE_ELIGIBLE → LIVE_DEPLOYED → PAUSED/RETIRED`.

Переход не обязан завершиться продвижением: `INSUFFICIENT_DATA`, `REJECTED`, `INCONCLUSIVE` и `RESEARCH_ONLY` — полноценные результаты. Live может быть недоступен даже для полезного исследования.

| Шаг | Что делает AI | Что определяет результат | Что сохраняется |
|---|---|---|---|
| Формализация | Выявляет неоднозначности; предлагает проверяемые правила | Пользователь подтверждает экономический смысл; schema validator проверяет полноту | Hypothesis, StrategySpec, список допущений |
| Поиск существующего | Ищет аналогичные templates/features/experiments | Версии и семантика библиотек | Reuse plan, а не копия нового модуля |
| Проверка данных | Вызывает capability query, объясняет пробелы | MKT-DS/DQ/TQ, права и PIT coverage | DataRequirement/CapabilityReport |
| План эксперимента | Предлагает baselines, splits, costs и sensitivities | Заранее фиксированный ExperimentSpec | Hash спецификации и budget поиска |
| Бэктест | Запускает задачу, отслеживает прогресс | Детерминированный replay и simulator | ExperimentResult, trade ledger, skips, provenance |
| Критика | Ищет leakage, переоптимизацию и неверные интерпретации | Автоматические проверки + независимый review | ValidationReport и ограничения |
| Paper | Помогает настроить наблюдение и лимиты | Отдельный PAPER runtime, реальные текущие данные, simulated fills | DeploymentManifest, исполненные/пропущенные события |
| Live | Готовит точную версию и объясняет разрешения | Eligibility, risk, mandate, human approval, route readiness | Immutable version + policy + audit |
| Сопровождение | Объясняет расхождения, предлагает исследовать изменения | Телеметрия, позиции, риск, versioned drift policy | DriftReport, pause или новый experiment |

### 4.2 Контракт стратегии

`StrategySpec` содержит `strategyId/version`, автора и tenant, цель, базовую валюту учёта, полный перечень источников, event-time/available-time policy, формулы и warmup, selectors, legs/ratios, sizing, entry schedule, exits, position overlap, costs, order type, partial-fill policy, expiry behavior, missing-data policy, risk limits, code/config digests и библиографию. Draft допускает нерешённые вопросы; runnable spec — нет.

`ExperimentSpec` отдельно фиксирует universe, период, обучающие и контрольные окна, embargo/purge для перекрывающихся labels, random seeds, варианты, критерии успеха и отказа, вычислительный бюджет. В ledger записываются все trials, включая неудачные, interrupted и отвергнутые. Повторный поиск после просмотра holdout создаёт новую исследовательскую итерацию с новым holdout; старый нельзя вновь назвать независимым.

`ValidatedStrategyVersion` связывает **конкретный** spec/code/data/engine и validation report. Это не награда стратегии навсегда. Доказательство процедуры, статистическая пригодность, разрешение торговли и фактическая прибыль — четыре отдельных утверждения.

### 4.3 Бэктест, который полезно сравнивать с реальностью

Один state/feature/strategy engine используется в replay и online; меняются source/clock/execution adapters. Историческая проверка включает реально существовавшие инструменты, делистинги, spreads, fees, lot/tick size, валюты, задержку и отсутствие котировки. Mid, mark и доступный bid/ask не взаимозаменяемы.

Обязательные результаты: число независимых сигналов и сделок, экспозиция и время в позиции, turnover, чистые денежные потоки, drawdown, хвостовые потери, стоимость исполнения, sensitivity, coverage, причины пропусков, результаты по режимам и интервал неопределённости. Нереализованный P&L на mark и liquidatable value показываются отдельно.

Baseline выбирается по гипотезе: cash/no trade, underlying exposure сопоставимого риска, та же опционная конструкция без сигнала, тот же сигнал без дополнительного фильтра. Сравнивать только с убыточным случайным вариантом недостаточно. Sharpe не вычисляется из редких сделок так, будто они независимые ежедневные наблюдения.

### 4.4 Продвижение и остановка

Для PAPER нужны replay/recovery, отрицательные сценарии, immutable policy и явная модель fills. Для live дополнительно: доступность инструментов, права клиента/площадки, экономические лимиты, защита от повторной отправки и сверка состояния. Новая версия модели сигнала, формулы, источника или выхода требует нового versioned review. AI не может «подправить» работающую стратегию в фоне.

Остановка стратегии прекращает новые sends и инициирует разрешённые отмены. Открытые позиции и незавершённые заявки продолжают учитываться. `run.cancel` отменяет аналитическую работу; `deployment.pause` меняет торговую политику; `order.cancel` имеет собственный жизненный цикл. Эти команды не заменяют друг друга.

## 7. Продуктовые поверхности и непрерывность работы

### 7.1 Один продукт и несколько способов работы

| Поверхность | Для чего нужна | Что остаётся общим |
|---|---|---|
| Connect: MCP/API/SDK/CLI | Пользователь работает в собственном агенте, notebook или приложении | Objects, данные, compute, hosted Skills, permissions, usage |
| Options Desk / начальный Market Workspace | Быстро исследовать выбранный рынок, собрать стратегию и разобрать результат | Scene/selection, Evidence, Thesis, StrategySpec, Experiments |
| Browser Workstation | Ежедневная многооконная работа, совместные исследования и глубокая визуализация | Та же предметная модель и серверные capabilities |
| Desktop Workstation | Несколько окон, hotkeys, локальный cache и desktop integration | Общая web-native клиентская основа и renderer contracts; wrapper выбирается проверкой |
| Pi Market Analyst | Тонкий агентный клиент и пользовательские локальные workflows | Hosted-вызовы private intelligence; открытые tools и личные настройки локально |
| Mobile companion | Просмотр разрешённых share-карточек, существенных уведомлений и follow-up | Стабильные object IDs и deep links; мобильный терминал не обязательный первый релиз |

Private Skills, рубрики Critic и eval corpus не распространяются вместе с клиентом. Одна и та же предметная операция из кнопки, команды, внешнего агента или AI-панели проходит одинаковые серверные проверки.

### 7.2 Что означает «Cursor для трейдера»

| Примитив | Реализация Odelix |
|---|---|
| Контекст открытого проекта | Market State, Workspace/Scene, активные Thesis, стратегии, позиции и разрешённая память |
| Выделение фрагмента | `SelectionRef`: instrument, time/price bounds, layers, object revisions, live/replay cursor и asOf |
| Ask | Короткий Fast Ask или ограниченный Deep Investigation с инструментами и проверкой evidence |
| Edit | Typed ChangeSet: изменить annotations/layout, создать Thesis/Watch, предложить новую версию стратегии |
| Diff и применение | Preview → Apply / Edit / Reject; exact Undo для обратимых изменений среды |
| Проверка | Replay, численные tests, experiment validation, Critic и воспроизводимый trace |
| Продолжение работы | Адресуемые objects, сохранённые layouts, Journal, Missions и Decision Memory |
| Расширения | Опубликованные contracts, SDK/MCP, templates, разрешённые predicates и Skills |

AI действует над структурированными объектами среды. Screenshot может помогать с расположением элементов, но цены, временные границы и риск он получает из typed tools. Undo интерфейса не отменяет финансовую сделку; cancel/close являются отдельными торговыми командами с собственными последствиями.

### 7.3 Workspace, View, Pane, Lens и команды

**Workspace** сохраняет рабочую задачу, bindings, открытые objects, focus и layout. **View** представляет объект или market scope. **Pane** задаёт геометрию и не владеет копией рыночной истины. **Layout Preset** меняет расположение. **Lens** меняет смысловой фокус той же Scene — например Order Flow, Liquidity, Options Context, Risk или Learning — без создания новой истории данных.

Command Bar, клавиатура, мышь и агент вызывают именованные команды одного registry. У каждой команды определены inputs, permissions, side effects и результат. Предлагая открыть панель, выделить уровень или построить overlay, агент указывает target object, reason и Evidence. ChangeSet содержит base revision; конфликт с уже изменённой Scene требует нового preview. Undo восстанавливает предыдущий UI/object state в допустимых границах.

Replay является режимом времени текущего Workspace: сохраняются layout и связанные objects, а data access ограничивается replay cursor. Переход назад не оставляет в контексте незаметно доступные будущие quotes, новости или результаты сделки. Live/paper/replay видны в интерфейсе постоянно.

### 7.4 Terminal и Options: конкретные рабочие возможности

| Область | Содержание и связь с работой |
|---|---|
| Terminal | Underlying chart, tape, depth/DOM, heatmap, footprint, CVD, VWAP и profiles по доступным данным; typed events и evidence |
| Options Overview | Spot/index, IV/RV, term/skew, expected move, ключевые уровни, quality и объяснение того, что изменилось |
| Smart Chain | Bid/ask и размеры, IV/Greeks/OI, spread/liquidity, expiry/delta filters; выбор нескольких контрактов для структуры |
| Strike Inspector | Единая карточка strike/expiry из Chain, Gamma, Flow, Scenario или underlying overlay; история и source timestamps |
| Gamma Map | Profile и time×price view; отдельные модели concentration, signed GEX assumptions, observed flow и estimated inventory |
| Options Flow | Prints, block/RFQ metadata при наличии, aggressor flow, inferred structures с маркированной неопределённостью |
| Volatility | Surface, smile/skew, risk reversal/butterfly, term structure, IV/RV и исторические percentiles |
| Scenarios | Ветки развития, условия подтверждения/отмены, evidence for/against, target/range и отличие от предыдущей версии |
| Strategy Lab | Шаблоны и свои legs/rules; payoff сейчас/на expiry, Greeks, time/spot/IV shocks, сравнение, переход в Quant/Paper |
| Positions | Book P&L, collateral/margin из правильного owner, aggregate Greeks, expiry/strike concentration и full revaluation/stress |
| Options Scalping | Синхронные underlying/option views, strikes in play, доступность ликвидности и gated execution ticket |

Это целевой каталог возможностей, а не перечень одновременно готовых экранов. Начальная поверхность использует минимальные панели для законченного процесса. Глубокий high-rate renderer выносит обработку потока из React; tick path не зависит от LLM.

Options и Terminal связаны временем, инструментами и Thesis. Нажатие на option print переводит underlying cursor к тому же моменту. Gamma-level открывает исходную модель и наблюдавшиеся реакции. Переход из Scenario в Strategy Lab переносит контекст, а не открывает пустую форму. OI/gamma не доказывают точный dealer inventory; оценка режима остаётся моделью. Недоступный quote не заменяется нулём, а cross-venue dispersion при единственном источнике остаётся неопределённой.

### 7.5 Живые Thesis, Watch и Journal

Thesis хранит horizon, claims, evidence for/against, scenarios, confirmation, invalidation, неизвестные и expiry. Новое наблюдение создаёт revision и видимый delta. Пользователь может вернуться к состоянию «что было известно тогда»; последующий исход не переписывает основание старого решения.

Watch показывает формальные predicates, inputs/freshness, semantic question, cooldown, срок действия и правила уведомления. На событии он сначала проверяет данные, затем при необходимости запускает ограниченное исследование. Пропавший feed переводит зависимую проверку в degraded/paused; отсутствие информации не считается подтверждением Thesis.

Journal связывает исходную Scene, действие пользователя, accepted/edited/rejected предложения, позиции и outcome. CSV-импорт собственной истории может дать начальный материал для Decision Memory до подключения exchange keys. Process review различает удачное решение, случайно прибыльный результат, нарушение правил и недостаток данных.

Post-trade companion использует актуальный position/risk snapshot: объясняет P&L по движению underlying, IV, времени, fees/spread и residual approximation error; сравнивает Hold/Close/Roll/Reduce как пересчитанные варианты. Watch или уведомление сами не открывают сделку. Закрытие/отзыв доступа не зависят от доступности AI-чата.

### 7.6 Personal Agent Builder и Agent Console

Пользователь задаёт **Goal → Inputs → Tools → Trigger → Policy → Output → Review**. Примеры: «Проверять мою опционную гипотезу после закрытия 4H», «Сообщать о существенном изменении skew», «Каждое утро собирать brief по моим Thesis». Это конфигурация ограниченной capability, а не разрешение на произвольный shell или кошелёк.

Перед активацией Builder показывает источники, доступные actions, triggers, бюджет, пример output, условия остановки и необходимые approvals. Новые права требуют отдельного действия пользователя. Один AgentDefinition может порождать короткие Runs или долгоживущую Mission; live permissions появляются только после соответствующего этапа готовности.

Console показывает state, текущую задачу, следующий запуск, tool calls, evidence, model/skill versions, стоимость, checkpoint, ошибки, approvals и причину блокировки. Pause Mission, cancel Run, revoke grant, disable deployment и cancel order различаются в UI. Отмена текущего LLM-ответа не гарантирует остановку уже отправленного приказа; состояние сверяется с Execution.

### 7.7 Учебная и профессиональная глубина

Simple / Trader / Quant меняют глубину представления одних данных. Simple объясняет экономический смысл, стоимость и ограничения; Trader открывает chain, Greeks и flow; Quant — формулы, inputs, timestamps, source versions и controls эксперимента. Профессионалу не нужно проходить обязательный обучающий диалог перед обычной операцией.

Learning Workspace объединяет Guided Replay, Live Tutor и Review. Tutor предлагает остановиться на историческом моменте, сформулировать гипотезу, проверить её на доступных тогда данных и сравнить решение с последующим исходом. Обучение использует те же Evidence, расчёты и защиту от будущих данных. Учебный прогресс не выдаёт торговые права.

Calm-by-default означает material updates, debounce, quiet hours, attention budget и группировку связанных событий. Пользователь делегирует наблюдение без обязанности читать каждый tick. Аварийные risk events обрабатываются по отдельной policy и не зависят от доступности AI.

### 7.8 Продолжение, экспорт и распространение

Decision Passport связывает goal, snapshot, alternatives, costs, risk, решение, approvals и outcome. Strategy Passport добавляет точную spec/code/data/engine version, trials, validation и раздельную paper/live историю. Research Package переносит разрешённые данные/refs, methodology и objects в внешнюю среду; доступность экспорта зависит от data rights.

Share-карточка рыночного момента или стратегии может быть открыта, изменена и проверена другим пользователем. Получатель получает новую версию и пересчёт на своём состоянии; чужой анализ не передаёт полномочий на торговлю и не раскрывает частную историю автора. Переносимые пакеты остаются полезны даже без публичного permalink.

WebMCP рассматривается как дополнительный браузерный интерфейс контекста и разрешённых действий. Он использует те же серверные contracts и permissions; поддержка конкретным browser/agent проверяется отдельно. Workstation-first не зависит от этого механизма. [Описание WebMCP от Chrome](https://developer.chrome.com/blog/webmcp-epp).


### 7.9 Полнота Workstation после отказа от Emacs

Переход на React/TypeScript сохраняет не внешний вид редактора, а объектную рабочую модель: независимые views вместо buffers, единые команды вместо mode-specific shortcuts, долговечные Workspace/Layout/Lens, точное выделение и общий agent context. Полный каталог восстановлен в [Workstation specification](DEVELOPMENT.md#DEP-568bec8dbe): **72 исходных типа views, 12 panel roles, 15 workspaces и 12 lenses**. Proposed IDs выведены из исторических modes и не объявлены опубликованным API.

15 workspaces: Morning, Market, Level Investigation, Flow, Options, Quant Lab, Thesis, Missions, Portfolio, Pretrade, Position Guard, Replay, Review, Operations и Custom. Двенадцать lenses: Price, Flow, Liquidity, Perpetuals, Options, Thesis, Position, Execution, Replay, Regime, Quant и Review. Это каталог задач и представлений, а не 72 панели на экране и не столько же microservices. Default остаётся спокойным: Primary Canvas, AI/Context, Command Bar и status strip; остальные views открываются по потребности.

Вернулись также точные поведенческие границы: comparison instruments/venues отдельно от similarity search; synchronized cursor/options/underlying; сохранить layout отдельно от Thesis; Scene snapshot/delta/epoch/resync; backpressure; renderer fallback; восстановление сессии после crash; privacy/export; keyboard access; scope-aware Agent Builder; предложения изменения среды и различие Undo/cancel/close. Полная Workstation расширяется по текущей очереди до PRO_WORKSTATION; FLOW_DESK_LOCAL и FLOW_DESK_PILOT имеют отдельный scope. Полный каталог и первый платёж не являются техническими предпосылками этих экранов.

### 7.10 Техническая приёмка среды

Проверяются object identity UI/API/agent, точность SelectionRef/asOf, один policy path для мыши/клавиатуры/агента, запрет stale ChangeSet, replay future firewall, корректный resync после gap, numerical/model labels в Options и отсутствие private hosted machinery в клиенте. Длительная работа и renderer recovery требуют отдельного runtime soak, не выводятся из наличия этих текстов. Исторические Emacs/Doom/Elisp и wrapper-harness предложения сохранены как reference, но не возвращаются в действующую архитектуру.

## Подробные требования реализации


## 3.1. Terminal Workspaces

| **System Workspace** | **Роль**                                                                    |
|----------------------|-----------------------------------------------------------------------------|
| Beginner Learning    | Guided Replay / Live Tutor / Review в одном обучающем рабочем пространстве. |
| Scalper Execution    | Высокоплотный order-flow и execution workspace.                             |
| Custom               | Пользовательская рабочая среда и база для собственных layouts.              |

> • Swing/Position/Options/Quant не показываются как ежедневные Terminal workspaces. Swing/Position-функции реализуются через layout presets и widgets; Options и Quant — отдельные глобальные модули.
>
> • My Workspaces сохраняются между сессиями и синхронизируются с аккаунтом. Действия: Rename, Duplicate, Delete; системные workspaces нельзя удалить, но можно дублировать.
>
> • Live/Replay — отдельный time/data mode, применяемый к открытому workspace.
>
> • Глобальный View selector отсутствует. Старые views становятся Layout Presets внутри Customize.

# 4. Основные сущности продукта

| **Сущность**  | **Минимальный смысл**                                                                                                   |
|---------------|-------------------------------------------------------------------------------------------------------------------------|
| Workspace     | Сохранённый рабочий стол: widgets, regions, geometry, bindings, density, AI preferences, instrument/timeframe defaults. |
| Layout Preset | Быстрый вариант раскладки текущего Workspace; не отдельная глобальная сущность.                                         |
| Widget        | Независимый модуль с symbol/time binding, state, settings, maximize/collapse и data/error states.                       |
| AI Event      | Стандартизированное рыночное событие с fact/evidence, horizon, data quality и chart annotations.                        |
| Thesis        | Persistent hypothesis: focus, evidence, confirmation, invalidation, lifecycle и history.                                |
| Watch         | Детерминированное условие мониторинга, созданное вручную или AI из natural language.                                    |
| Scenario      | Условная ветка будущего поведения; использует weight/probability только при корректной калибровке.                      |
| Journal Entry | Решение/наблюдение с thesis, evidence, action, outcome, AI review и reproducible chart state.                           |
| Strategy Spec | Читаемая формализация quant-идеи: universe, events, filters, entry/exit, risk, execution assumptions.                   |
| Experiment    | Dataset/version + parameters + code/spec + result + validator output.                                                   |
| Agent         | Goal, inputs, tools, trigger, policy, output, approvals, state, budget и audit.                                         |
| Risk Policy   | Human-readable и machine-validatable ограничения исполнения.                                                            |
| Option Level  | Strike/price level + model type + strength + expiry/venue mix + role hypothesis + evidence/history.                     |

# 5. Global App Shell

## 5.1. Постоянная оболочка

> • Odelix brand / app menu.
>
> • Глобальная навигация: Terminal, Options, Quant Lab, Journal, Settings.
>
> • В Terminal — один Workspace selector, например «Scalper Execution ▾».
>
> • LIVE / REPLAY mode.
>
> • Environment: read-only / paper / testnet / live.
>
> • Risk status: daily loss, risk policy state, account/venue status; Kill Switch всегда доступен в execution-контексте.
>
> • Компактный AI autonomy indicator, например «AI L3 · WATCH».
>
> • ASK ⌘K — вход в AI Command Bar; постоянное большое поле Ask в header не нужно.
>
> • Customize и Add Widget в Terminal; Data Map/Data Quality там, где это существенно.
>
> • Account, exchange latency/data freshness и connection state.

## 5.2. Calm-by-default

> • В спокойном состоянии chart и order-flow widgets занимают визуальный приоритет.
>
> • AI rail, thesis details, options details и secondary diagnostics раскрываются on demand.
>
> • Активная Thesis показывается узкой полосой только когда реально существует; отдельный дублирующий persistent chip не нужен.

# 6. Workspace, Layout Presets и Widget System

## 6.1. Workspace semantics

> • Workspace отвечает на вопрос «какой рабочий стол открыт?». Он может менять состав widgets, geometry, bindings и основную задачу пользователя.
>
> • Layout Preset отвечает только за раскладку внутри текущего Workspace.
>
> • Примеры presets: Volume Profiles, Full Flow, Chart Focus, Review, Multi-monitor, Custom.
>
> • Не добавлять локальный глобальный уровень Chart \| Execution \| Risk над всей рабочей зоной; режимы должны жить внутри конкретных widgets.

## 6.2. Layout engine

> • MVP/default: region-based Main / Right / Bottom / Auxiliary — предсказуемый layout и стабильный chart cluster.
>
> • Widgets могут быть tabbed, stacked, split, collapsed и maximized; focus mode не разрушает сохранённый layout.
>
> • Native/Desktop future: detachable windows и multi-monitor groups.
>
> • Link groups синхронизируют symbol, timeframe/time, price cursor и replay timestamp.
>
> • Autosave текущего workspace + indicator «изменено».
>
> • Snapshot перед reset/large changes, undo/redo и recovery после crash/reload.
>
> • Import/export layout без API keys, secrets и персональных данных.
>
> • Закрытие widget удаляет его только из текущего layout, но не из Widget Registry.
>
> • Недоступный widget показывается честным placeholder, а не пустым production-like экраном.

## 6.3. Customize

> • Show/hide Right/Bottom/Auxiliary regions.
>
> • Resize region boundaries.
>
> • Move/reorder widgets.
>
> • Create tab/stack groups.
>
> • Maximize/collapse.
>
> • Layout Presets.
>
> • Duplicate Workspace, reset to system template, save user copy.
>
> • Density/profile customization.
>
> • Color calibration и accessibility options.

# 7. Terminal: market-data и order-flow ядро

## 7.1. Chart Cluster

> • Heatmap / liquidity history с configurable depth/intensity thresholds.
>
> • Footprint / cluster view.
>
> • Candles / price bars как альтернативный слой.
>
> • Executed trade bubbles/prints с агрессором и размером.
>
> • POC, VAH/VAL, HVN/LVN и volume profile overlays.
>
> • VWAP и Anchored VWAP; центральная линия и deviations должны быть семантически различимы.
>
> • Drawings: horizontal/vertical lines, zones, trendlines, notes, selected ranges.
>
> • Synchronized crosshair и cursor linking между chart, CVD, tape, options и replay.
>
> • Instrument/timeframe bindings; multi-symbol tabs.
>
> • Chart annotations от AI Event/Thesis/Watch используют единые event IDs.

## 7.2. Professional widgets

| **Widget**              | **Функции**                                                                                                 |
|-------------------------|-------------------------------------------------------------------------------------------------------------|
| DOM / COB               | Price ladder, size, own orders, queue estimate, quick cancel, spread/depth context, pull/add/replenishment. |
| Tape / Time & Sales     | Executed trades, aggressor, filters, large prints, synchronization with chart.                              |
| CVD                     | Cumulative aggressive buy minus sell volume, reset scope, divergence context.                               |
| BID/ASK / Delta         | Aggressor split, per-bar delta, stacked imbalances, divergence.                                             |
| SVP / CVP               | Session/Composite Volume Profiles; POC, value area, HVN/LVN.                                                |
| Positions/Risk Strip    | Positions, P&L, liquidation distance, active orders, daily loss, risk state.                                |
| Fast Ticket             | Size, order type, bracket presets, fee/slippage preview, reduce-only, quick actions.                        |
| Options Context Overlay | Gamma levels, zero-gamma estimate, expected move, relevant option-flow events и regime summary.             |

## 7.3. Event types

> • Liquidity added/pulled
>
> • Sweep
>
> • Absorption
>
> • Replenishment / iceberg-like activity
>
> • Failed breakout
>
> • Stop cascade
>
> • Liquidation cluster
>
> • CVD divergence
>
> • Spot/perpetual divergence
>
> • Funding/OI regime change
>
> • Basis change
>
> • IV shock
>
> • Skew rotation
>
> • Large option block
>
> • Expiry pinning risk

## 7.4. Terminal Options overlays

> • Default Γ Levels density = Top 3; controls: Top 3 / Top 5 / All, hide weak, min confidence, scenario-relevant only.
>
> • Collapsed options strip: Mixed Γ · Zero 64.2k · EM ±1.7% · Focus/Nearest 66k. Secondary details по hover/detail.
>
> • Relevant levels may carry role hypotheses: magnet, pinning, barrier, catalyst; role is inference, not property of price.
>
> • Invalidated overlays stay muted/struck-through until user hides them; AI does not silently remove historical evidence.

# 8. Contextual Chart Intelligence: выделение области и Ask AI

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Одна из ключевых UX-фишек</strong></p>
<p>Пользователь выделяет область графика как в IDE/Cursor, прикладывает вопрос, а Odelix передаёт в AI не только изображение, а структурированный рыночный контекст выбранного диапазона.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 8.1. Выделение области

> • Инструмент range/rectangle selection выбирает time range + price range.
>
> • После выделения появляется mini action bar: Explain · Compare · Ask AI · Create alert · Turn into rule · Add to journal.
>
> • Ask AI открывает AI Chat/Command Bar с прикреплённым context chip выбранной области.
>
> • Выделение остаётся визуально подсвеченным, пока AI объясняет его; ответ может добавлять временные annotations.
>
> • Compare позволяет сопоставить область с другой выбранной областью или найти похожие исторические эпизоды по явным similarity criteria.
>
> • Feature работает и в LIVE, и в REPLAY; в Live Tutor запрещён hindsight.

## 8.2. Что попадает в AI вместе с выделением

> • Instrument, venue, environment, workspace, timeframe, live/replay timestamp.
>
> • Точные time/price bounds selected region.
>
> • Visible chart layers и их current settings.
>
> • Executed trades, CVD/Delta, nearby liquidity changes, VWAP/profile levels в диапазоне.
>
> • Активные Options overlays/levels и option-flow events, если включены.
>
> • Активная Thesis/Watch и relevant AI Events.
>
> • Data freshness/quality и gaps.
>
> • Visual snapshot для человеческой геометрии + structured data для доказуемых чисел.

## 8.3. Contextual AI actions по другим widgets

> • Indicator/widget → Explain this move.
>
> • DOM/COB → Explain liquidity pull, replenishment, spread quality, adverse selection.
>
> • Order ticket → Position sizing, fee/slippage preview, invalidation consistency, risk check.
>
> • Option strike/level → Explain IV/Greeks/OI/flow/role and open inspector.
>
> • Replay → Find key moments, compare, quiz, compressed review.
>
> • Journal → Session summary, recurring mistakes, playbook performance, next exercise.

# 9. AI-native Shell и единый AI Runtime

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Архитектурное правило</strong></p>
<p>Market data → deterministic engines → FACT/CALCULATION → inference models → CONFIRMATION → AI orchestration → THESIS/WATCH/EVENT. AI runtime не вычисляет рыночные метрики и не invents levels.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 9.1. AI surfaces

| **Surface**          | **Роль**                                                                         |
|----------------------|----------------------------------------------------------------------------------|
| ASK ⌘K / Command Bar | Natural-language вход и управление UI. Временный: ask → preview → apply → close. |
| AI Panel · Events    | Проактивные meaningful state changes, compressed feed.                           |
| AI Panel · Chat      | Свободный диалог с текущим market/UI context.                                    |
| AI Panel · Agent     | Задачи/агенты, состояние и контролируемые actions.                               |
| AI Panel · History   | История вопросов, thesis/watch transitions, previous explanations.               |
| Beginner Tutor       | Lesson / Ask / History поверх того же shared runtime.                            |
| Options Analyst      | Options-specific context в той же AI системе, не отдельный AI-продукт.           |

## 9.2. AI Command Bar

> • Расширяет существующую ⌘K palette: сверху ASK, ниже parsed intent, answer и UI actions; обычные команды остаются в том же списке.
>
> • Примеры: «Почему BTC не проходит 66k?», «Покажи ближайшую ликвидность и gamma levels», «Что изменилось за час?», «Собери setup для scalp short от 66k», «Следи за 64.2k».
>
> • AI выбирает из существующих workspace/widgets/layers; не генерирует произвольный новый интерфейс.
>
> • Parsed intent имеет безопасный unknown outcome: если запрос не понят — никаких thesis/actions/watch не создаётся.
>
> • Causal wording должен быть осторожным: «what may be limiting price», а не «why price is held», если причинность не доказана.

## 9.3. UI orchestration

> • AI показывает grouped WILL CHANGE: Focus / Evidence layers / Options context.
>
> • Preview-before-apply — default. Кнопка «Apply N chart changes».
>
> • Workspace change и navigation требуют отдельного explicit confirmation с объяснением, что заменится.
>
> • После apply появляется «AI CHANGED THE VIEW · what changed · UNDO». Undo возвращает toggles, zoom, pan, timeframe и panes к точному pre-AI snapshot.
>
> • Все tool calls видимы пользователю и пишутся в audit/history.
>
> • Пользователь может позже включить preference Auto-apply safe changes, но не по умолчанию.

## 9.4. AI Event contract

| **Поле**                        | **Описание**                                                                            |
|---------------------------------|-----------------------------------------------------------------------------------------|
| eventId                         | Один ID для chart chip, compact card, feed event и journal reference.                   |
| timestamp / instrument          | Когда и где произошло.                                                                  |
| observation                     | Что наблюдалось.                                                                        |
| interpretation                  | Что это может означать; не смешивать с observation.                                     |
| importance / horizon            | Severity и срок актуальности.                                                           |
| supporting/conflicting evidence | Данные за и против.                                                                     |
| confirmation / invalidation     | Заранее названные observable conditions.                                                |
| data quality                    | Freshness, gaps, unavailable feeds.                                                     |
| chart annotations               | Levels/time ranges/markers.                                                             |
| actions                         | Show on chart, Ask AI, Pin, Mute similar, Create alert, Add to journal, Turn into rule. |

## 9.5. Evidence taxonomy

| **Тип**      | **Правило**                                     |
|--------------|-------------------------------------------------|
| FACT         | Observed data, unmodified.                      |
| CALCULATION  | Deterministic computation over facts.           |
| INFERENCE    | Model estimate / role hypothesis; withdrawable. |
| CONFIRMATION | Actual market reaction supporting a thesis.     |
| SCENARIO     | Conditional branch with explicit invalidation.  |

> • CONFIRMATION используется только для реально подтверждающего чтения. Противоречащий CVD — FACT/conflicting evidence, а не confirmation.
>
> • Не показывать необоснованные точные probabilities/confidence. Default: Low/Medium/High + evidence count. Scenario number называется Weight, если он не статистически calibrated probability.

## 9.6. Simple / Trader / Quant

| **Depth** | **Presentation**                                                                              |
|-----------|-----------------------------------------------------------------------------------------------|
| Simple    | Простой summary, один focus level, expected range, thesis, минимум jargon.                    |
| Trader    | Top 3 levels, CVD/VWAP, confirmation/invalidation, relevant option flow, actionable context.  |
| Quant     | Raw values, formulas, assumptions, model/version, sample size, history, statistical evidence. |

> • Default выбирается по workspace, но пользователь имеет явный переключатель.

## 9.7. Thesis

> • Thesis — persistent AI/workflow object, не просто сообщение в chat.

| **Поле**   | **Содержание**                                                     |
|------------|--------------------------------------------------------------------|
| Identity   | title, asset, timeframe, source scenario.                          |
| Focus      | levels/range и role hypotheses.                                    |
| Evidence   | fact/calculation/inference references.                             |
| Conditions | confirmation + invalidation.                                       |
| State      | FORMING / ACTIVE / CONFIRMED / WEAKENING / INVALIDATED / RESOLVED. |
| History    | timestamped transitions + reasons.                                 |
| Context    | related watches, journal entries, options/quant links.             |
| Quality    | data freshness and monitoring qualifier.                           |

> • Collapsed Terminal strip: ACTIVE · thesis name · watching focus. Expanded: focus, confirmation, invalidation, confidence/weight, source, watches, history.
>
> • Lifecycle transition должен происходить по deterministic condition, не потому что LLM «передумал».
>
> • FORMING — AI собирает evidence; RESOLVED — нормальное завершение без invalidation.
>
> • Legacy/incompatible thesis никогда не удаляется: LEGACY · MONITORING OFF · Rebuild Thesis / Keep in History.

## 9.8. Watch

> • Natural-language watch компилируется в видимые deterministic predicates.
>
> • Пример acceptance: 2 consecutive 5m closes \> threshold + spot CVD positive/rising + feed freshness \< N sec.
>
> • Пользователь может раскрыть и отредактировать predicate.
>
> • При stale feed watch становится PAUSED, а не fires/cancels.
>
> • Trigger создаёт одно Event в общей AI Events ленте и не является приказом на сделку.

## 9.9. Data degraded и monitoring

> • DATA DEGRADED / MONITORING PAUSED — qualifier поверх lifecycle. ACTIVE остаётся ACTIVE; отсутствие данных не означает weakening.
>
> • При восстановлении monitoring resumes с текущего чтения; gap не backfill-ится выдуманными выводами.

## 9.10. Proactive AI и event compression

> • AI проявляется только при meaningful state change: regime changed, thesis confirmed/weakening/invalidated, watch triggered, important level materially changed, unusual flow, data degraded/restored.
>
> • Несколько чтений объединяются: THESIS WEAKENING · 3 THINGS CHANGED, а не четыре popup подряд.
>
> • AI rail может быть collapsed: «AI · 1 active thesis · 2 watches».
>
> • AI не удаляет layers, не меняет workspace и не выполняет live order незаметно.

## 9.11. Automation Ladder

L0–L5 — читаемые UX-профили, производные от Product AgentSpec: interaction × placement × environment × authority × brain. Номер уровня не определяет разрешение сам по себе. L5 — только PAPER; L6 остаётся будущей отдельно допускаемой возможностью.

MANAGED разрешён для read/draft/симуляции в соответствующем мандате, не для автоматической активации реального исполнения. BYOK сам по себе не требует Node. Hosted 24/7 observation — без VPS пользователя. Аутентифицированная ApprovalInbox вызывает Product; уведомления не разрешают действий. Полная нормативная модель принадлежит Product AGENT-LIFECYCLE, локальная сводка — DEVELOPMENT.

# 10. Beginner Learning и AI Tutor

## 10.1. Цель

Обучать order flow непосредственно на живом или replay-графике, заставляя пользователя сначала сформировать гипотезу, затем раскрывая evidence и оценивая process, а не только outcome.

## 10.2. Layout

| **Зона** | **Содержание**                                                         |
|----------|------------------------------------------------------------------------|
| Main     | Simplified chart/heatmap; ограниченная palette; 1–3 highlighted zones. |
| Tutor    | Lesson / Ask / History; 35–42% width, full height.                     |
| Bottom   | Replay controls, Paper Decision, Journal, Process Review.              |
| Header   | Lesson goal/progress, REPLAY/DELAYED/LIVE, explain terms.              |

## 10.3. Tutor workflow

> **1.** Система замечает или выбирает учебное событие.
>
> **2.** Пользователь формулирует гипотезу до объяснения.
>
> **3.** Tutor показывает evidence и competing interpretation на графике.
>
> **4.** Пользователь выбирает Wait / Skip / Paper trade.
>
> **5.** После развития события AI оценивает reasoning и risk adherence, а не только P&L.
>
> **6.** Результат сохраняется в Journal и влияет на персональный curriculum.

## 10.4. Tutor AI tools

> • Highlight level/range.
>
> • Open CVD/Footprint/DOM temporarily and explain.
>
> • Pause/step replay/backtrack.
>
> • Show competing interpretation.
>
> • Find similar episodes with similarity criteria.
>
> • Create journal note.
>
> • Draft paper ticket.
>
> • Answer «почему это absorption?» с evidence на chart.
>
> • Free text and voice input.
>
> • Live Tutor may answer «insufficient data» and cannot use future facts.

## 10.5. Tutor states

> • Waiting for hypothesis
>
> • AI thinking/tool use
>
> • Evidence revealed
>
> • Paper decision
>
> • Outcome hidden/revealed
>
> • Reasoning review
>
> • Insufficient data
>
> • Live Tutor — no hindsight

# 11. Scalper Execution

## 11.1. Default layout

> • Main heatmap/footprint.
>
> • DOM/COB.
>
> • Tape.
>
> • Fast execution rail/ticket.
>
> • Compact AI Events feed.
>
> • Compact CVD.
>
> • Positions/Risk strip + Kill Switch.
>
> • Optional Options context strip and Γ overlays.

## 11.2. Execution UX

> • Hotkeys, preset sizes, bracket presets, reduce-only, flatten, cancel all, quick cancel.
>
> • Header/strip shows symbol, venue, spread, latency/data quality, environment, P&L and risk state.
>
> • AI Events are compact chips/cards, not long chat; hover = evidence preview, click = detail.
>
> • Show on chart highlights time/level synchronously.
>
> • Turn into rule opens a formal condition editor; does not create autonomous bot directly.
>
> • Voice alert is allowed for high-importance events; events visually expire after their horizon.

# 12. Replay

> • Replay swaps the data source but preserves workspace layout and relevant widgets.
>
> • Unified clock synchronizes chart, DOM/tape where recorded, CVD, options surface/GEX/flow, AI events and journal references.
>
> • Controls: play/pause, step, speed, event jump, return live.
>
> • Replay Library opens historical sessions/recordings.
>
> • Recorded gaps are first-class events and visible on timeline.
>
> • AI receives replay timestamp and sourceMode=replay; Tutor can know outcome only when the lesson explicitly reveals it.
>
> • Options Replay frame can show spot, regime, ATM IV, zero-gamma estimate, prints seen and surface availability.

# 13. Journal и память продукта

> • Journal существует как global screen и как dockable widget/context action.
>
> • Filters: Workspace, instrument, strategy/thesis, date, Live/Replay, outcome, process score.
>
> • Entry stores thesis, evidence, action, outcome, AI review, chart state, annotations and lightweight thumbnail.
>
> • Restore chart state recreates workspace, instrument, range, overlays and annotations.
>
> • Journal stores full Thesis transition history: what trader believed, pre-defined invalidation, what actually fired.
>
> • AI uses Journal as memory for recurring mistakes, playbook performance and personalized learning.
>
> • Context actions: Add selected chart region, AI Event, Watch trigger, order execution report, Options scenario/structure, Quant result.

# 14. Trading, Execution и Risk

## 14.1. Environment ladder

> read-only → paper → testnet → confirmed live orders → controlled automation
>
> • Каждый fill: expected vs actual, fees, slippage, quality and link to thesis.
>
> • Paper/testnet simulate spread, slippage, partial fills, latency and rejection reasons.
>
> • Order-book strategies expose queue/fill probability as an estimate, not fact.
>
> • Live environment is always explicit in UI; no hidden environment switch.

## 14.2. Risk Center

> • Account equity, margin, liquidation distance.
>
> • Exposure by instrument/exchange/risk type.
>
> • Daily loss, max drawdown, concentration, correlated exposure.
>
> • All active bots/agents with immediate kill/disable.
>
> • Risk override and blocked-action history.
>
> • Stress: price shock, volatility shock, exchange outage, spread widening.

## 14.3. Critical confirmations

> • paper/testnet → live
>
> • first API key connection
>
> • leverage/risk-limit changes
>
> • autonomous bot/agent start
>
> • safeguard disable
>
> • illiquid multi-leg option order

# 15. Odelix Options — полный модуль

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Options USP</strong></p>
<p>Crypto option-flow terminal, который связывает option positioning/volatility с фактическим order flow underlying/perpetual на одной временной и ценовой шкале.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 15.1. Навигация Options

| **Экран**    | **Назначение**                                                                                    |
|--------------|---------------------------------------------------------------------------------------------------|
| Overview     | Command Center, regime, expected move, volatility snapshot, key levels, AI summary, data quality. |
| Chain        | Smart Option Chain + presets + strike selection/inspector.                                        |
| Gamma        | Gamma models, price profile, time×price heatmap, key levels and history.                          |
| Flow         | Option flow tape, big flow bubbles, option CVD, blocks/RFQ, inferred structures.                  |
| Volatility   | IV surface, smile/skew, risk reversal/butterfly, term structure, expected move, IV vs RV.         |
| Scenarios    | Evidence-based scenario weights, target zone, invalidation, what changed, analyst log.            |
| Replay       | Unified options clock + event timeline + data/feed state.                                         |
| Strategy Lab | Structure builder, payoff now/expiry, scenario shocks, Greeks, risk and Quant/Journal bridge.     |
| Positions    | Options book P&L, margin, Greeks, stress/shocks, expiry/strike concentration.                     |
| Scalping     | Short-horizon underlying + options context, strikes in play, liquidity, flow tape and ticket.     |

## 15.2. Global Options context

> • Underlying
>
> • Venue scope: All/Deribit/Binance/Bybit/OKX
>
> • Expiry scope
>
> • Spot/index reference
>
> • ATM IV
>
> • Data timestamp/latency
>
> • Live/Replay
>
> • Data Map / data confidence

## 15.3. Overview / Command Center

> • Human-readable one-line summary: what market is doing and what would change the read.
>
> • Spot/index = FACT.
>
> • Expected move = CALCULATION.
>
> • ATM IV / IV-RV / percentile = CALCULATION.
>
> • Gamma regime = INFERENCE.
>
> • Regime breakdown: Pinning / Expansion / Neither (or other model-defined components).
>
> • Evidence chain + invalidation + history 1h/4h/24h.
>
> • Key levels list with distance, strength, role hypothesis (magnet/pinning/barrier/catalyst), trend and confidence.
>
> • AI Options Analyst summary + CTA Open Scenarios.
>
> • Data Quality block: venue coverage, history depth, stale/missing fields, single-venue dispersion = undefined rather than zero.

## 15.4. Smart Option Chain

> • Calls left, strike center, puts right; near-spot/ATM visually anchored.
>
> • Column presets: Beginner / Flow / Greeks / Liquidity / Research.
>
> • Heatmap modes: Gamma Concentration, OI, ΔOI, IV, Observed Gamma Flow, Off.

| **Data family** | **Fields**                                                                       |
|-----------------|----------------------------------------------------------------------------------|
| Quote           | bid, ask, mid/mark, spread, size, liquidity grade.                               |
| Volatility      | bid IV, ask IV, mark IV, ΔIV.                                                    |
| Greeks          | delta, gamma, theta, vega; future charm/vanna only when calculated.              |
| Positioning     | OI, ΔOI, concentration.                                                          |
| Flow            | volume, aggressive buy/sell, option CVD, gamma-flow contribution, event markers. |

> • Hover strike → mini data card.
>
> • Click strike → persistent Strike Inspector.
>
> • Multi-select strikes → compare/build structure.
>
> • Missing field stays «no quote»/unavailable, never zero-filled.
>
> • Filters: expiry, strike range, delta/target delta, hide illiquid, venue.

## 15.5. Universal Strike Inspector

> • Accessible from Chain, Gamma, Flow, Scenarios, Scalping and Terminal gamma label.
>
> • Quote/IV/Greeks/OI/ΔOI/volume/buy-sell flow/gamma concentration/observed gamma flow/liquidity/large trades/history.
>
> • Venue contribution and data quality.
>
> • Actions: Add to structure, Open in Gamma, Open on underlying, Add to Journal.

## 15.6. Gamma models

| **Layer**                     | **Meaning / limitation**                                                                                           |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Gamma Concentration           | Unsigned OI × multiplier × gamma × normalization. Tells where sensitivity sits, not positioning direction.         |
| Classical GEX                 | Signed model under dealer-opposite/call-put assumptions. INFERENCE, not observed dealer inventory.                 |
| Observed Gamma Flow           | Aggressor-signed executed flow × gamma × size/multiplier. Observed flow, not dealer inventory.                     |
| Estimated Net Gamma Inventory | Probabilistic inventory model using flow + OI changes + expiry/reconciliation. Must expose confidence/assumptions. |
| Gamma Change                  | Change/build/decay/migration of concentration over selected lookback.                                              |
| Cross-Venue Consensus         | Normalized venue contribution and dispersion; requires ≥2 venues, otherwise undefined.                             |

> • Zero Gamma = model regime boundary, not VWAP/fair value.
>
> • Positive/negative gamma describes possible shape/regime, not guaranteed direction.

## 15.7. Gamma Map

> • Default split: price-aligned profile left + time×price heatmap right; shared price scale and underlying path.
>
> • Layer selector: Gamma Concentration / Classical GEX / Observed Gamma Flow / Estimated Inventory / Gamma Change / Cross-Venue Consensus.
>
> • Controls: lookback 1h/4h/12h/24h/3d/custom; buckets 5m/15m/30m/1h/4h; strike range; expiry/DTE filters; raw/normalized; cutoff; underlying path toggle.
>
> • Hover cell → tooltip time, strike, value/change, expiry/venue contribution; corresponding strike highlights in profile and underlying point highlights.
>
> • Click/pin → fixed detail/level inspector.
>
> • Key Levels inspector: model, strength, call/put mix, expiry mix, created time, trend, distance, zero-gamma relation, venue contribution, observed reactions, role hypothesis, confidence.
>
> • Role label is withdrawable hypothesis and is removed/muted when behavior no longer supports it.

## 15.8. Flow

> • Big Flow Bubbles: call/put shape, buy/sell color, size=notional/premium, block/RFQ outline; avoid overloading opacity with confidence unless essential.
>
> • Option CVD: cumulative aggressor-signed contracts/flow across window; measures executed flow, not current inventory.
>
> • Flow Tape fields: time, venue, instrument, side/aggressor, size/notional, trade IV, spot at trade, block/RFQ, gamma contribution.
>
> • Filters: all prints, blocks/RFQ, calls, puts, structures.
>
> • Inferred Structures are INFERENCE with confidence and matching rationale: vertical, calendar/roll, straddle/strangle, risk reversal, single-leg, unclassified.
>
> • Click option trade → synchronized underlying cursor at same timestamp.

## 15.9. Volatility

> • IV Surface heatmap by expiry × strike/moneyness.
>
> • Smile & skew by expiry.
>
> • Risk reversal and butterfly.
>
> • Term structure across expiries.
>
> • Expected move by session/expiry.
>
> • IV vs realized volatility with history/percentile.
>
> • Beginner copy explains that IV is price of uncertainty, not directional forecast.

## 15.10. Scenarios

> • Primary/secondary/residual branches; use Weight unless probabilities are statistically calibrated.
>
> • Each scenario: title, regime read, evidence chain, target zone, invalidation, confidence/data scope, action links.
>
> • What Changed panel: facts/calculations/inferences since previous state.
>
> • Analyst Log answers Why? with taxonomy and weaker-link notes.
>
> • Actions: Watch on underlying, Build fitting structure, Log thesis, Open Chain/Gamma/Terminal.
>
> • Scenario → Terminal transfers thesis/focus/confirmation/invalidation and auto-highlights relevant level.
>
> • Scenario → Strategy Lab transfers expiry/range/strikes/expected move; structure is expression of thesis, not guaranteed recommendation.

## 15.11. Strategy Lab

> • Templates: long call, call spread, put spread, straddle, strangle, iron condor, covered call; extensible.
>
> • Payoff today vs expiry.
>
> • Expected-move overlay and spot marker.
>
> • Market-state sliders: time to expiry, IV shock, underlying price; optional skew shock.
>
> • Greeks now vs scenario.
>
> • Net cost, max profit/loss, breakevens; model probability only if properly defined.
>
> • Compare structures.
>
> • Risk check before paper/live.
>
> • Actions: Place paper structure, Send live (gated), Send to Quant Lab for historical test, Write to Journal.

## 15.12. Positions

> • Total/realized/unrealized P&L, margin used, net theta and portfolio Greeks.
>
> • Book payoff today vs expiry.
>
> • Stress & shocks: underlying ±5/10%, IV ±5/10 vol points, +1 day theta, combined shocks.
>
> • Margin/equity/available.
>
> • Where risk sits: delta/gamma/vega/theta by contract.
>
> • Expiry concentration and Greeks by expiry/strike.
>
> • Full revaluation where possible; clearly label model/venue-reported values.

## 15.13. Options Scalping

> • Top cards: gamma regime, nearest level, daily expected range, zero gamma, option CVD.
>
> • Underlying chart with gamma levels / expected range / zero gamma / VWAP / big prints; CVD and volume below.
>
> • Strikes in play ranked by relevance and role hypothesis.
>
> • Contract liquidity card: bid/ask, spread, mark IV, delta/theta, OI, liquidity grade.
>
> • Option Flow Tape synchronized with underlying.
>
> • Quick ticket paper-first; live depends on venue/risk backend.
>
> • Optional deep execution: underlying DOM + option DOM, synchronized tapes, IV tick, theoretical value, delta-adjusted P&L and quick hedge.

## 15.14. Replay in Options

> • One unified clock rewinds whole Options context.
>
> • Frame: spot, regime, ATM IV, zero-gamma estimate, prints seen, surface recorded.
>
> • Timeline event types: regime change, level event, flow print, underlying confirmation, recorded gap.
>
> • Feed states: Live, Stale surface, Partial outage, No history yet, Disconnected.
>
> • Historical depth is honestly limited to recorded period when venues do not serve retroactive surfaces/Greeks/OI.

## 15.15. Options ↔ Terminal integration

> • Gamma levels / zero gamma / expected move / option-flow markers overlay directly on underlying Terminal.
>
> • Click Gamma level → highlight relevant underlying reactions / CVD / bubbles.
>
> • Confirm on underlying opens Terminal in existing workspace with thesis context, not a blank chart.
>
> • Compact AI Options Analyst inside Terminal uses same AI history/runtime and CTA Open full scenario.
>
> • Options context is a contextual layer; user can trade underlying without opening Options page.

## 15.16. Data sources и ограничения

| **Venue**       | **Publicly usable data**                                                                                  | **Structural limitation**                                           |
|-----------------|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| Deribit         | Instruments, ticker, mark/bid/ask IV, Greeks, OI, book, trades, underlying/index. Primary Phase 1 source. | No public dealer/customer classification or exact dealer inventory. |
| Binance Options | Index, klines, OI, mark/bid/ask IV, Greeks, book, trades, block trades.                                   | Unsigned OI; no dealer classification.                              |
| Bybit Options   | Ticker, IV, Greeks, OI, trades, websocket, historical volatility.                                         | No exact inventory side; historical surface must be recorded.       |
| OKX Options     | Instruments, tickers, trades, books, OI, option/mark/index data, Greeks.                                  | No dealer/customer classification.                                  |

> • Exact dealer GEX / exact market-maker hedging demand cannot be claimed from public crypto venue APIs.
>
> • History of IV surface, Greeks, OI and order book must be recorded from day one for Replay/research.
>
> • Cross-venue normalization requires canonical option ID, multiplier/settlement normalization and data-quality metadata.

# 16. Quant Lab

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Workflow</strong></p>
<p>Idea → Formalize → Test → Inspect → Validate → Deploy</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 16.1. Natural-language research

> **1.** User enters natural-language hypothesis.
>
> **2.** AI generates readable Strategy Spec and clarification questions.
>
> **3.** User selects dataset/features and execution assumptions.
>
> **4.** Run creates visible job, progress, logs and chart markers.
>
> **5.** Results show expectancy, drawdown, fill rate, slippage, regime breakdown, parameter sensitivity, OOS degradation.
>
> **6.** Independent validator checks leakage, overfitting, sample size and sensitivity.
>
> **7.** Validated strategy can become Paper Agent, then Testnet, then controlled Live.

## 16.2. Workspace components

> • Project/data tree.
>
> • Readable spec + optional Python/DSL editor.
>
> • AI Research Copilot / inspector.
>
> • Dataset catalog with schema/version/time range/quality/gaps/cost/retention.
>
> • Backtest chart with signals, fills and rejected opportunities.
>
> • Results/Validation/Regimes/Logs/Deployment tabs.
>
> • Multi-run compare.
>
> • Bot monitor after deployment.

## 16.3. Validation and realism

> • Fees, spread, slippage, partial fills, latency, funding.
>
> • Queue/fill probability where order book is relevant.
>
> • OOS, bootstrap, multiple-testing control.
>
> • Regime robustness: liquidity, volatility, funding/basis.
>
> • Capacity, adverse selection and exchange fragmentation.
>
> • Parameter stability and sensitivity.
>
> • No misleading headline if sample insufficient or backtest invalid.

## 16.4. Research swarm

> • Idea Agent
>
> • Feature Agent
>
> • Backtest Agent
>
> • Validator (maker-checker)
>
> • Regime Auditor
>
> • Execution Auditor
>
> • Risk Agent
>
> • Decay Monitor

# 17. Personal Agents и B2A

## 17.1. Personal Agent Builder

> Goal → Inputs → Tools → Trigger → Policy → Output → Review
>
> • Goal: что агент должен искать/делать.
>
> • Inputs/context sources.
>
> • Allowed tools.
>
> • Trigger/schedule/conditions.
>
> • Risk & permission policy.
>
> • Output/notification/action.
>
> • Human review/approval requirements.

## 17.2. Agent Console

> • Agent name/state/current task/progress/next run.
>
> • Tool calls, model/version, costs, memory, results/errors.
>
> • Approval queue and policy blocks.
>
> • Audit timeline linked to chart/journal/backtest.
>
> • Pause/kill/disable.
>
> • No live execution button without risk policy and explicit approval.

## 17.3. B2A platform contracts

| **API/Service**    | **Agent получает**                                                  |
|--------------------|---------------------------------------------------------------------|
| Market Context API | Regime, bias/context, key levels, invalidation, data quality.       |
| Event Stream       | Standardized market events with timestamp/evidence.                 |
| Agent Sandbox      | Data, backtests, paper orders, feedback loop.                       |
| Execution Gateway  | Place/cancel/hedge/reduce/close through unified policy layer.       |
| Risk Policy Engine | Allow/block before exchange.                                        |
| Audit API          | Data/model/prompt/decision/order/result history.                    |
| Agent Marketplace  | Connectable order-flow/options/news/risk/portfolio agents — future. |

# 18. Onboarding и Settings

## 18.1. Adaptive onboarding

> • 4–6 minutes, not every question for every user.
>
> • Collects experience, horizon/frequency, markets, order-flow knowledge, leverage/options/code, AI preferences, risk, devices.
>
> • Result: one ready Workspace preview, widget list, AI explanation depth and density.
>
> • Default start: Beginner Learning or Scalper Execution; Custom available manually.
>
> • Profile after onboarding moves to Settings and does not occupy topbar.

## 18.2. Settings sections

> • Profile & onboarding answers.
>
> • Workspaces and layout presets.
>
> • Notifications/alerts/watches.
>
> • AI preferences: explanation depth, auto-apply safe changes, autonomy level.
>
> • Data/exchange connections and latency/quality.
>
> • Trading safeguards and risk limits.
>
> • Hotkeys.
>
> • Display/accessibility/color calibration.
>
> • Security/2FA/API keys.
>
> • Billing/plan if commercialized.

# 19. Persistence, state sync и reproducibility

> • Workspace persists name/template version, instrument, timeframe, regions, widgets, settings, tab/stack state, collapsed/maximized state, density and AI preferences.
>
> • Autosave; snapshot before reset/large AI/UI changes; recovery after reload/crash.
>
> • Journal stores reproducible chart state + compressed thumbnail.
>
> • AI Event, Thesis and Watch histories use stable IDs and timestamps.
>
> • One Thesis object changes representation across Terminal/Options/Strategy Lab/Quant/Journal; no duplicated independent copies.
>
> • Native/Desktop implementation should store sensitive secrets separately from exportable workspace state.
>
> • Optional account sync should merge workspaces/journal preferences while exchange credentials remain in secure secret storage.

# 20. Data architecture и market/event store

## 20.1. Shared core

> • Market data connectors / collectors.
>
> • Normalized event schemas.
>
> • Order-book/market microstructure store.
>
> • Deterministic calculation engine.
>
> • Event Engine.
>
> • Options analytics engine.
>
> • Market State Model.
>
> • Shared AI Runtime.
>
> • Journal/Replay persistence.
>
> • Risk Policy Engine.
>
> • Experiment/backtest store.
>
> • Product Host embeds the published Odelix Pi fork runtime package; Web
> consumes Product commands and `harness.events.v1`, never the runtime itself;
> no external harness wrapper or second runtime exists.

## 20.2. Canonical events

> • Trade
>
> • Quote/Ticker
>
> • BookDelta
>
> • OpenInterest
>
> • Funding/Basis
>
> • Liquidation
>
> • OptionInstrument
>
> • OptionTrade
>
> • OptionQuote
>
> • OptionBookDelta
>
> • OptionTicker
>
> • AIEvent
>
> • ThesisTransition
>
> • WatchEvent
>
> • ExecutionFill

## 20.3. Options recording cadence

> • Trades: event-by-event.
>
> • Order book: incremental.
>
> • Ticker/Greeks: each update or normalized stream cadence.
>
> • OI: as frequently as venue supplies.
>
> • IV surface snapshot: seconds-level as capacity allows.
>
> • Persistent aggregate snapshot: ~1 minute.
>
> • Research bars: 1m/5m/15m and derived horizons.

# 21. Состояния ошибок, stale data и доверие

| **State**               | **UI behavior**                                                                 |
|-------------------------|---------------------------------------------------------------------------------|
| Loading                 | Skeleton; no fake market values.                                                |
| No data                 | Reason/source/last update/retry.                                                |
| Partial data            | Banner + missing channels; AI lowers/qualifies confidence.                      |
| Stale data              | Timestamp; watches/automation pause per policy.                                 |
| Disconnected            | Freeze last known state + DISCONNECTED watermark.                               |
| Exchange degraded       | Disable risky actions; show safe recovery/cancel/close options where available. |
| Permission denied       | Required scope + safe remediation.                                              |
| Policy blocked          | Explain specific rule; never suggest bypass.                                    |
| AI unavailable          | Deterministic events/manual workflow remain fully usable.                       |
| Backtest invalid        | Leakage/gaps/insufficient sample; no misleading result.                         |
| Options no quote        | Unavailable stays unavailable, never 0.                                         |
| Cross-venue unavailable | Dispersion/consensus undefined, not zero.                                       |
| Historical gap          | Visible in Replay timeline and data quality.                                    |

# 22. Native/Desktop требования

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Адаптация к новой цели</strong></p>
<p>Ниже — требования, логически вытекающие из решения строить не web-only продукт. Они не меняют продуктовую семантику, а используют преимущества desktop/native оболочки.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

> • Desktop-first full-screen workstation with hardware-accelerated chart/heatmap rendering.
>
> • OS-level hotkeys, window focus and low-latency input for scalping.
>
> • Detachable DOM/Tape/Chart windows and multi-monitor groups as post-MVP capability.
>
> • Background market-data collectors continue while individual screens change.
>
> • Encrypted local cache for replay/history/workspaces; secure OS credential store for exchange secrets.
>
> • Crash recovery restores last workspace and pre-AI/pre-layout snapshots.
>
> • System notifications for Watches/critical risk events with per-event severity and mute policies.
>
> • Offline/replay mode remains usable when live connectors are unavailable where local data exists.
>
> • No dependency on browser route semantics; module navigation uses application state and stable object IDs.

# 23. Security, permissions и safeguards

> • Separate read/write/execute permissions.
>
> • No withdrawal permission for trading agents.
>
> • Human approval for live orders by default.
>
> • Block automation on stale data/exchange degradation/risk-policy violation.
>
> • Full audit trail and rollback/kill.
>
> • API keys stored in dedicated secure secret storage, not workspace exports or journal files.
>
> • Tool/prompt injection defenses for agent workflows and external content.
>
> • Explicit environment indicator on every money-moving action.
>
> • Live multi-leg options execution requires liquidity/risk preview.

# 24. Что продукт принципиально НЕ делает

> • Не заменяет рыночные данные LLM-оценкой.
>
> • Не называет model estimate «точной позицией маркет-мейкеров».
>
> • Не показывает точную вероятность прибыли/confidence без калиброванной модели.
>
> • Не смешивает FACT, CALCULATION, INFERENCE, CONFIRMATION и SCENARIO.
>
> • Не скрывает missing/stale/partial data.
>
> • Не меняет workspace/nav и не выполняет live order молча.
>
> • Не удаляет invalidated/legacy research history для соответствия новой схеме.
>
> • Не превращает каждое market tick/change в AI notification.
>
> • Не плодит AI Coach/Options AI/Quant AI как независимые конкурирующие продукты — один runtime, разные contextual surfaces.
>
> • Не возвращает глобальный View selector, Pane Set или несколько competing layout abstractions.
>
> • Не обещает future backend через работающие production-like кнопки.

# 25. Сквозные пользовательские workflows

## 25.1. Ask → UI → Thesis → Watch

> **1.** Пользователь нажимает ⌘K и спрашивает «Почему BTC не проходит 66k?».
>
> **2.** AI parses intent, отвечает на выбранной глубине и показывает grouped chart changes.
>
> **3.** Пользователь применяет изменения; chart focuses level, opens CVD/VWAP/Γ context. Undo available.
>
> **4.** AI предлагает/создаёт Thesis с explicit confirmation and invalidation.
>
> **5.** Пользователь arms deterministic Watch.
>
> **6.** Market state changes; Event Engine triggers CONFIRMED/WEAKENING/INVALIDATED or data pause.
>
> **7.** AI Events compresses relevant changes; trader decides next step.

## 25.2. Выделить участок графика → Ask AI

> **1.** Пользователь выделяет time×price region.
>
> **2.** Mini action bar появляется рядом с selection.
>
> **3.** Ask AI прикрепляет region context в Chat/Command Bar.
>
> **4.** AI объясняет только то, что поддержано data, и маркирует taxonomy.
>
> **5.** Пользователь может Compare, Create alert/watch, Turn into rule, Add to journal or build thesis.

## 25.3. Options Scenario → Underlying → Structure

> **1.** Options Scenarios формирует weighted branches and invalidations.
>
> **2.** Watch on underlying opens Terminal with transferred thesis/focus and highlights level.
>
> **3.** Order-flow confirmation strengthens/weakens the scenario.
>
> **4.** Build fitting structure opens Strategy Lab with expiry/range/strikes/expected move prefilled.
>
> **5.** User paper trades / sends to Quant Lab / writes to Journal.

## 25.4. Beginner Guided Replay

> **1.** Tutor presents event without outcome.
>
> **2.** User states hypothesis.
>
> **3.** AI reveals evidence/competing view.
>
> **4.** User chooses Wait/Skip/Paper trade.
>
> **5.** Outcome and reasoning review appear.
>
> **6.** Journal stores process score and next exercise.

## 25.5. Quant idea → Agent

> **1.** User writes idea in natural language.
>
> **2.** AI formalizes spec.
>
> **3.** Run against chosen dataset with realistic execution assumptions.
>
> **4.** Validator checks OOS/leakage/overfitting/regimes.
>
> **5.** Create Paper Agent.
>
> **6.** Promote only through policy gates to Testnet/Live.

# 26. Рекомендуемый build order

| **Stage**                | **Scope**                                                                                                                             |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| A. Shared Core           | Market/event schemas, chart/order-flow preservation, workspace/layout engine, persistence, data-quality model, replay clock.          |
| B. Terminal + AI Shell   | Scalper, Beginner, AI Event, Command Bar, contextual selection, Thesis/Watch, Journal integration, Risk strip.                        |
| C. Options Phase 1       | Deribit collector, Chain, Volatility, Gamma Concentration/Classical GEX/Observed Flow, Overview, Terminal overlays, Replay recording. |
| D. Options Advanced      | Scenarios, Strategy Lab, Positions, Scalping, Estimated Inventory, cross-venue Binance/Bybit/OKX.                                     |
| E. Quant Lab             | Natural-language spec, data catalog, realistic backtest, validator, job/log/result UX.                                                |
| F. Agents/B2A            | Agent Builder/Console, policies, approvals, paper/testnet, API contracts.                                                             |
| G. Controlled Automation | L4 suggestions → L5 paper automation → future L6 live with explicit policy and auditing.                                              |

> • До Options/Quant вынести shared core так, чтобы новые modules не дублировали AI/runtime/data/chart contracts.

# 27. Master acceptance criteria

> • Global navigation ясно разделяет Terminal / Options / Quant Lab / Journal / Settings.
>
> • Terminal имеет один Workspace selector; глобального View нет.
>
> • Существующие heatmap/footprint/DOM/tape/CVD/profiles сохранены.
>
> • Chart selection → contextual AI actions работает в Live и Replay.
>
> • ASK ⌘K умеет parse intent, preview UI actions, apply and undo.
>
> • AI может честно ответить unknown/insufficient data.
>
> • AI Event имеет единый contract и одинаковые actions.
>
> • Thesis lifecycle и Watch conditions deterministic; data degradation не считается weakening.
>
> • AI never calculates/invents market metrics; taxonomy visible on demand.
>
> • Beginner Tutor имеет Lesson/Ask/History, voice state and no-hindsight live mode.
>
> • Scalper сохраняет low-latency dense order-flow workflow, hotkeys and risk kill switch.
>
> • Replay preserves layout and uses one unified clock.
>
> • Journal restores reproducible market/chart state and stores thesis history.
>
> • Options реализует Overview/Chain/Gamma/Flow/Volatility/Scenarios/Replay/Strategy Lab/Positions/Scalping.
>
> • Options overlays and scenarios confirm on underlying Terminal.
>
> • Exact dealer GEX is never presented as observed fact.
>
> • Quant workflow проходит Idea→Spec→Test→Validate→Deploy and exposes invalid backtests.
>
> • Personal Agents have goal/tools/trigger/policy/approval/audit.
>
> • Live execution is explicit, permissioned and risk-gated.
>
> • All missing/stale/partial/disconnected states are visible and cannot silently feed automation.
>
> • Every active control causes visible result or explicit disabled reason.
>
> • Product remains fully usable manually when AI is unavailable.

# Приложение A. Полный перечень экранов и major states

| **Модуль**     | **Экраны/состояния**                                                                                                           |
|----------------|--------------------------------------------------------------------------------------------------------------------------------|
| Access         | Sign in, account creation, 2FA, exchange connection, permissions.                                                              |
| Onboarding     | Goals, experience, style, data, AI, risk, options/automation, workspace preview.                                               |
| Terminal       | Beginner Learning, Scalper Execution, Custom; chart focus/volume/full-flow layout presets; Live/Replay; Customize; Add Widget. |
| AI             | Command Bar, Events, Chat, Agent, History, Thesis detail, Watch detail, unknown, data degraded, legacy.                        |
| Journal        | Timeline/list, entry detail, filters, restore chart, AI review, playbook/process views.                                        |
| Risk/Execution | Ticket, positions, active orders, Risk Center, policy blocked, execution report, kill switch.                                  |
| Options        | Overview, Chain, Strike Inspector, Gamma, Flow, Volatility, Scenarios, Replay, Strategy Lab, Positions, Scalping.              |
| Quant Lab      | Idea/spec, data catalog, editor, run/job, results, validation, regimes, logs, deployment.                                      |
| Agents         | Builder, Console, approvals, policy, audit, usage/budget.                                                                      |
| Settings       | Profile, workspaces, AI, notifications, connections/data, safeguards, hotkeys, display, security, billing.                     |

# Приложение B. AI / Thesis / Watch state map

| **\#** | **State**               | **Rule**                                                      |
|--------|-------------------------|---------------------------------------------------------------|
| 01     | Terminal at rest        | Chart owns screen; compact workspace/live/AI status only.     |
| 02     | Command Bar open        | Temporary Ask surface + examples/ordinary commands.           |
| 03     | Parsed intent           | Intent + cautious headline + Simple/Trader/Quant.             |
| 04     | Unknown intent          | No thesis/actions/watch; examples and manual commands remain. |
| 05     | Grouped Will Change     | Focus / Evidence / Options Context groups.                    |
| 06     | Navigation confirmation | Explicit confirmation for workspace/page switch.              |
| 07     | Applied + Undo          | Exact pre-AI state recoverable.                               |
| 08     | Thesis FORMING          | Evidence/conditions not yet sufficient for ACTIVE.            |
| 09     | Thesis ACTIVE collapsed | State visible, report hidden.                                 |
| 10     | Thesis ACTIVE expanded  | Focus, confirmation, invalidation, source, watches.           |
| 11     | Why? evidence           | FACT / CALCULATION / INFERENCE / CONFIRMATION / SCENARIO.     |
| 12     | Watch predicate         | Human phrase + deterministic conditions.                      |
| 13     | Watch triggered         | One event, observation not instruction.                       |
| 14     | Monitoring paused       | ACTIVE + DATA DEGRADED; watches paused.                       |
| 15     | Data restored           | Resume from current reading, gap remains a gap.               |
| 16     | CONFIRMED               | Pre-named confirmation condition fired.                       |
| 17     | WEAKENING               | Compressed event with reasons.                                |
| 18     | INVALIDATED             | Overlays muted, nothing silently removed.                     |
| 19     | RESOLVED                | Normal closure: target/session/expiry/context finished.       |
| 20     | LEGACY / MONITORING OFF | History preserved; rebuild/keep actions.                      |
| 21     | Options · Understand    | Same Thesis object in Options context.                        |
| 22     | Strategy Lab · Express  | Built from thesis; expiry/range/strikes transferred.          |
| 23     | Quant Lab · Test        | Testing thesis with sample scope/venue caveats.               |
| 24     | Journal · Record        | Decision/outcome and lifecycle history linked.                |

# Приложение C. Exhaustive feature inventory

| **Module**          | **Feature**                                     | **Status**            |
|---------------------|-------------------------------------------------|-----------------------|
| Global Shell        | Global module navigation                        | Required              |
| Global Shell        | Workspace selector                              | Required              |
| Global Shell        | Live/Replay                                     | Required              |
| Global Shell        | Environment selector                            | Required              |
| Global Shell        | Risk status                                     | Required              |
| Global Shell        | AI autonomy level                               | Required              |
| Global Shell        | ASK ⌘K                                          | Required              |
| Global Shell        | Account/latency/data quality                    | Required              |
| Workspace           | System workspaces                               | Required              |
| Workspace           | My Workspaces                                   | Required              |
| Workspace           | Layout presets                                  | Required              |
| Workspace           | Region layout                                   | Required              |
| Workspace           | Tabs/stacks/splits                              | Required              |
| Workspace           | Maximize/collapse/focus                         | Required              |
| Workspace           | Link groups                                     | Required              |
| Workspace           | Autosave                                        | Required              |
| Workspace           | Snapshots                                       | Required              |
| Workspace           | Undo/redo                                       | Required              |
| Workspace           | Import/export                                   | Required              |
| Terminal Chart      | Heatmap                                         | Required              |
| Terminal Chart      | Footprint                                       | Required              |
| Terminal Chart      | Candles                                         | Required              |
| Terminal Chart      | Trade bubbles                                   | Required              |
| Terminal Chart      | VWAP/Anchored VWAP                              | Required              |
| Terminal Chart      | POC/VAH/VAL/HVN/LVN                             | Required              |
| Terminal Chart      | Drawings/zones                                  | Required              |
| Terminal Chart      | Synchronized crosshair                          | Required              |
| Terminal Chart      | Instrument tabs                                 | Required              |
| Terminal Chart      | Gamma/Options overlays                          | Required              |
| Order Flow          | DOM/COB                                         | Required              |
| Order Flow          | Tape                                            | Required              |
| Order Flow          | CVD                                             | Required              |
| Order Flow          | Bid/Ask                                         | Required              |
| Order Flow          | Delta                                           | Required              |
| Order Flow          | SVP                                             | Required              |
| Order Flow          | CVP                                             | Required              |
| Order Flow          | Liquidity pull/add                              | Required              |
| Order Flow          | Absorption                                      | Required              |
| Order Flow          | Replenishment                                   | Required              |
| Order Flow          | Sweeps                                          | Required              |
| Order Flow          | Failed breakouts                                | Required              |
| Order Flow          | Liquidations                                    | Required              |
| Order Flow          | Divergences                                     | Required              |
| Contextual AI       | Region selection                                | Required              |
| Contextual AI       | Mini action bar                                 | Required              |
| Contextual AI       | Explain                                         | Required              |
| Contextual AI       | Compare                                         | Required              |
| Contextual AI       | Ask AI                                          | Required              |
| Contextual AI       | Create alert                                    | Required              |
| Contextual AI       | Turn into rule                                  | Required              |
| Contextual AI       | Add to journal                                  | Required              |
| Contextual AI       | Attach structured context                       | Required              |
| Contextual AI       | Widget-level Explain                            | Required              |
| Contextual AI       | DOM explain                                     | Required              |
| Contextual AI       | Ticket risk explain                             | Required              |
| AI Shell            | Command Bar                                     | Required              |
| AI Shell            | Parsed intent                                   | Required              |
| AI Shell            | Unknown intent                                  | Required              |
| AI Shell            | Grouped UI changes                              | Required              |
| AI Shell            | Preview/apply                                   | Required              |
| AI Shell            | Navigation confirmation                         | Required              |
| AI Shell            | Undo                                            | Required              |
| AI Shell            | AI Events                                       | Required              |
| AI Shell            | Chat                                            | Required              |
| AI Shell            | Agent tab                                       | Required              |
| AI Shell            | History                                         | Required              |
| AI Shell            | Simple/Trader/Quant                             | Required              |
| AI Shell            | Evidence taxonomy                               | Required              |
| AI Shell            | Event compression                               | Required              |
| Thesis/Watch        | Thesis persistent object                        | Required              |
| Thesis/Watch        | FORMING                                         | Required              |
| Thesis/Watch        | ACTIVE                                          | Required              |
| Thesis/Watch        | CONFIRMED                                       | Required              |
| Thesis/Watch        | WEAKENING                                       | Required              |
| Thesis/Watch        | INVALIDATED                                     | Required              |
| Thesis/Watch        | RESOLVED                                        | Required              |
| Thesis/Watch        | Legacy thesis                                   | Required              |
| Thesis/Watch        | Watch compiler                                  | Required              |
| Thesis/Watch        | Visible predicates                              | Required              |
| Thesis/Watch        | Watch pause on stale data                       | Required              |
| Thesis/Watch        | Watch triggered event                           | Required              |
| Beginner            | Guided Replay                                   | Required              |
| Beginner            | Live Tutor                                      | Required              |
| Beginner            | Review                                          | Required              |
| Beginner            | Lesson/Ask/History                              | Required              |
| Beginner            | Hypothesis-first lesson                         | Required              |
| Beginner            | Competing interpretation                        | Required              |
| Beginner            | Paper decision                                  | Required              |
| Beginner            | Process score                                   | Required              |
| Beginner            | Voice input                                     | Required              |
| Beginner            | Similar episodes                                | Required              |
| Beginner            | No-hindsight live                               | Required              |
| Scalper             | Dense execution layout                          | Required              |
| Scalper             | Fast Ticket                                     | Required              |
| Scalper             | Hotkeys                                         | Required              |
| Scalper             | Brackets                                        | Required              |
| Scalper             | Reduce-only                                     | Required              |
| Scalper             | Flatten                                         | Required              |
| Scalper             | Cancel all                                      | Required              |
| Scalper             | Risk lock/Kill switch                           | Required              |
| Scalper             | Compact AI Events                               | Required              |
| Scalper             | Options context                                 | Required              |
| Replay              | Unified clock                                   | Required              |
| Replay              | Play/pause/step/speed                           | Required              |
| Replay              | Event jump                                      | Required              |
| Replay              | Return live                                     | Required              |
| Replay              | Replay Library                                  | Required              |
| Replay              | Recorded gaps                                   | Required              |
| Replay              | AI replay context                               | Required              |
| Replay              | Options replay sync                             | Required              |
| Journal             | Global screen                                   | Required              |
| Journal             | Dockable widget                                 | Required              |
| Journal             | Filters                                         | Required              |
| Journal             | Thesis/evidence/action/outcome                  | Required              |
| Journal             | AI review                                       | Required              |
| Journal             | Process score                                   | Required              |
| Journal             | Restore chart state                             | Required              |
| Journal             | Thumbnail                                       | Required              |
| Journal             | Session summary                                 | Required              |
| Journal             | Playbook/mistake analytics                      | Required              |
| Risk/Execution      | Read-only                                       | Required              |
| Risk/Execution      | Paper                                           | Required              |
| Risk/Execution      | Testnet                                         | Required              |
| Risk/Execution      | Live                                            | Required              |
| Risk/Execution      | Fees/slippage                                   | Required              |
| Risk/Execution      | Partial fills                                   | Required              |
| Risk/Execution      | Latency                                         | Required              |
| Risk/Execution      | Queue estimate                                  | Required              |
| Risk/Execution      | Execution report                                | Required              |
| Risk/Execution      | Risk Center                                     | Required              |
| Risk/Execution      | Stress tests                                    | Required              |
| Risk/Execution      | Policy blocks                                   | Required              |
| Risk/Execution      | Kill bot/agent                                  | Required              |
| Options Overview    | Spot/index                                      | Required              |
| Options Overview    | Expected move                                   | Required              |
| Options Overview    | ATM IV                                          | Required              |
| Options Overview    | IV-RV                                           | Required              |
| Options Overview    | Gamma regime                                    | Required              |
| Options Overview    | Key levels                                      | Required              |
| Options Overview    | Evidence/history                                | Required              |
| Options Overview    | AI Analyst                                      | Required              |
| Options Overview    | Data Quality                                    | Required              |
| Options Chain       | Calls/puts ladder                               | Required              |
| Options Chain       | Beginner/Flow/Greeks/Liquidity/Research presets | Required              |
| Options Chain       | Quote/IV/Greeks/OI/ΔOI/flow                     | Required              |
| Options Chain       | Heatmaps                                        | Required              |
| Options Chain       | Filters                                         | Required              |
| Options Chain       | No quote state                                  | Required              |
| Options Chain       | Strike Inspector                                | Required              |
| Options Chain       | Multi-select                                    | Required              |
| Options Gamma       | Gamma Concentration                             | Required              |
| Options Gamma       | Classical GEX                                   | Required              |
| Options Gamma       | Observed Gamma Flow                             | Required              |
| Options Gamma       | Estimated Inventory                             | Required              |
| Options Gamma       | Gamma Change                                    | Required              |
| Options Gamma       | Cross-Venue Consensus                           | Required              |
| Options Gamma       | Zero Gamma                                      | Required              |
| Options Gamma       | Profile                                         | Required              |
| Options Gamma       | Time×Price heatmap                              | Required              |
| Options Gamma       | Level inspector                                 | Required              |
| Options Gamma       | Interactive lookback/buckets/filters            | Required              |
| Options Flow        | Big flow bubbles                                | Required              |
| Options Flow        | Option CVD                                      | Required              |
| Options Flow        | Flow tape                                       | Required              |
| Options Flow        | Blocks/RFQ                                      | Required              |
| Options Flow        | Structure inference                             | Required              |
| Options Flow        | Trade→underlying sync                           | Required              |
| Options Volatility  | IV Surface                                      | Required              |
| Options Volatility  | Smile/skew                                      | Required              |
| Options Volatility  | Risk reversal                                   | Required              |
| Options Volatility  | Butterfly                                       | Required              |
| Options Volatility  | Term structure                                  | Required              |
| Options Volatility  | Expected move                                   | Required              |
| Options Volatility  | IV vs RV                                        | Required              |
| Options Volatility  | Percentiles                                     | Required              |
| Options Scenarios   | Scenario weights                                | Required              |
| Options Scenarios   | Residual case                                   | Required              |
| Options Scenarios   | Evidence chain                                  | Required              |
| Options Scenarios   | Target zone                                     | Required              |
| Options Scenarios   | Invalidation                                    | Required              |
| Options Scenarios   | What changed                                    | Required              |
| Options Scenarios   | Analyst log                                     | Required              |
| Options Scenarios   | Watch on underlying                             | Required              |
| Options Scenarios   | Build fitting structure                         | Required              |
| Strategy Lab        | Structure templates                             | Required              |
| Strategy Lab        | Payoff today/expiry                             | Required              |
| Strategy Lab        | Expected move overlay                           | Required              |
| Strategy Lab        | Underlying/IV/time shocks                       | Required              |
| Strategy Lab        | Greeks now/scenario                             | Required              |
| Strategy Lab        | Compare structures                              | Required              |
| Strategy Lab        | Risk check                                      | Required              |
| Strategy Lab        | Paper/live/Quant/Journal actions                | Required              |
| Options Positions   | Total/realized/unrealized P&L                   | Required              |
| Options Positions   | Margin                                          | Required              |
| Options Positions   | Portfolio Greeks                                | Required              |
| Options Positions   | Book payoff                                     | Required              |
| Options Positions   | Stress/shocks                                   | Required              |
| Options Positions   | Risk by contract                                | Required              |
| Options Positions   | Expiry concentration                            | Required              |
| Options Positions   | Greeks by expiry/strike                         | Required              |
| Options Scalping    | Underlying + option context                     | Required/Future depth |
| Options Scalping    | Strikes in play                                 | Required/Future depth |
| Options Scalping    | Contract liquidity                              | Required/Future depth |
| Options Scalping    | Flow tape                                       | Required/Future depth |
| Options Scalping    | Option CVD                                      | Required/Future depth |
| Options Scalping    | Quick ticket                                    | Required/Future depth |
| Options Scalping    | Underlying DOM + Option DOM                     | Required/Future depth |
| Options Scalping    | IV tick/theoretical value/hedge panel           | Required/Future depth |
| Quant Lab           | NL idea                                         | Required              |
| Quant Lab           | Strategy Spec                                   | Required              |
| Quant Lab           | Clarification questions                         | Required              |
| Quant Lab           | Dataset catalog                                 | Required              |
| Quant Lab           | Editor/DSL                                      | Required              |
| Quant Lab           | Visible run job                                 | Required              |
| Quant Lab           | Signals/fills/rejected opportunities            | Required              |
| Quant Lab           | Metrics                                         | Required              |
| Quant Lab           | OOS/validation                                  | Required              |
| Quant Lab           | Regimes                                         | Required              |
| Quant Lab           | Logs                                            | Required              |
| Quant Lab           | Deployment                                      | Required              |
| Quant Lab           | Multi-run compare                               | Required              |
| Agents/B2A          | Agent Builder                                   | Required/Future       |
| Agents/B2A          | Agent Console                                   | Required/Future       |
| Agents/B2A          | Tools/permissions                               | Required/Future       |
| Agents/B2A          | Triggers                                        | Required/Future       |
| Agents/B2A          | Policies                                        | Required/Future       |
| Agents/B2A          | Approvals                                       | Required/Future       |
| Agents/B2A          | Budgets/cost                                    | Required/Future       |
| Agents/B2A          | Audit                                           | Required/Future       |
| Agents/B2A          | Market Context API                              | Required/Future       |
| Agents/B2A          | Event Stream                                    | Required/Future       |
| Agents/B2A          | Sandbox                                         | Required/Future       |
| Agents/B2A          | Execution Gateway                               | Required/Future       |
| Agents/B2A          | Risk Policy Engine                              | Required/Future       |
| Agents/B2A          | Audit API                                       | Required/Future       |
| Agents/B2A          | Marketplace                                     | Required/Future       |
| Settings/Onboarding | Adaptive onboarding                             | Required              |
| Settings/Onboarding | Profile                                         | Required              |
| Settings/Onboarding | Workspace settings                              | Required              |
| Settings/Onboarding | AI preferences                                  | Required              |
| Settings/Onboarding | Notifications                                   | Required              |
| Settings/Onboarding | Data/connections                                | Required              |
| Settings/Onboarding | Safeguards                                      | Required              |
| Settings/Onboarding | Hotkeys                                         | Required              |
| Settings/Onboarding | Display/accessibility                           | Required              |
| Settings/Onboarding | Security/2FA                                    | Required              |
| Settings/Onboarding | Billing                                         | Required              |
| Native Desktop      | Hardware-accelerated chart                      | Native target         |
| Native Desktop      | OS hotkeys                                      | Native target         |
| Native Desktop      | Detachable windows                              | Native target         |
| Native Desktop      | Multi-monitor                                   | Native target         |
| Native Desktop      | Secure credential store                         | Native target         |
| Native Desktop      | Encrypted local cache                           | Native target         |
| Native Desktop      | Crash recovery                                  | Native target         |
| Native Desktop      | OS notifications                                | Native target         |
| Native Desktop      | Offline replay                                  | Native target         |

# Приложение D. Глоссарий

| **Термин**          | **Определение**                                                                               |
|---------------------|-----------------------------------------------------------------------------------------------|
| AI Event            | Стандартизированное рыночное событие, а не отдельный popup type.                              |
| Thesis              | Гипотеза с evidence, conditions и lifecycle.                                                  |
| Watch               | Исполняемый deterministic predicate для мониторинга.                                          |
| Scenario Weight     | Вес ветки модели; не probability, пока нет калибровки.                                        |
| Gamma Concentration | Unsigned gamma sensitivity concentration.                                                     |
| Classical GEX       | Signed model estimate under assumptions.                                                      |
| Observed Gamma Flow | Aggressor-signed executed gamma flow.                                                         |
| Zero Gamma          | Model boundary where modeled exposure changes sign.                                           |
| CVD                 | Cumulative aggressive buy volume minus aggressive sell volume.                                |
| VWAP                | Volume-weighted average execution price over defined scope.                                   |
| Workspace           | Saved working environment.                                                                    |
| Layout Preset       | Layout variant inside one Workspace.                                                          |
| Evidence taxonomy   | FACT / CALCULATION / INFERENCE / CONFIRMATION / SCENARIO.                                     |
| Data degraded       | System cannot reliably evaluate a condition; thesis state is preserved and monitoring pauses. |
| Replay gap          | Recorded absence of data, never silently interpolated by AI.                                  |

# Приложение E. Артефакты-источники и решения, сведённые в master spec

> • Odelix_Описание_проекта_RU.docx — product vision, AI/event/tutor/order-flow/quant/B2A principles.
>
> • Odelix_Project_and_UXUI_Spec_RU.docx — product + UX/UI requirements, screen/widget/risk/options/quant inventories.
>
> • Odelix_ТЗ_UXUI_Claude_Design_RU.docx — refined workspace/AI/tutor/scalper/options/quant interactions and acceptance criteria.
>
> • Odelix_DeepGamma_Crypto_Options_Research.docx — DeepGamma teardown, data-source limitations and gamma model methodology.
>
> • Odelix_Options_UI_UX_Spec.docx — detailed Options IA, chain/gamma/flow/volatility/scenarios/terminal integration.
>
> • Odelix Pre-build Review v2.1.html — preservation audit, final Stage 1A region-layout and system workspaces.
>
> • Odelix UXUI Refactor Plan (2).html — contextual Ask AI, AI Event unification, Journal restore state, Options/Quant wireframes.
>
> • ff6fab38-0a12-4d62-abab-60a9715950ac.html — latest AI-native UX pass and state storyboard.
>
> • Последующие review-решения в диалоге: remove global View, Layout Presets in Customize, Top-3 Γ overlays, Options↔Terminal integration, AI Command Bar, Thesis/Watch lifecycle, calm-by-default UX and non-web/native target.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Итог</strong></p>
<p>Этот master spec должен использоваться как baseline при проектировании новой native/desktop версии. Новые решения добавляются поверх него только как явно версионированные изменения, а не как параллельные competing UX models.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>


## Options / Strategy Lab activation

Options Desk / Strategy Lab supports the shared strategy catalogue and custom experiments.
Research and PAPER activate by their data/compute/policy dependencies; a first payment is not
a technical prerequisite. Full Workstation is a later, deeper surface.
The extensive inventory below is not the first-release checklist. Market supplies
IV/Greeks/payoff, Product objectives/evidence, and Execution signed quote/policy/
finality after activation. Preserve Simple/Trader/Quant projections and the same
Select→Ask→Evidence→Thesis references. Show residual portfolio loss, total fees,
indicative/firm status and finality; no browser balance or LLM arithmetic authority.
Optional WebMCP is a read/compute/draft capability projection, not another agent
or replacement for external remote MCP. See ../../odelix-product/docs/OPTIONS-WORKFLOW.md.

## Полнота совместной среды r12

Полный перенесённый workstation catalog и shared runtime behavior: [WORKSTATION-SPEC.md](DEVELOPMENT.md#DEP-568bec8dbe). Исторические platform/timeline/vendor statements выше — исходный материал; новые решения определяются актуальными owner docs и единым roadmap.

## Уточнение Quant Lab: пользовательские зависимости

Обязательный UX — выбрать доступные sources, создать/изменить formula, увидеть dependency view и time/quality assumptions, отдельно отредактировать entry/exit и сравнить experiments. Для relationship-only исследования позиция и execution ticket не требуются. NLP/формула/таблица правил ведут к одному draft; visual node editor не блокирует первый релиз. Историческое срабатывание открывается с values каждого условия и объяснением entry/exit/skip; смена exit создаёт новую сравнимую версию. Это работа над существующими объектами, не обязательный новый экран или runtime.


## Согласование r13

Полная web-native модель сохраняется. Внешний terminal не становится встроенным iframe или новой бизнес-логикой. UI может редактировать inputs/features/условия независимого entry/exit, но не получает прямых внешних ключей и не обходит Context/DataCapability/owner APIs. Из code-review источников в интерфейс передаются только разрешённые typed results; supplier-specific types и actions не являются пользовательским контрактом.

Подробности и статусы: [единый реестр заимствования](DEVELOPMENT.md#DEP-52ea262d35), [проверка исходников](DEVELOPMENT.md#DEP-143d61eff7).


## Актуализация r15 — рабочий flow и проверяемая семантика

Использовать WKS shared packages и user AlphaQuant interactions. Не считать Workspaces HTML единственным источником Options depth, не переносить sample detectors/probabilities как готовый backend.

Подробная обязательная спецификация: [PROTOTYPE-MIGRATION.md](#prototype-migration). Старые утверждения о полноте прототипа и исторические оценки сроков не являются runtime evidence.



---
<a id="prototype-migration"></a>
## AlphaQuant → Odelix: что переносить из UI prototype


**1.0.0 · 2026-09-20.** Исходный пользовательский ZIP сохранён в `DEP-90bbeb40e2` (см. локальный реестр зависимостей), полный список members и hashes — [PROTOTYPE-SOURCES](DEVELOPMENT.md#DEP-14e5071e3a). ZIP является источником дизайна, не новым runtime. Права/технические зависимости проверяются перед переносом.

### Приоритет источников

`AlphaQuant Workspaces.dc.html` задаёт Observe/Analyze/Build/Learn и общую рабочую композицию; это не замена всех предыдущих Options функций. `AlphaQuant Options.dc.html` и `AlphaQuant Options Workflow Storyboard.dc.html` сохраняют отдельные Flow/Chain/Volatility/Strike и переходы в underlying. `AlphaQuant AI-native Storyboard.dc.html` сохраняет намерение/контекст/Preview/Apply/Undo. Stage 1B и shared modules полезны как donor взаимодействий.

`BACKEND_REQUIREMENTS.md`, `FEATURES.md`, `HANDOFF.md` прочитаны как прототипные требования. Их built/done означает статус прототипа, не реализации Odelix. Утверждение старого файла «source of truth — Stage 1B» не отменяет поздний Workspaces design: взаимодействия сопоставляются по feature, спорные изменения решаются отдельно. Требование mirror shared contracts интерпретируется как совместимый adapter к producer-owned Odelix schemas, не второй канон JS-контрактов.

### Source → destination → Issue

| Донор | Перенос | Владелец/Issue |
|---|---|---|
| Workspaces HTML | Навигация задач, chart cluster, профиль vs workspace, status axes | WKS-011, WEB-003; общие WKS packages |
| Stage 1B / shared | Выделение области и точный instrument/time/context | PRD-027, WKS-001/002/012 |
| AI-native storyboard | Observation/Interpretation/Alternative, Preview/Apply/Undo | WKS-013, PRD-011/028 |
| Options HTML / workflow storyboard | Flow tape, strike/expiry, linked underlying, source quality | MKT-023/024, WKS-009 |
| shared/similarity.js | Объяснимый UX сигнатуры и совпадений | MKT-026, WKS-015 |
| Backend Requirements / Features | Coverage checklist конкретных fields и interactions | Phase-A requirements + acceptance owner Issues |
| Quant Lab | Независимые entry/exit, связь с research | PRD-015…019, WEB-005…008, не built-in named strategy |

### Что не переносить как готовую аналитику

Синтетические prices/returns/probabilities, заранее подготовленные outcomes, sample-only detectors/refill heuristics и UI-generated risk. Статусы data/connection/maturity сохранить, но не скрывать sampling/unsupported: честность должна быть видна у конкретного значения. Research prototypes не доказывают PIT dataset или прибыльность. Similarity distance не forecast probability.

### React/TypeScript интеграция

Извлечь компактные interaction/formatting функции только после сопоставления contracts и тестов. Не грузить целый HTML как production приложение и не внедрять глобальный mutable `window.AQ` в domain. Web/App composition использует общие WKS packages; Product Fastify — единственный пользовательский backend. Старый chart cluster не дробить на несогласованные оси ради свободного docking.

### Приёмка

Для каждой переносимой feature: source member hash → destination → Issue → UI behavior fixture. Нужны реальные backend fields; отсутствие означает disabled/fixture, не invented live. Смена режима сохраняет selection; read-only first. Клик по кнопке prototype не является тестом финансовой операции.

<a id="external-data-preview"></a>
## External-data preview в browser surface

Используются общие WKS source-aware graph/Inspector/chain components; отдельный provider SDK в Web не создаётся. Local preview доступен на ограниченных настоящих external observations до полного native Market, оплаты и Desktop. Он не заменяет gates native flow и не активирует public access.

Каждый слой сохраняет source/dataset/receipt/revision и units/time/quality. Public route/Share/Ask/Export повторно проверяют Product rights. Source switch не обновляет чужую Evidence; refresh — новая revision, offline reopen — сохранённый ответ без vendor call. Полные правила UI находятся в Workstation spec через локальный dependency artifact; не дублировать их реализацию.
<!-- R159 staged-user-ui -->
## Hosted continuity и staged Agent UI

Ранний Watch editor независим от Paper Journal/полного Desk. Flight Recorder отображает импортированные события до подключения собственной модели; Fleet/Approval отдельно от Builder. History показывает unavailable/redacted, не придумывает inputs. Marketplace/B2A distribution и реальные среды остаются отдельной стратегической веткой.

Источник данных привязан к слою: OpenBB preview, native Deribit и FRED vintages имеют различную точность, coverage и права. Полная data-программа не делает любой слой доступным; current external preview не исключает будущую полную историю.
