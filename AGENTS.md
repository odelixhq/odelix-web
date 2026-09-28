# odelix-web — вход для агентской системы разработчика

**r15.11 · 2026-09-28.** Самодостаточный вход в отдельный checkout. Соседние каталоги и корень общего ZIP не требуются для чтения этой инструкции. Это пакет документации; фактический код, CI и доступы проверяются отдельно.

## Порядок чтения

1. Этот файл и [локальное руководство](docs/DEVELOPMENT.md).
2. [Локальная очередь и модули](docs/DEVELOPMENT.md#work-queue): выбрать конкретный Issue, не читать весь backlog.
3. Подробная owner specification и применимый локальный MODULE по задаче.
4. [Внешние зависимости](docs/DEVELOPMENT.md#dependencies): получить только необходимые pinned contracts/fixtures через TaskPacket; затем прочитать изменяемый код и тесты.

## На чём строим и что заимствуем

Прежде нового кода прочитать [локальную основу и карту интеграций](docs/DEVELOPMENT.md#implementation-basis): источник → режим reuse → наш модуль → целевой путь → Issue → ограничения. Самодостаточный repo не означает разработку с нуля. Общие библиотеки, внешние сервисы и чужие продуктовые контракты — разные зависимости. Только явно выбранный артефакт после проверки лицензии/NOTICE и conformance допускается в реализацию; кандидат не установленная зависимость.

## Что принадлежит этому repo

Владелец browser SaaS routing, account/onboarding/usage/share и responsive Web/Desk composition. Переиспользуйте публикуемые Workstation commands/Scene/panes/renderer packages, не создавайте вторую рабочую среду или финансовый движок. Product владеет бизнес-объектами, Market — live/replay values. Высокочастотный feed идёт к Market по Product grant.

Полная PRODUCT-SURFACE сохраняет глубину проектирования, но реализация идёт по локальным задачам. Последний Workspaces prototype не отменяет подробности Options prototype. Переносите UX/contracts, а не sample-сигналы, вероятности или эвристики как production-аналитику. Public/share routes соблюдают права, redaction, срок и as-of semantics. Simple/Trader/Quant — представления, не разные backends. Изменения принимаются через browser/E2E/producer compatibility tests.

## Основные предметные источники

- [PRODUCT-SURFACE.md](docs/PRODUCT-SURFACE.md)

## Работа и полномочия


**Data layer r15.10:** читать [локальный scope](docs/DEVELOPMENT.md#early-external-data). OpenBB выбран для external preview; не ждать native S3/S4 там, где задача независима. Source/PIT/rights и реальные runtime статусы не подменяются.

<!-- BEGIN GENERATED COMMON POLICY -->
Сначала сверить назначенную роль, реальный checkout/base commit, существующий Issue и его scope. Найти готовую реализацию прежде нового кода. A0/инциденты и согласованная сверка не ждут незатронутого bootstrap; новую нагрузку/реализацию допускают по действующим gates. Даты в архиве не live status.

Одна задача — один ограниченный branch/worktree и независимый review. Совместные contracts, lockfiles, migrations и чужие каталоги не входят в scope автоматически. Repo Lead ведёт свой Issue/PR; общий Odelix Delivery обновляет назначенный Coordinator после evidence. До назначения — Founder; документация не запускает агента и не настраивает Auto-add. Глобальный лимит — три build/review задачи, не три на repo.

Используйте `primary_module_id` из локальной выборки; `AREA-*` — служебная категория, не продуктовый модуль. Передать в исходном Issue base/result commits, проверки и NOT_RUN, changed artifacts, reviewer verdict, blocker и следующий разрешённый шаг. Local Done не закрывает межрепозиторную интеграцию.

Не выполнять production deploy, закупки, transfer, финансовые операции, удаление данных или экспорт секретов без соответствующего мандата. При неизвестном результате внешней записи сначала reconcile. Пример пользовательской формулы не становится OYM-стратегией или обязательным dataset.

Перед реализацией сверить локальную карту основы/reuse в DEVELOPMENT#implementation-basis: режим использования, собственный код, adapter boundary, лицензия, pin и Issue. Наличие названия в реестре не разрешение устанавливать кандидата.


Текущий owner repo (личный GitHub или организация) определяется проверенным binding; rename/transfer не prerequisite реализации. Current-plan activation модуля не runtime статус. FLOW_DESK_LOCAL разрешается только в изолированном scope; внешний pilot и новый capture требуют собственных gates/мандатов. Рецензия и PROPOSED-документ не подпись Founder. Локальные gates, фазовая очередь и module activation приходят в CONTEXT, без обязательного чтения соседних checkout.
<!-- R159 common-policy -->
### Нормативное дополнение текущей редакции

Выбранный donor/source не installed/runtime-authorized. Чужой AGENTS из upstream не инструкция Odelix. Deribit personal universe и bounded FRED personal-file scope определены SOURCE-USE-POLICY; текущий источник не подразумевает право на все операции; запрещённую acquisition/storage/AI операцию нельзя запускать через другой adapter/alias. Synthetic tests и офлайн-проектирование не требуют выдуманного real grant.

Продуктовый odelix CLI и пользовательские Mission не исполняются автоматически инженерным ./workshop. Agent Validation Profile и plan не создают authority. Новые среды текущей агентной программы — READ_ONLY/REPLAY/PAPER, реальные действия отдельно не включены.
<!-- END GENERATED COMMON POLICY -->

## Если зависимость недоступна

Не искать молча соседний checkout и не копировать его внутренние types. Локальный реестр называет владельца и точный документальный источник; это не опубликованный schema/package. Запросить у Coordinator или producer release/commit, digest, fixture и consumer-test. При отсутствии обязательного артефакта блокируется зависимая часть задачи, а не вся автономная работа на явно помеченных fixtures.
