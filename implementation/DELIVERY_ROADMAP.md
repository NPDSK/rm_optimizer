# RM Optimizer — Delivery Roadmap

Статус: DESIGN DRAFT, 12 сентября 2026. Рекомендуемый порядок крупных вертикальных milestones, **не разрешение на реализацию, commit, deploy или install сейчас**.
В этом milestone созданы только документы; SQL/продуктовый код и зависимости не менялись.

Основания: [CJM](../product/PERSONAS_AND_CJM.md), [requirements](../product/PRODUCT_REQUIREMENTS.md), [acceptance](../product/ACCEPTANCE_CRITERIA.md), [domain](../architecture/DOMAIN_MODEL_TARGET.md), [architecture](../architecture/TARGET_ARCHITECTURE.md), [gap и migration](../architecture/CURRENT_TO_TARGET_GAP.md).

## Почему такой порядок

Менеджер сначала должен получить законное управление определённым объектом, затем подготовить людей и работу, затем увидеть реальный feasible scenario и только после этого публиковать обязательства. Поэтому Context identity и authorization входят в один foundation milestone: нельзя выпустить новый организационный chooser с прежними незащищёнными management APIs. Mapping, ingestion, hierarchy и readiness объединены в одну вертикаль, чтобы не выпускать ещё один список, который не соответствует optimizer inputs.

M1–M6 образуют рекомендуемый целостный MVP D-01. M4 полезен как scenario pilot, но ещё не законченный продукт. M7 — отдельная post-MVP автоматизация. Multi-Team solve и fine-grained multi-context capacity reservations требуют следующего отдельного решения, не скрыты в обычном Team selector.

## Общий release contract каждого milestone

- Вход: approved scope milestone и затрагиваемые D-решения, read AGENTS/override, зафиксированные branch/status и intentional pre-existing changes, проверенные инфраструктурные prerequisites.
- Schema changes только additive/versioned с backfill manifest, collision report, resumable retries и verification. Не изменять уже применённые v001–v039; no destructive reset/rebuild. Ambiguous legacy binding остаётся видимым blocker.
- Один authoritative read/write owner после cutover. Transitional adapters имеют exit criteria; нельзя бесконечно поддерживать конкурирующие project и Context pointers.
- Node/domain/repository tests, frontend tests/build при UI, Python tests при solver/boundary изменениях, backend syntax, forge lint, diff/status. Mock SQL query tests дополняются реальным supported SQL concurrency/recovery proof для соответствующего milestone.
- **Commit/deploy checkpoint:** сначала review конкретного diff, migration plan, validation и rollback/forward-recovery evidence; затем отдельное явное разрешение пользователя на commit/deploy. После разрешённого dev deploy — реальный Jira smoke и решение о pilot/promotion. Локальные tests не разрешают deployment; этот roadmap не меняет scopes и не предписывает install автоматически.
- Если prerequisite не доказан (например bootstrap authority, job transport или conditional SQL write), завершённый design/локальный код не называется production-ready. Риск и следующий проверяемый шаг входят в отчёт.

## M1 — Правильное ownership и работающий Planning Context

**Пользовательский результат:** проверенный administrator назначает Manager; Manager создаёт/выбирает Context, явно связывает Team и безопасно возвращается к своему legacy состоянию. Другой пользователь может запросить доступ, но не администрировать обнаруженную Team.

**Entry criteria:** утверждены D-04 bootstrap, D-06 viewer access, D-13 directory visibility, D-11 legacy policy и минимальные privacy правила D-09. Технически доказаны authority verification и нужные SQL unique/revision primitives. Есть read-only inventory legacy IDs/claims и план неоднозначностей.

**Работа по слоям:**

- Backend: Installation/UserIdentity/RMGrant/Context/ContextTeam use cases; capability matrix и foreign-reference checks для всех exposed reads/writes; Context resolution envelope, explicit selection required вместо silent project default. Legacy endpoints либо идут через тот же ACL+Context adapter, либо недоступны после cutover.
- Frontend: Context picker с Team и sources отдельно, access/ownership pending screens, Admin manager lifecycle, consistent selection/cache generation, controlled legacy binding review. Preserve Custom UI; нет переименования старой Team как способа исправить выбор.
- Persistence/migration: добавить Context и ACL/binding records, сохранить Team/resource/history IDs. Однозначный legacy state получает собственный Context; ambiguous Cools/Legal остаётся read-only/review-required. Project-origin не permission. Bootstrap/grant finalization идемпотентны.
- Solver: математическое ядро без изменений; application contract получает versioned Context ownership. Старый project-bound optimization не становится доступным через bypass.
- Reuse/refactor/remove: сохранить external ID normalization, unique lookup/bind, resolver whitelists и React race guards; переделать `getPlanningSetup` default ownership; убрать автоматическое authority из assumptions/compatibility flows, если обнаружится при реализации.

**Acceptance:** AUTH-001…AUTH-004, CTX-001/CTX-002, identity часть TEAM-001; AC-001…AC-011 и AC-048/AC-049 для новых operations. Rename/entry-project switch не меняет identity; unauthorized management отклоняется backend; revoke effective для следующей операции.

**Tests:** `authorization.test.js`, `planning-context.test.js`, `context-persistence.test.js`, existing resolver/Team/SQL suites и новые Context/Admin React suites из таблицы ниже. Fixture migration повторяется и прерывается на каждом write; проверяются collisions и отсутствие history loss.

**Реальный Jira smoke:** на dev site с отдельными approved test accounts authority/Manager/ordinary user/JiraProjectAdmin; обнаружить одну Team из двух проектов, открыть CA/CB, проверить отказ direct APIs и revoked grant. Legacy реальные имена рассматриваются как labels; не менять membership без отдельного approved test plan.

**Commit/deploy checkpoint:** review mapping manifest и access matrix, commit/deploy только после отдельного разрешения; smoke доказывает entry/ownership/isolation. Acceptance M1 не требует работающего solver или self-enrollment approval UI.

## M2 — Полный onboarding людей, profiles и approved capacity

**Пользовательский результат:** Manager собирает состав, Member запрашивает join, видит review и свой профиль, предлагает skills/availability; Manager получает готовые approved daily capacities своей Team/Context.

**Entry criteria:** M1 ownership/ACL/Context cutover принят; утверждены D-07 membership→grant policy, D-08 catalogs, D-09 absence visibility и D-03 начальный guard shared people/commitments. TEAM identity API limitation признана штатным unavailable state.

**Работа по слоям:**

- Backend: manager add/exclude, join request approve/reject/withdraw, own profile proposal/review, availability requests/cancellation, Context allocation validation. Explicit completed join создаёт RM_MEMBER; no auto Manager или planning profession.
- Frontend: Manager inbox и searchable user selection; Member pending/rejected/approved journey, own profile/availability; capacity preview с explanation; membership source и organizational Team identity раздельны.
- Persistence/migration: UserIdentity links к существующим Team resources, отдельные join/approval operation records и versioned profiles/calendar/allocation. Catalog migration не merges by name. Сохранить membership overrides и Resource/history; завершение membership+grant защищено completion marker.
- Solver: переиспользовать calendar/eligibility compiler и PERSON_DAYS; добавить boundary fixtures approved/pending и shared-person double counting guard. Не вводить productivity.
- Reuse/refactor/remove: reuse Team Foundation override formula, manual mode, safe GraphQL diagnostic и failed-refresh tests. Добавить generation fencing stale sync; прежние TeamSetup save paths подчинить ACL и revision contracts.

**Acceptance:** TEAM-002…TEAM-004, PROF-001/PROF-002, CAP-001/CAP-002; AC-012…AC-021. Same-Team failed refresh сохраняет last-good; A/B isolation и MANUAL остаются; pending никогда не становится approved input.

**Tests:** membership onboarding/approval fault injection, own/foreign profile APIs, role vs ACL, explicit exclusions, calendar overlays/overlaps, approved availability events и source-generation races; TeamSetup/Member/Profile/Availability UI states.

**Реальный Jira smoke:** Manager adds permitted test user, другой user requests join, rejection затем новый approved request; user search availability и permission failures; текущий GraphQL failure сохраняет identity и manual membership. Проверить Alice в CA, отсутствие в CB, возврат CA и profile independence.

**Commit/deploy checkpoint:** review audit/retention и membership migration, отдельно разрешённый commit/deploy; на dev подтвердить полный Manager↔Member handoff без scopes/auth workaround.

## M3 — Доверенные Jira inputs: scope, mapping, canonical graph и readiness

**Пользовательский результат:** Manager видит правильные работы Context из согласованных проектов/board, понимает hierarchy/dependencies, исправляет missing inputs и получает воспроизводимый READY input set.

**Entry criteria:** M1/M2 приняты; утверждены D-02 source bounds, D-05 group value ownership, D-10 gates и timezone часть D-12. Есть проверенные Jira pagination/permissions/stable-field adapters, набор cross-project fixtures и canonical contract version plan.

**Работа по слоям:**

- Backend: Context WorkSource/WorkScopeVersion, per-source mapping set; общий scoped display+ingestion contract, generation completeness, SourceWorkIdentity и ContextWorkItem projections. Structure отдельно от F2S, explicit EXECUTABLE/CONTAINER, deliverable group validation, external gate fact/assumption review, readiness/freeze.
- Frontend: scope preview и источники, mapping wizard/sample validation, work tree/table с kind/provenance, readiness correction queue, unresolved gates и explicit assumptions. Inspection работает до READY; UI-фильтр не scope.
- Persistence/migration: Context source/mapping/overlay/policy records, bindings существующих work/history IDs. Старый upsert с overwrite `team_id` больше не authoritative. Migrate current Team scope в explicit Context definition, не добавлять все доступные проекты молча. Legacy external deadline resolutions переводить в review-required, не превращать их задним числом в manager approvals.
- Solver: сохранить solver graph/value/capacity semantics; compiler получает Context snapshot и явные gate bounds. Изменить domain tests, допускавшие automatic PLATFORM_DEADLINE fallback; hierarchy/value fixtures остаются semantic regression.
- Reuse/refactor/remove: reuse Jira Team field discovery/pagination/transforms/normalizers/history dedup, TeamWorkScope predicates; rework project-centric sync/mapping activation/canonical identity; удалить silent 1 effort/value из доверенного input path.

**Acceptance:** SCOPE-001/SCOPE-002, ING-001, MAP-001/MAP-002, WORK-001/WORK-002, DEP-001/DEP-002, READY-001; AC-022…AC-031, cross-domain AC-007/AC-009/AC-024/AC-025. Partial data никогда не READY complete; unassigned matching work входит.

**Tests:** multi-project scope/board bounds/permission loss/page retry; same source issue in two Context with independent overrides; mapping schema drift; source issue key/project move; hierarchy/graph/value overlap; gate due date remains unresolved; consistent list/compiler candidate set.

**Реальный Jira smoke:** два permitted проекта, одна Team, unassigned matching issue и assigned wrong-Team issue; board с согласованными границами; все страницы и типы hierarchy; invalid mapping/fix; внешний prerequisite без DONE, явное assumption. Пользователь без доступа к одному источнику не видит cached details/totals.

**Commit/deploy checkpoint:** review canonical migration counts/IDs/input diffs и scope preview, отдельное разрешение commit/deploy; Jira smoke должен доказать, что work list и approved input set означают один Context scope.

## M4 — Реальный asynchronous optimizer и review сценариев

**Пользовательский результат:** Manager задаёт horizon/hard policies, запускает Python CP-SAT, может закрыть UI и вернуться, получает feasible portfolio либо actionable diagnosis, сравнивает сценарии. Это scenario pilot, публикация ещё отдельный этап.

**Entry criteria:** M3 READY manifests приняты; hosting/auth/queue/callback-or-polling/limits и data minimization проверены как в [architecture](../architecture/TARGET_ARCHITECTURE.md). Утверждены benchmark/budget policy D-12 и D-03 scenario vs commitment. Никакого обещания native Forge Python без доказательства.

**Работа по слоям:**

- Backend: submit/get/cancel/finalize run use cases, immutable approved input hash, durable dispatch/attempts/fencing, validated result materialization и stale marker. Hard horizon, constraints, eligibility/WIP, in-progress/handover, result proof/termination handling; review lineage.
- Frontend: horizon/calendar/WIP/constraint editors, run progress/resume/retry/cancel, diagnostic correction links, scenario comparison по value/churn/dates/deferred/carryover/assumptions. Browser knapsack и его positive-int defaults уходят из production path.
- Persistence/migration: Context runs/attempts/input manifests/scenarios/draft items+allocations, materialization complete marker, comparison baseline refs. Legacy runs сохраняются как historical artifacts, не объявляются новыми verified Context results.
- Solver: reuse CP-SAT formulation и contracts; production service wrapper, explicit budget/termination metadata и boundary validation. Изменять ядро только по выявленному contract/performance gap; no utilization objective, no weekly simplification/productivity.
- Reuse/refactor/remove: reuse SolverInputBuilder, sparse allocations/lexicographic objectives/verification fixtures и draft materialization idea. Разделить synchronous `optimizeRoadmap`; fake service только в tests; удалить demo optimize controls/results и ненужные production demo tests после replacement coverage.

**Acceptance:** PLAN-001…PLAN-003, OPT-001…OPT-003, SCEN-001, runtime часть OPS-001/OPS-002; AC-019/AC-031…AC-041, AC-048/AC-049. Time limit не INFEASIBLE, partial output не Draft READY; objective order неизменен.

**Tests:** service replay/cancel/stale attempt/invalid output/context hash/credential; Node compiler-contract tests и async persistence fault boundaries; frontend run/review states; Python constraints/objectives/termination tests, 8 hand-check fixtures, representative benchmarks без выдуманного SLA.

**Реальный Jira smoke:** frozen permitted Context data → service run → reconnect UI → preview result; optional deadline defer и mandatory infeasibility; cancel и повтор; проверить отсутствие customer names/accounts/credentials в service telemetry. Service callback никогда не publishes.

**Commit/deploy checkpoint:** review hosting evidence/cost/privacy/schema compatibility и benchmark report; отдельное разрешение commit/deploy необходимой проверенной инфраструктуры и приложения. Forge scopes/egress изменения, если когда-либо понадобятся, требуют отдельного рассмотренного решения; не подразумеваются текущим планом.

## M5 — Публикация roadmap и читаемые обязательства

**Пользовательский результат:** Manager после preview явно публикует версию; Member видит свой план и изменения, stakeholder — разрешённый baseline. Concurrent publish не повреждает current pointer и не обещает общую capacity дважды.

**Entry criteria:** M4 verified Draft; D-03 commitment rule и D-06 viewer grant утверждены. Conditional Context pointer/fencing и shared Team/person/work conflict guard доказаны persistence/concurrency tests; без них этап не готов к pilot.

**Работа по слоям:**

- Backend: publish operation с повторной ACL/input/baseline checks, complete version manifest, authoritative Context pointer, predecessor lineage, delivery status для in-app notification; authorized personal/stakeholder projections.
- Frontend: preview acknowledgement, stale/conflict handling, Publish; published timeline/table, own assignments/change explanations, viewer read-only view и явные deferred/carryover/actual/deadline различия.
- Persistence/migration: Context published pointer и revision, version sequence, publish operations/commitment guards. Перенести legacy Project pointer только к однозначно связанному Context; не копировать current plan каждой Team. Materialize before pointer, reconcile after interruption.
- Solver: ядро без нового objective; проверка complete output и preserved input hash, no solve at publish. Revalidation hard constraints/freshness не подменяет original result неизвестными inputs.
- Reuse/refactor/remove: reuse current immutable draft/pointer workflow; убрать Project как owner published version и синхронные ожидания «optimize значит publish». Browser presentation assets можно сохранить, бизнес-значение перепроверить.

**Acceptance:** PUB-001/PUB-002; AC-005/AC-006/AC-021/AC-042/AC-043/AC-046/AC-049. Один complete current plan; callbacks/approvals не публикуют. Нет автоматического Jira write-back.

**Tests:** concurrent publishers и input activation, crash before/after pointer, idempotent finalization, stale reviewer, revoked actor, same person across Team conflicts, Context-isolated version numbering, visibility-filtered personal/viewer reads.

**Реальный Jira smoke:** два test Managers review/publish competing drafts; Member и Viewer проверяют разрешённые work views; revoke Jira permission и обновить read; reload после simulated recoverable publication failure. Проверить что в Jira не появились несанкционированные write-back изменения.

**Commit/deploy checkpoint:** review pointer/commitment proof и migration pointer mapping, отдельное разрешение commit/deploy; dev acceptance включает реальные actor views и recovery. Этот этап ещё не обещает автоматическое отслеживание исполнения.

## M6 — Полный управляемый closed loop

**Пользовательский результат:** Manager обновляет execution snapshot, рассматривает изменения/запросы, перепланирует и публикует следующую версию; Member понимает изменённое назначение/дату и статус своего запроса. Здесь предлагаемый MVP становится завершённым.

**Entry criteria:** M5 baseline/ACL/commitment guarantees; утверждены D-01 manual refresh+in-app loop, retention/freshness/limits D-09/D-12. Published lineage и immutable source snapshots доступны.

**Работа по слоям:**

- Backend: complete ExecutionSnapshot, detected variances/inbox, PlanningChangeRequest review, approved future-input revisions и replan с current facts/baseline. DONE/actual facts нельзя reject как предложения; scope/member departures дают impact/remaining-work conflicts. Deadline performance отдельно от baseline variance.
- Frontend: refresh/freshness, Manager inbox с source facts/proposals/decision history, own requests/notifications Member, baseline-vs-new preview и понятный change explanation; archive/history navigation.
- Persistence/migration: Context snapshots/variances/requests/input lineage и read/delivery state; перенести existing change/history records без искусственного approval. Pending/rejected historical records не попадают в approved input set. Retention не разрушает обязательную объяснимость published versions.
- Solver: reuse remaining-effort, immutable history, IN_PROGRESS handover и churn baseline; integration tests approved vs pending, newest facts/stale drafts. Никакого automatic relaxation mandatory/deadline.
- Reuse/refactor/remove: reuse execution history dedup, planning change entities, completion-performance utility, reoptimization fixtures; закончить application/runtime/UX orchestration. Убрать project/team-only inbox queries и unsafe diagnostic details.

**Acceptance:** LOOP-001…LOOP-003 и все OPS-001/OPS-002; AC-044…AC-051, включая полный new installation→first publish→change→replan→second publish. Failed replan не меняет baseline; новая версия сохраняет всю lineage.

**Tests:** source actual vs approval distinction, partial snapshot/stale scope, request conflicts/replay, newly approved absence impact, in-progress member departure, replan baseline lineage и notification idempotency; full React journey и service-backed contract integration.

**Реальный Jira smoke:** изменить status/assignee/remaining effort у approved test work, refresh, review impact, добавить approved absence, выполнить replan и publish; Member видит diff, Viewer — только разрешённый baseline. Проверить DONE history и independent CA/CB после повторного входа.

**Commit/deploy checkpoint:** review closed-loop traces и пилотные metrics без customer-content telemetry, отдельное разрешение commit/deploy; owner принимает целостный MVP только после AC-051 и безопасности/истории остальных acceptance gates.

## M7 — Post-MVP автоматизация того же цикла

**Пользовательский результат:** разрешённые фоновые refresh/уведомления уменьшают ручной контроль freshness, сохраняя manager review и explicit publication.

**Entry criteria:** M6 пилот стабилен, owner отдельно утверждает автоматические triggers/channels и D-12 limits; проверены platform event/queue/service permissions и operational costs. Нет обязанности добавлять scope, который не поддерживается платформой.

**Работа по слоям:** backend scheduled/event triggers, dedup/backpressure/reconciliation через existing commands; frontend freshness/notification preferences и source health; persistence job cursors/delivery/retention с additive migration; solver повторно используется, auto replan только по отдельно утверждённой policy, automatic publish не подразумевается. Reuse snapshot/inbox/run ports, убрать ручные технические workarounds, оставить ручной refresh/recovery.

**Acceptance:** те же AUTH/LOOP/OPS guarantees и AC-025/AC-039/AC-044…AC-050 под delayed/duplicate/out-of-order events; новые численные freshness thresholds устанавливаются после measured pilot. Потеря event восполняется reconciliation, не silent drift.

**Tests:** fake-clock trigger/delivery/revocation, rate limits/backpressure, worker retries, source permission changes, coalesced in-app notifications; real Jira smoke с controllable changes и подтверждёнными triggers/permissions. Solver semantics suites без изменений, performance tests при изменении нагрузки.

**Migration / commit/deploy checkpoint:** resumable enablement на выбранных Context, old manual path остаётся доступен; review rollout/disable/recovery и delivery retention; отдельное разрешение commit/deploy, затем dev/pilot наблюдение по approved metrics.

Multi-Team portfolio planning **после** этого не добавляется checkbox multi-select: потребуется отдельный owner-approved vertical program для ContextTeam cardinality, общей UserIdentity capacity/reservations, совместных ACL и work/value overlap. Текущая модель связей готовит этот путь, но M1–M7 не утверждают его продуктовые правила.

## Точные additions и изменения tests при реализации

Новые пути ниже — предложения, файлы **не созданы** в documentation milestone. Сохраняются существующие conventions Node `node:test`, React/Jest и Python `unittest`.

| Suite / действие | Конкретные assertions | Этап / acceptance |
|---|---|---|
| Новый `test/authorization.test.js` | Bootstrap one winner; project admin cannot take existing Team; no observed/discovered auto grants; deny all protected read/write/foreign IDs; revoke/last-owner; diagnostic/viewer visibility | M1; AC-001…AC-006, AC-048 |
| Новый `test/planning-context.test.js`; изменить `test/team-foundation-resolvers.test.js` | Context selection reference validated, tenant/site/actor server-derived, arbitrary payload project/team cannot override; all mapping/WIP/gate/ingest/run/publish calls keep selected Context; no default fallback when ambiguous | M1 onward; AC-004, AC-007…AC-010 |
| Новый `test/context-persistence.test.js`; расширить `test/sql-repositories.test.js` | Unique Context identity/external Team, legacy bindings preserve IDs, separate pointers/revisions, compare-and-set winner, failure after each write. SQL contract test+supported SQL integration, а не только string matching | M1/M3/M5; AC-007, AC-011, AC-027, AC-042, AC-049 |
| Изменить `test/team-foundation.test.js`; новый `test/membership-onboarding.test.js` | Сохранить A→B defensive failure, same-Team last-good, MANUAL/exclusion/history; source late generation rejected; Alice A/B isolation; pending/approve/reject/withdraw/dedup, interrupted membership+grant finalization | M2; AC-012…AC-017 |
| Новый `test/context-capacity.test.js`; расширить profile/capacity cases Team Foundation | Own proposals not effective until approval; roles!=ACL; same identity Team profiles independent; approved calendar/absence overlap applied once; Context shares no double count; archived/excluded resource history | M2/M5; AC-018…AC-021 |
| Изменить `test/jira-adapter.test.js`; новый `test/context-ingestion.test.js`; расширить `test/persistence-ingestion.test.js` | Team ID matching bare/ARI, unassigned inclusion, board/project bounds, all pages, scope/readiness completeness; permission loss not deletion; source key/project move; two Context overlays never overwrite each other | M3; AC-022…AC-027 |
| Изменить `test/planning-model-contract.test.js` и `test/planning-semantics-integration.test.js` | Versioned Context manifest and per-source mapping profiles; independent scope/member sets; pending inputs excluded; source/config revisions frozen atomically in observable contract | M3/M4; AC-026…AC-031 |
| Изменить `test/planning-semantics.test.js` | Future platformDeadline alone leaves gate unresolved; explicit assumption/DONE evidence permit; no assumption without actor/provenance; hierarchy not F2S, Epic configurable, group overlap no double count, WIP precedence retained | M3/M4; AC-028…AC-034 |
| Новый `test/optimization-workflow.test.js`; расширить `test/application-core.test.js` | Duplicate submit/attempt, cancel/timeout/UNKNOWN, input/output hash and tenant, invalid output not draft, partial materialization hidden, revoked actor, stale baseline/facts, objective status honest | M4; AC-035…AC-041, AC-049 |
| Новый `test/roadmap-publication.test.js` | Context pointer versus Project; complete manifest; concurrent publish/input activation; shared Team/person/work commitment conflict; before/after-pointer crash; old version lineage; viewer/own-plan visibility | M5; AC-021, AC-042/AC-043, AC-046 |
| Новый `test/planning-change-loop.test.js` | Fact DONE cannot be rejected as reality; proposal approval not publish; partial/newest snapshot; remaining effort/handover; request conflict/retry; failed replan leaves pointer; second publication immutable lineage | M6; AC-044…AC-047, AC-051 |
| Новый `test/diagnostic-privacy.test.js`; изменить diagnostic expectations SQL/application suites | Raw error bodies/messages/account IDs fixture никогда не logs/telemetry; controlled audit access/retention; exact raw-details test заменён minimum-safe evidence expectation | M1 onward; AC-048/AC-050 |
| Изменить `static/hello-world/src/App.test.js` и `TeamSetup.test.js` | Перейти с externalTeam selection на Context ID; A/B people/work/config/plan return; delayed results ignored; cancel/pending/legacy/access states; no auto Context ownership; manual identity retained | M1/M2; AC-007…AC-017 |
| Новые React suites `PlanningContext.test.js`, `AccessAndOnboarding.test.js`, `PlanningInputs.test.js`, `ScenarioReview.test.js`, `PublishedRoadmap.test.js`, `PlanningInbox.test.js` | Реальные user actions, keyboard/accessibility labels, loading/empty/error/denied/revision-conflict, readiness corrections, run resume, preview→publish, request decisions и changed personal work | M1…M6; соответствующие AC и AC-051 |
| Изменить `static/hello-world/src/optimizer.test.js` только при удалении demo | Удалённый production knapsack не сохранять как второй optimizer; value-choice coverage перенести в Python contract/verification, UI тестирует service result/status, а не повтор алгоритма | M4; AC-035/AC-037…AC-039 |
| Сохранить `optimizer/tests/test_*.py`, добавить `test_service_contract.py` и при необходимости `test_termination.py` | Existing hard constraints/DAILY/PERSON_DAYS/no-productivity/F2S/WIP/groups/overflow/handover objectives; versioned envelope IDs, bounded termination не INFEASIBLE, invalid output, stable fixture interpretation | M4/M6; AC-019, AC-028…AC-040 |
| Сохранить `optimizer/verification/case_01…case_08` и `run_verification.py` | Hand-verifiable value=12 combination, dependency+deadline, handover choice, mandatory carryover, churn tie, completion cost, exclusion, mandatory excluded-prerequisite conflict | M4 и regression M6 |

Заведомо неправильное current expectation меняется с объяснением (например PLATFORM_DEADLINE assumption или project-global owner). Его нельзя оставлять «для обратной совместимости», если оно нарушает утверждённый domain principle. При этом last-good/manual override/history/security regression tests сохраняются.

## Проверенные entry points для будущей validation

Команды ниже описывают запуск **при реализации**, а не выполнены сейчас. Использовать уже имеющееся окружение; отсутствующие dependencies/CLI фиксировать как blocker, не устанавливать без разрешения.

Из repository root:

```sh
node --test test/*.test.js
node --check src/index.js
git diff --check
git status --short
```

`node --check` повторить для каждого фактически изменённого backend JS. Root package.json не определяет `npm test`; для focused run передавать конкретные Node test paths из таблицы. После focused tests расширять только по затронутым boundaries/обязательным release checks.

Из `static/hello-world`:

```sh
CI=true npm test -- --watchAll=false --runInBand
npm run build
```

Из `optimizer`, Python окружением с уже установленными requirements:

```sh
python3 -m unittest discover -s tests -v
python3 -m verification.run_verification
python3 -m benchmarks.benchmark_daily --scenario SMALL
python3 -m benchmarks.benchmark_daily --scenario MEDIUM
```

Fixtures/benchmarks имеют отдельные цели: arithmetic/semantic regression и measured performance соответственно. Benchmark не обещает SLA и при bounded solve может быть FEASIBLE. Новые integration tests не должны повторять весь solver в frontend.

Для Forge CLI сначала выполнить `pwd` в repository root, использовать полученный cwd и затем `forge lint`. Failure output включать в отчёт. Lint не подтверждает runtime permissions, выполненные migrations или готовность solver hosting; это проверяет отдельно разрешённый Jira smoke.

## Трассировка требований по основному milestone

| Milestone | Requirement IDs |
|---|---|
| M1 | AUTH-001…AUTH-004, CTX-001/CTX-002, TEAM-001; OPS-001/OPS-002 для foundation |
| M2 | TEAM-002…TEAM-004, PROF-001/PROF-002, CAP-001/CAP-002 |
| M3 | SCOPE-001/SCOPE-002, MAP-001/MAP-002, ING-001, WORK-001/WORK-002, DEP-001/DEP-002, READY-001 |
| M4 | PLAN-001…PLAN-003, OPT-001…OPT-003, SCEN-001; OPS-001/OPS-002 execution |
| M5 | PUB-001/PUB-002; AUTH/OPS/CTX cross-domain publication checks |
| M6 | LOOP-001…LOOP-003; все OPS-001/OPS-002 и сквозной AC-051 |
| M7 | Те же LOOP/OPS/AUTH contracts под background triggers; отдельные owner-approved nonfunctional thresholds |

## PRODUCT DECISIONS REQUIRING OWNER APPROVAL

Полный реестр с вопросом, важностью, Option A/B, рекомендацией и последствиями: [PRODUCT_VISION — D-01…D-13](../product/PRODUCT_VISION.md#product-decisions-requiring-owner-approval).

До M1 нужны ownership/legacy/access decisions; до M2 — grants/catalogs/privacy/общая capacity; до M3 — source bounds/value/gates/timezone; до M4/M5 — budget и commitment/publication guarantees; до M6 — граница closed loop и retention. Рекомендуемый порядок не подразумевает утверждения этих вариантов. Проверка Forge инфраструктуры — самостоятельный technical gate, а не молчаливо выбранная deployment technology.
