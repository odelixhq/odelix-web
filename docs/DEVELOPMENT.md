# odelix-web — руководство разработчика

**r15.11 · 2026-09-28 · PROPOSED / не статус runtime.** В этом файле объединены локальная архитектура, контракты, карта модулей, тесты и выпуск. Подробные предметные спецификации сохранены отдельно.

[Вход в repo](../AGENTS.md) · [Локальная выборка задач](../delivery/CONTEXT.json)


<!-- BEGIN GENERATED IMPLEMENTATION BASIS -->
<a id="implementation-basis"></a>
## Основа реализации: что берём, что пишем и где интегрируем

**r15.11. Основа — React/TypeScript browser host и общие Workstation packages. Здесь account/route/share/UI composition; не ещё один terminal engine, не отдельные численные модели. External preview использует общие WKS-компоненты, без второго data-router.**

Это локальная генерируемая выборка общего решения, а не отдельный редактируемый реестр. Полные metadata — `delivery/CONTEXT.json → implementation_blueprint`. Изменения предлагает владелец repo через Coordinator; генератор обновляет общий и локальные виды вместе. Конкретный upstream/pin и supply-chain проверяются перед включением, не по наличию названия в таблице.

**Три разных зависимости:** библиотека/внешний код; внешний сервис/данные; контракт другого Odelix repo. Последний не разрешает копировать чужую реализацию. Ниже сами архитектурные bindings; реестр документальных источников в конце файла — другая сущность.

| Источник | Режим | Модули этого repo | Задачи |
|---|---|---|---|
| [OIDC](#reuse-BND-042) | SERVICE_ADAPTER | AREA-WEB-SHELL | [ODX-WEB-001](#issue-ODX-WEB-001) |
| [REACT](#reuse-BND-043) | LIBRARY | AREA-WEB-ACCOUNT, AREA-WEB-ASK, AREA-WEB-DESK, AREA-WEB-JOURNAL, AREA-WEB-SHELL, AREA-WEB-WATCH, WEB-STRATEGY-LAB | [ODX-WEB-001](#issue-ODX-WEB-001), [ODX-WEB-002](#issue-ODX-WEB-002), [ODX-WEB-003](#issue-ODX-WEB-003), [ODX-WEB-004](#issue-ODX-WEB-004), [ODX-WEB-005](#issue-ODX-WEB-005), [ODX-WEB-006](#issue-ODX-WEB-006), [ODX-WEB-007](#issue-ODX-WEB-007), [ODX-WEB-008](#issue-ODX-WEB-008), [ODX-WEB-009](#issue-ODX-WEB-009), [ODX-WEB-010](#issue-ODX-WEB-010) |
| [TANSTACK](#reuse-BND-044) | LIBRARY | AREA-WEB-ASK, AREA-WEB-DESK, AREA-WEB-SHELL, WEB-STRATEGY-LAB | [ODX-WEB-001](#issue-ODX-WEB-001), [ODX-WEB-002](#issue-ODX-WEB-002), [ODX-WEB-003](#issue-ODX-WEB-003), [ODX-WEB-005](#issue-ODX-WEB-005), [ODX-WEB-007](#issue-ODX-WEB-007) |
| [STRIPE](#reuse-BND-045) | SERVICE_ADAPTER | AREA-WEB-ACCOUNT | [ODX-WEB-004](#issue-ODX-WEB-004) |

<a id="reuse-BND-042"></a>
### OIDC: SERVICE_ADAPTER

**Берём:** Managed OIDC, Auth0 — прежний кандидат; jose для верификации JWT/JWKS.

**Пишем сами:** IdentityPort, issuer/audience/expiry, session boundary; отдельные Odelix entitlements и grants.

**Не переносим / граница:** IdP не выдаёт право читать любой инструмент и не вычисляет Odelix billing permissions.

**Источник:** `https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce`. **Source pin:** `NOT_SELECTED`; для ETAPE/OPENBB это выбранная ревизия исходников, для прочих — provenance; не installed package lock. **Проверка:** `INHERITED_DECLARED_SCOPE_NOT_FRESH_SOURCE_AUDIT`.

**Предлагаемые локальные места из действующих карточек** (не подтверждённый code checkout):

| Issue / модуль | Пути |
|---|---|
| [ODX-WEB-001](#issue-ODX-WEB-001) / `AREA-WEB-SHELL` | `apps/web/src/app/`; `packages/web-ui/`; `packages/product-api-client/`; `apps/web/tests/account.spec.ts` |

<a id="reuse-BND-043"></a>
### REACT: LIBRARY

**Берём:** React + TypeScript: UI composition и типы. Next.js относится к browser host при подтверждении выбранной конфигурации, не ко всем пакетам WKS.

**Пишем сами:** Наши компоненты и bindings; high-rate данные вне React setState на каждый tick.

**Не переносим / граница:** Это не готовый терминал, не общий backend и не permission boundary.

**Источник:** `https://nextjs.org/docs`. **Source pin:** `NOT_SELECTED`; для ETAPE/OPENBB это выбранная ревизия исходников, для прочих — provenance; не installed package lock. **Проверка:** `INHERITED_DECLARED_SCOPE_NOT_FRESH_SOURCE_AUDIT`.

**Предлагаемые локальные места из действующих карточек** (не подтверждённый code checkout):

| Issue / модуль | Пути |
|---|---|
| [ODX-WEB-001](#issue-ODX-WEB-001) / `AREA-WEB-SHELL` | `apps/web/src/app/`; `packages/web-ui/`; `packages/product-api-client/`; `apps/web/tests/account.spec.ts` |
| [ODX-WEB-002](#issue-ODX-WEB-002) / `AREA-WEB-ASK` | `apps/web/src/features/runs/`; `apps/web/src/features/evidence/`; `apps/web/tests/run-viewer.spec.ts` |
| [ODX-WEB-003](#issue-ODX-WEB-003) / `AREA-WEB-DESK` | `apps/web/src/features/desk/`; `apps/web/tests/select-ask-save.spec.ts` |
| [ODX-WEB-004](#issue-ODX-WEB-004) / `AREA-WEB-ACCOUNT` | `apps/web/src/features/account/`; `apps/web/tests/usage-access.spec.ts` |
| [ODX-WEB-005](#issue-ODX-WEB-005) / `WEB-STRATEGY-LAB` | `apps/web/src/features/research-builder/`; `apps/web/tests/research-draft.spec.ts` |
| [ODX-WEB-006](#issue-ODX-WEB-006) / `WEB-STRATEGY-LAB` | `apps/web/src/features/experiments/`; `apps/web/tests/experiment-lifecycle.spec.ts` |
| [ODX-WEB-007](#issue-ODX-WEB-007) / `WEB-STRATEGY-LAB` | `apps/web/src/features/experiment-compare/`; `apps/web/tests/compare-variants.spec.ts` |
| [ODX-WEB-008](#issue-ODX-WEB-008) / `WEB-STRATEGY-LAB` | `apps/web/src/features/research-passport/`; `apps/web/tests/research-export.spec.ts` |
| [ODX-WEB-009](#issue-ODX-WEB-009) / `AREA-WEB-JOURNAL` | `apps/web/src/features/paper/`; `apps/web/src/features/journal/`; `apps/web/tests/paper-journal.spec.ts` |
| [ODX-WEB-010](#issue-ODX-WEB-010) / `AREA-WEB-WATCH` | `apps/web/src/features/watches/`; `apps/web/src/features/missions/`; `apps/web/tests/mission-policy.spec.ts` |

<a id="reuse-BND-044"></a>
### TANSTACK: LIBRARY

**Берём:** TanStack Query для request lifecycle/cache server objects; точный пакет/pin выбирается в задаче.

**Пишем сами:** Query keys с tenant/object/revision, invalidation и typed API client.

**Не переносим / граница:** Не класть каждый tick и полную книгу в query cache как authoritative Market state.

**Источник:** `https://tanstack.com/query/latest`. **Source pin:** `NOT_SELECTED`; для ETAPE/OPENBB это выбранная ревизия исходников, для прочих — provenance; не installed package lock. **Проверка:** `INHERITED_DECLARED_SCOPE_NOT_FRESH_SOURCE_AUDIT`.

**Предлагаемые локальные места из действующих карточек** (не подтверждённый code checkout):

| Issue / модуль | Пути |
|---|---|
| [ODX-WEB-001](#issue-ODX-WEB-001) / `AREA-WEB-SHELL` | `apps/web/src/app/`; `packages/web-ui/`; `packages/product-api-client/`; `apps/web/tests/account.spec.ts` |
| [ODX-WEB-002](#issue-ODX-WEB-002) / `AREA-WEB-ASK` | `apps/web/src/features/runs/`; `apps/web/src/features/evidence/`; `apps/web/tests/run-viewer.spec.ts` |
| [ODX-WEB-003](#issue-ODX-WEB-003) / `AREA-WEB-DESK` | `apps/web/src/features/desk/`; `apps/web/tests/select-ask-save.spec.ts` |
| [ODX-WEB-005](#issue-ODX-WEB-005) / `WEB-STRATEGY-LAB` | `apps/web/src/features/research-builder/`; `apps/web/tests/research-draft.spec.ts` |
| [ODX-WEB-007](#issue-ODX-WEB-007) / `WEB-STRATEGY-LAB` | `apps/web/src/features/experiment-compare/`; `apps/web/tests/compare-variants.spec.ts` |

<a id="reuse-BND-045"></a>
### STRIPE: SERVICE_ADAPTER

**Берём:** Stripe Billing/Checkout/portal и проверка webhook через SDK.

**Пишем сами:** BillingProviderPort, idempotent event projection, usage ledger/outbox и entitlements.

**Не переносим / граница:** Не доверять success URL; не вводить синхронную зависимость каждой операции от Stripe.

**Источник:** `https://docs.stripe.com/billing/subscriptions/webhooks`. **Source pin:** `NOT_SELECTED`; для ETAPE/OPENBB это выбранная ревизия исходников, для прочих — provenance; не installed package lock. **Проверка:** `INHERITED_DECLARED_SCOPE_NOT_FRESH_SOURCE_AUDIT`.

**Предлагаемые локальные места из действующих карточек** (не подтверждённый code checkout):

| Issue / модуль | Пути |
|---|---|
| [ODX-WEB-004](#issue-ODX-WEB-004) / `AREA-WEB-ACCOUNT` | `apps/web/src/features/account/`; `apps/web/tests/usage-access.spec.ts` |

### Сервисы и datasets: прямые, общие и отложенные подключения

Только direct Issue означает scope конкретного подключения. Related/generic — класс работ, не выбранный provider. Future/deferred строки не надо реализовывать автоматически; они сохранены, чтобы отдельный repo не потерял архитектурный замысел.

| Источник | Покрытие | Прямые задачи | Связанный общий scope | Решение/условие |
|---|---|---|---|---|
| viem/wagmi; Privy при подтверждённом friction | FUTURE_NO_ISSUE |  |  | Сначала standard external wallet, Privy условен. Read-only Research не требует кошелька.; Execution users / V1 |

### Унаследованные варианты — не дополнительные зависимости

Связь по source group не доказывает выбор каждого пакета. Ниже сохранены reference-кандидаты, связанные с локальными sources. Более старые решения могут расходиться с текущим scope; не разрешать их молча.

| Компонент | Исторический статус | Роль/граница |
|---|---|---|
| [React](https://github.com/facebook/react) + TypeScript | `ADOPT` | Presentation/composition baseline собственной web-native Workstation; React components не владеют domain state; shell semantics принадлежат `WKS-*` |
| [TanStack Query](https://github.com/TanStack/query) | `ADOPT-LIMITED` | Server-state cache, retries и invalidation в `WKS-APP`; Не становится event/domain store; mutations идут только через application commands |

### Ранний внешний data layer (генерируемый scope)

OpenBB ODP SELECTED_FOR_IMPLEMENTATION, runtime NOT_RUN; в этом repo применяются только его обязанности. Native flow/replay и права не подменяются внешним snapshot. Полный normative contract принадлежит Market; consumer получает producer artifact.

| Profile | Mode | Доказанная граница / ограничения | Issues |
|---|---|---|---|
| ODP-DERIBIT-BARS | HISTORICAL_POLL | explicit date range; no full-history default; source precision preserved, not exact exchange reconstruction; no aggression/price-bin data | `ODX-MKT-091` |
| ODP-ECB-REFERENCE | DATED_CURRENT_SNAPSHOT | daily.xml, not historical series; not executable intraday FX price; observation date not exact publication timestamp | `ODX-MKT-091` |
| ODP-DERIBIT-CHAIN | COMPOSITE_CURRENT_SNAPSHOT | one connection per expiry in upstream, require measured fanout cap; 2s per expiry receive timeout; partial/exceptions not complete proof; BTC/ETH price conversion to USD and round(2); IV percent divided by 100; New York aware row timestamps normalized to UTC preserving instant; contract_size=1 and today-derived DTE are not verified terms; date-only expiry and current universe do not supply historical PIT | `ODX-MKT-037`, `ODX-WKS-021` |
| ODP-FRED-REVISED-PREVIEW | DISABLED_API_PROFILE_NOT_PERSONAL_FILE_ROUTE | Not installed/selected as active ODP fallback; FRED general restrictions apply beyond API: personal download not corpus storage; Use separate bounded personal file policy; no automated ODP/ML route | `ODX-MKT-090`, `ODX-MKT-042`, `ODX-MKT-043`, `ODX-MKT-044` |

Cache/replay policy: source+model+canonical instrument+query+normalization version+rights credential partition; reauthorize on hit; no silent provider fallback. stored OpenBB response plus versioned deterministic normalization, not upstream event replay; no vendor call during saved replay. Public activation: requires actual Product grants, source/AGPL/data rights and operational acceptance; local preview is not public authorization.

**Приёмка внешнего компонента:** existing-code check → точный package/commit и license/NOTICE/dependencies → adapter test с недоступностью/ошибками/качеством → фактический pin и результат. Не создавать второй runtime/численный engine под видом ускорения. Текст этого блока не устанавливает packages, не покупает сервисы и не выдаёт production rights.
<!-- END GENERATED IMPLEMENTATION BASIS -->

<a id="architecture"></a>
## Архитектура


`odelix-web` is a separate release and product surface. For the early local/pilot path it owns
account, checkout, OAuth consent/callback, usage and portable-result/share views;
it is not another copy of the Workstation engine.

```text
apps/web
  routes/             authenticated terminal, journal, public/share, settings
  compositions/       workstation, options, replay, learning, mobile viewer
  adapters/            Product API, Market stream, billing/auth/analytics
packages/web-ui/       Web-only design system and responsive shells
```

Versioned dependencies:

- `@odelix/workstation-*` for command/object/pane/layout/scene runtime;
- generated `@odelix/market-client` for Scene/Replay stream;
- generated `@odelix/product-client` for product objects/commands;
- Product-issued short-lived grant for direct Market connection.

No deep import into sibling repository source. Public/mobile composition may use
a smaller view set but preserves IDs, time semantics, quality and evidence refs.

Connect account/results consume published Odelix Pi Harness run/event
projections via Product transport. Web does not embed Pi, duplicate Skills or
implement another agent loop. Terminal composition follows technical data/contract readiness, independently of first payment. Public pilot needs access/grants; commercial access needs CONNECT_COMMERCIAL.


### Options и research integration (PROPOSED)

Desk / Strategy Lab activates by research prerequisites alongside Connect. It supports the strategy catalogue and custom experiments; full Workstation follows user demand, and payment is not a technical prerequisite for PAPER. All panes/views consume authoritative producer outputs.

Detailed scope: [OPTIONS-WORKFLOW.md](#DEP-2fd02fb327).


### Research/strategy capability requirements

Стандартные и custom strategies, experiment review и capability-aware UI. Основные owners: Strategy Lab composition.

Новая поверхность использует эти же contracts и semantic fixtures; численные/торговые engines не копируются в UI или prompts. MODULE описывает actual/planned paths раздельно.


### Workstation-first r15: текущая приёмка

Использовать WKS shared packages и user AlphaQuant interactions. Не считать Workspaces HTML единственным источником Options depth, не переносить sample detectors/probabilities как готовый backend.

Каноническая детализация: [PROTOTYPE-MIGRATION.md](PRODUCT-SURFACE.md#prototype-migration); [current delivery](#DEP-b335630551).


<a id="contracts"></a>
## Контракты

**Статус этой сборки:** перечисленные семейства — спецификация. Заголовок «Published» в унаследованном тексте означает целевую поверхность публикации, не доказательство существующего release. Реальный pin/digest и conformance требуются до integration acceptance.


Consumes Workstation SDK, Market Scene/Replay and Product API. Web-published
contracts are limited to public route/package representation and any explicit
embed/share API. Do not re-export sibling internal types as Web-owned semantics.

Compatibility is checked against the exact package/client ranges in stack
manifest. Public/share payloads require rights, expiry/redaction policy,
point-in-time manifest and unsupported-data behavior.

For `DRAFT` inputs before their consumer gate, ranges are replaced by exact
git/schema/package digests. Account/Connect contracts freeze before CONNECT_COMMERCIAL; Scene freezes at SCENE_CONTRACT_READY before external pilot.
Web updates only together with producer fixtures; no local DTO fork.

Agent views consume `AgentSessionRef/RunRequest/HarnessEvent/TerminalOutcome`
owned by `odelix-harness`; no Pi private type is re-exported as a Web
contract.


### Options и research integration (PROPOSED)

Optional WebMCP adapter exposes current-context and the same read/compute/draft contracts only after browser/client support tests. Do not expose keys or withdrawals. Strategy budget, residual portfolio loss and quote finality are separate UI fields.

Detailed scope: [OPTIONS-WORKFLOW.md](#DEP-2fd02fb327).


### Research/strategy capability requirements


Общие StrategySpec/ExperimentSpec — Product; calculation/data manifests — Market; agent handoff — Harness; order/policy — Execution. Schema producer owns source, Stack registry discovers it. DRAFT entries не считаются published; copy/paste бизнес-типов между repos запрещён.


<a id="modules"></a>
## Модули


These are Web feature slices, not new domain owners.

| Area | Owns in this repo | Delegates |
|---|---|---|
| App shell | routes/navigation/session/error boundaries | WKS commands/objects/panes |
| Terminal | responsive assembly of Market Scene and inspectors | WKS-SCN + Market |
| Ask flow | selection/ask/evidence/thesis presentation | Product application contracts |
| Options | chain/surface/underlying overlays composition | Market MKT-OPT |
| Replay | controls/timeline/share presentation | Market replay + Product episode |
| Journal | product object views/search/export UI | Product Thesis/Memory |
| Learning | simplified explanation mode/tooltips | same Evidence, different lens |
| Public/share | point-in-time permalink/package viewer | rights policy + portable package |
| Mobile companion | view/follow-up/material attention | narrow WKS/Product projections |
| Account | onboarding/BYOK/billing/settings UI | Product Platform contracts |
| Connect | OAuth consent/status, capability catalogue, usage, result history | Product + Harness projections |

Execution, DOM trading and autonomous agents remain dormant unless Founder gates
activate them, even if the master surface spec describes the long-term target.
CONNECT_COMMERCIAL adds account/Connect/usage/payment surfaces; it is not the first activation of the technical Desk.


### Актуальные подробные спецификации

| ID | Gate | MODULE |
|---|---|---|
| WEB-STRATEGY-LAB | RES-1 | [WEB-STRATEGY-LAB](../modules/strategy-lab/MODULE.md) |


<a id="testing"></a>
## Проверка


- shared Workstation SDK acceptance in browser;
- responsive desktop/tablet/mobile viewer tests;
- real and replay Market stream resync/degradation;
- CONNECT_COMMERCIAL OAuth→MCP/account→usage/result visibility E2E;
- later Select → Ask → Evidence → Thesis vertical E2E;
- public/share authorization, data-right and expiry tests;
- onboarding/BYOK/billing capability parity;
- accessibility/keyboard completeness and focus behavior;
- bundle/cold-start/performance budgets and long-session memory.


### Options и research integration (PROPOSED)

Test payoff labels/units, portfolio residual-loss disclosure, stale/indicative/firm state, accessibility, API/MCP/WebMCP parity where supported, and inability of browser state to authorize orders.

Detailed scope: [OPTIONS-WORKFLOW.md](#DEP-2fd02fb327).


### Research/strategy capability requirements


Meaningful acceptance scenarios: real vs synthetic vs paper labels, stale form revision, actual eligibility, exact strategy version and read-only quote semantics. Здесь перечислены planned tests; executed results должны ссылаться на commit/CI.


<a id="runbook"></a>
## Выпуск и эксплуатация


Pin Workstation SDK and generated clients; run compatibility and E2E against the
release `stack.lock`; publish immutable assets; verify CSP/secret absence/source
maps, public-route rights, error tracking and rollback. A Web release must not
restart Market recorder or require Product database changes unless declared.


### Options и research integration (PROPOSED)

Feature flags for options/WebMCP/trading are separate. No deployment toggles on live handlers because research UI shipped. On unknown outcome show pending/reconciling rather than optimistic balance.

Detailed scope: [OPTIONS-WORKFLOW.md](#DEP-2fd02fb327).


### Research/strategy capability requirements


Начинать с inventory существующего кода/Issues. Глобальный порядок и A0 находятся в `DEP-9359295523` (см. локальный реестр зависимостей); Project metadata включает нормализованный Module согласно r15.3; дополнительные поля не добавляются автоматически. Не добавлять ручной второй статус-журнал.



## Выбранная UI-механика eTape

Web подключает готовые Odelix Workstation packages. Source intake и порт ET-01..07 выполняются только владельцем Workstation; Web не создаёт второй eTape fork, не копирует его WsClient/engine и не запускает broker UI. AlphaQuant workflows и собственные Product/Harness interfaces сохраняются.


<a id="early-external-data"></a>
<a id="ранняя-data-ветка-r158-реализация-и-владельцы"></a>
## Ранняя data-ветка: реализация и владельцы

Ранний external preview использует те же WKS-020/021 packages и MarketDataPort. В Web остаются browser routes, account и composition; provider registry и OpenBB client не дублируются. [Пользовательский surface](PRODUCT-SURFACE.md#external-data-preview) сохраняет источник по каждому слою.

Нельзя показывать current polled snapshot как streaming orderflow или внешний BTC perpetual как тот же Binance spot. UI использует enum availability + completeness + delivery mode, а не один зелёный connected badge. Техническая разработка не зависит от оплаты; публичное использование зависит от актуальных прав/допуска.
<!-- R159 hosted-ui -->
## Ранний Watch и аутентифицированные подтверждения

WEB-010 использует WEB-001 и общие WKS-компоненты с PRD-022/024; не ждёт WEB-003 или Paper Journal. Mobile web вызывает Product ApprovalPort после аутентификации; уведомление не authorization. ONLINE/24-7 — пользовательские комбинации осей Product, не второй runtime. Browser не запускает CLI или supplier API напрямую.

<!-- BEGIN GENERATED REPO CONTEXT -->

<a id="modules-index"></a>
## Каталог модулей этой области

Один primary owner в каждой задаче; affected modules отдельно. AREA — служебная классификация. Идентификаторы сохранены. Activation — scope текущего плана, не доказательство исполнения; независимый implementation_status в JSON.

| ID | Вид | Activation | Источник в этом repo |
|---|---|---|---|
| `AREA-WEB-ACCOUNT` | DELIVERY_AREA | ACTIVE | [docs/DEVELOPMENT.md](#modules) |
| `AREA-WEB-ASK` | DELIVERY_AREA | ACTIVE | [docs/DEVELOPMENT.md](#modules) |
| `AREA-WEB-DESK` | DELIVERY_AREA | ACTIVE | [docs/DEVELOPMENT.md](#modules) |
| `AREA-WEB-JOURNAL` | DELIVERY_AREA | ACTIVE | [docs/DEVELOPMENT.md](#modules) |
| `AREA-WEB-SHELL` | DELIVERY_AREA | ACTIVE | [docs/DEVELOPMENT.md](#modules) |
| `AREA-WEB-WATCH` | DELIVERY_AREA | ACTIVE | [docs/PRODUCT-SURFACE.md](PRODUCT-SURFACE.md) |
| `WEB-STRATEGY-LAB` | MODULE | ACTIVE | [modules/strategy-lab/MODULE.md](../modules/strategy-lab/MODULE.md) |

<a id="work-queue"></a>
## Локальная очередь и следующий шаг

Полный текст каждой задачи — в [delivery/CONTEXT.json](../delivery/CONTEXT.json), это автоматически полученная выборка, не второй редактируемый backlog. Найдите объект по `id`, прочитайте `work`, `acceptance`, `paths`, `depends_on`, `primary_module_id`.

Сначала действующий Issue, actual commit/permissions, затем следующее допустимое действие. Planned wave не статус и не мандат. Внешняя зависимость должна предоставить артефакт/fixture; соседний checkout не предполагается.

| ID | Модуль | Волна | Результат | Зависимости |
|---|---|---|---|---|
| <a id="issue-ODX-WEB-001"></a>`ODX-WEB-001` | `AREA-WEB-SHELL` | 3 | Создать Web shell, login и thin Product API client | ODX-PRD-004 |
| <a id="issue-ODX-WEB-002"></a>`ODX-WEB-002` | `AREA-WEB-ASK` | 3 | Сделать Run progress/result/Evidence viewer | ODX-WEB-001, ODX-PRD-004 |
| <a id="issue-ODX-WEB-003"></a>`ODX-WEB-003` | `AREA-WEB-DESK` | 3 | Собрать первый Desk из общих WKS packages | ODX-WEB-002, ODX-WKS-002, ODX-WKS-003, ODX-PRD-011, ODX-HAR-007 |
| <a id="issue-ODX-WEB-004"></a>`ODX-WEB-004` | `AREA-WEB-ACCOUNT` | 6 | Добавить Account usage/checkout и revoke UX | ODX-WEB-003, ODX-PRD-012, ODX-PRD-013 |
| <a id="issue-ODX-WEB-005"></a>`ODX-WEB-005` | `WEB-STRATEGY-LAB` | 4 | Собрать Research Builder: inputs,formula,target,entry и exit | ODX-WKS-004, ODX-PRD-017 |
| <a id="issue-ODX-WEB-006"></a>`ODX-WEB-006` | `WEB-STRATEGY-LAB` | 4 | Добавить Experiment launch/progress/results UX | ODX-WEB-005, ODX-PRD-018 |
| <a id="issue-ODX-WEB-007"></a>`ODX-WEB-007` | `WEB-STRATEGY-LAB` | 5 | Сделать Compare для формул,input и независимых exits | ODX-WEB-006, ODX-PRD-019 |
| <a id="issue-ODX-WEB-008"></a>`ODX-WEB-008` | `WEB-STRATEGY-LAB` | 5 | Добавить Research Passport/export и возврат к source context | ODX-WEB-007, ODX-PRD-018, ODX-HAR-011 |
| <a id="issue-ODX-WEB-009"></a>`ODX-WEB-009` | `AREA-WEB-JOURNAL` | 5 | Реализовать Paper Positions/Orders/Journal views | ODX-WEB-008, ODX-PRD-021, ODX-EXE-004 |
| <a id="issue-ODX-WEB-010"></a>`ODX-WEB-010` | `AREA-WEB-WATCH` | 3 | Добавить Watch/Mission editor и управление уведомлениями | ODX-WEB-001, ODX-WKS-011, ODX-PRD-022, ODX-PRD-024 |

<a id="dependencies"></a>
## Внешние зависимости и источники

**Реестр ниже не является очередью обязательного чтения.** Большинство записей — происхождение решений; артефакт запрашивается только для конкретной необходимой зависимости задачи. **Независимость checkout не означает отсутствие зависимостей продукта.** Здесь записаны логические координаты издателя, а не относительные переходы в соседнюю папку. `source_sha256` в локальном JSON удостоверяет только исходный документ r15.3. Ни одна строка не является доказательством выпуска schema/SDK.

Для конкретной задачи получить через TaskPacket или разрешённый артефактный канал: producer + contract ID, точный release/commit, schema/package digest, fixtures, consumer-conformance и разрешённые операции. Записать фактический локальный путь после получения; пока artifact не предоставлен, зависимая runtime-работа не готова. Локальные fixture/design-задачи возможны по своему мандату.

Не заменять чужую схему её ручной копией. Обновление версии — producer release → consumer pin → conformance → интеграция. Разрешение читать артефакт не даёт права изменять другой repo.

| Ref | Publisher | Логический источник | Статус исходника |
|---|---|---|---|
| <a id="DEP-143d61eff7"></a>`DEP-143d61eff7` | `odelix-stack` | `reference/OSS-REVIEW-RU.md` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-14e5071e3a"></a>`DEP-14e5071e3a` | `odelix-stack` | `reference/PROTOTYPE-SOURCES.json` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-2fd02fb327"></a>`DEP-2fd02fb327` | `odelix-product` | `docs/OPTIONS-WORKFLOW.md` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-52ea262d35"></a>`DEP-52ea262d35` | `odelix-stack` | `docs/INTEGRATIONS.md#code-reuse` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-568bec8dbe"></a>`DEP-568bec8dbe` | `odelix-workstation` | `docs/WORKSTATION-SPEC.md` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-90bbeb40e2"></a>`DEP-90bbeb40e2` | `odelix-stack` | `archive/r15.1-source.zip` | EXTERNAL_HISTORY_OPTIONAL |
| <a id="DEP-9359295523"></a>`DEP-9359295523` | `odelix-stack` | `docs/DELIVERY-RUNBOOK.md` | DOCUMENT_SNAPSHOT_ONLY |
| <a id="DEP-b335630551"></a>`DEP-b335630551` | `odelix-stack` | `AGENTS.md` | DOCUMENT_SNAPSHOT_ONLY |

`UNRESOLVED_SOURCE_REFERENCE` — отсутствующий документ/якорь исходного пакета явно зарегистрирован; содержание не придумано. Для historical source его можно оставить архивной ссылкой, для обязательной зависимости — запросить источник. Public URLs в предметных документах сохранены как датированные ссылки и не проверялись онлайн этой сборкой.
