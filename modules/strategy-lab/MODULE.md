# WEB-STRATEGY-LAB — strategy-lab

**Owner repo:** `odelix-web` · **Activation gate:** RES-1 · **Design:** 1.0.0 / 2026-09-28.
**Implementation status:** SPEC_ONLY / requires code inventory. Proposed path: `modules/strategy-lab`; this path does not assert an existing crate/package. Reuse existing implementation before introducing files.

## Назначение и предел ответственности

Browser composition для catalog/custom builder, experiment review, Strategy Passport и paper controls; не численный engine.

## Public ports и контракты

Consumes Product strategy/experiment and Market projections; commands carry exact revision, tenant and user intent.

Имена операций задают semantics, не подтверждают опубликованный API. Producer schema/fixtures и точная версия выпускаются вместе с реализацией; Stack registry содержит discovery/pins. Ошибки типизированы, environment/tenant/asOf/unit fields обязательны там, где применимы.

## Разрешённые зависимости

Generated SDK, shared scene/object refs, auth session, chart/table primitives; no raw provider credentials.

Private imports и запись в чужие таблицы запрещены. Domain logic получает immutable inputs; infrastructure I/O подключается composition root. При новом общем контракте сначала согласуются producer fixtures и consumer migration.

## Состояние, время и восстановление

Local draft UI state distinct from saved server revision. Unsaved edits never change running version; pending jobs show durable server status.

Version/config/code/dataset/contract refs связывают результат с исходными условиями. Retry действует в пределах operation id; unknown external state сверяется до повторного side effect. Денежные величины и timestamps не теряют точность при сериализации.

## Инварианты и отказ

Synthetic/historical/paper/live visually distinct; missing field не рисуется как ноль; live action disabled without actual capability, not just UI checkbox.

## Acceptance и review

Resume experiment after refresh; stale revision conflict; wrong tenant link; no data backtest blocked; unsupported family explanation; cancel-run vs pause-deployment distinction.

До Done нужны реальный code map/commit, выполненные meaningful checks, peer review и ссылки в Issue. Здесь тесты описаны как требования, не отмечены выполненными. Проверить публичные exports, allowed dependencies и отсутствие дублирующей реализации. Нет требования создавать новый runtime или deployment для этого модуля.

## Наблюдаемость и знание проекта

Логировать operation/run/strategy refs, duration, typed outcome, coverage и retry count без secrets/скрытых рассуждений. Метрики применяются к своей ответственности: module-specific failures из раздела инвариантов должны отличаться от infrastructure timeouts. Builder обновляет факты этого MODULE с кодом; Coordinator обновляет index/status. Не копировать полный отчёт в несколько документов.
