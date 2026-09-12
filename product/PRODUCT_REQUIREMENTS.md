# RM Optimizer — Product Requirements

Статус: DESIGN DRAFT, 12 сентября 2026. Целевые требования; доступность в коде проверяется отдельно в [gap analysis](../architecture/CURRENT_TO_TARGET_GAP.md).
Основание: [CJM](PERSONAS_AND_CJM.md), [vision и открытые D-решения](PRODUCT_VISION.md#product-decisions-requiring-owner-approval). Все рекомендации D-01…D-13 остаются OPEN.

## Общий контракт

Каждая команда получает authenticated actor и installation из доверенного invocation/service context. Выбранный `planningContextId` — недоверенная ссылка: сервер загружает Context, проверяет tenant, ACL и связанные сущности. Payload не может подменять installation/site/project/Team, вычисленные сервером. Jira permission и RM permission проверяются независимо. Чтение тоже авторизуется.

Для изменяющих команд обязательны operation ID, ожидаемая revision и разрешённые поля. Повтор возвращает тот же результат; конфликт revision требует reload/review. Pending, rejected, failed и частично записанные операции не становятся действующими входами. Ответ содержит Context ID, revision, freshness и безопасный error code. UI не применяет ответ к другому выбранному Context.

Ниже «аудит» означает защищённую tenant-запись решения с actor reference, scope, временем, revision и причиной там, где она нужна. Это не console/telemetry. Техническая telemetry содержит allowlisted codes, counts, durations и correlation ID, без account IDs, customer content, raw API bodies и секретов. Общие ошибки: UNAUTHENTICATED, FORBIDDEN, NOT_FOUND, REVISION_CONFLICT, VALIDATION_FAILED, RETRYABLE_FAILURE. Недоступная чужая сущность не раскрывается через детали ошибки.

## Installation и authorization

### AUTH-001 — Installation и безопасный первый владелец

- **Actor / предусловия:** подтверждённый installation/site authority; установленное приложение, ещё нет действующего RM ownership.
- **Trigger / основной поток:** первый вход → проверка credential сервером → явное назначение RM_ADMIN → назначение первого Manager для Team/Context. D-04 рекомендует подтверждённое назначение, способ проверки ещё требует технической верификации.
- **Альтернативы:** обычный пользователь видит запрос доступа; существующее ownership переводит вход в обычную ACL-проверку. Jira Project Admin может подать bootstrap request, не захватить существующий Team.
- **Валидация:** единственная завершённая bootstrap operation; credential связан с installation; конкурентный запрос не перезаписывает владельца.
- **Persisted state:** installation lifecycle, bootstrap decision, scoped grants, operation result.
- **Permissions / ошибки:** только проверенный authority; BOOTSTRAP_NOT_AVAILABLE, CREDENTIAL_UNVERIFIED, OWNERSHIP_EXISTS. Неполная операция не выдаёт права.
- **Результат / аудит и telemetry:** управляемая installation или pending access; кто/на каком основании назначил владельца, success/conflict counts без credential contents.

### AUTH-002 — Scoped access и проверка каждой операции

- **Actor / предусловия:** любой пользователь или доверенный worker; identity проверена, grant может отсутствовать.
- **Trigger / основной поток:** read/write → resolve installation и Context/Team → проверить требуемую capability и Jira visibility → проверить foreign references → исполнить.
- **Альтернативы:** RM_MEMBER редактирует только собственные предложения; viewer читает опубликованный разрешённый срез по D-06; background actor действует только в сохранённой service policy.
- **Валидация:** discovery, assignee, project role и platform membership не создают Manager/Admin. Grant Team не даёт управление другим Context без соответствующего grant; grants не обходят Jira access.
- **Persisted state:** RM grants и их revisions; решение об отказе не меняет бизнес-данные.
- **Permissions / ошибки:** capability matrix из CJM; FORBIDDEN и ограниченный NOT_FOUND без раскрытия чужих данных.
- **Результат / аудит и telemetry:** разрешённая операция либо отсутствие эффекта; аудит важных grant/write решений, агрегаты denial, без перечисления скрытых issues.

### AUTH-003 — Жизненный цикл управления

- **Actor / предусловия:** RM_ADMIN с governance capability; существующие grants.
- **Trigger / основной поток:** смена Manager, отзыв доступа или archive → preview затрагиваемых scopes → явное подтверждение → новая grant revision → прекращение последующих привилегированных действий.
- **Альтернативы:** менеджер передаёт запрос администратору; восстановление доступа создаёт новую запись, не переписывает историю. Последний владелец требует заранее подтверждённой замены/процедуры recovery.
- **Валидация:** нельзя оставить installation без recovery authority; нельзя самому повысить роль; pending worker result не публикуется за отозванного actor.
- **Persisted state:** grants ACTIVE/REVOKED, ownership history, revocation time, recovery decision.
- **Permissions / ошибки:** governance не означает автоматическое чтение customer plans; LAST_OWNER, STALE_GRANT, FORBIDDEN.
- **Результат / аудит и telemetry:** права изменены, данные и baseline сохранены; обязательный аудит до/после grants и invalidation outcome.

### AUTH-004 — Безопасный вход и доступ к опубликованному плану

- **Actor / предусловия:** посетитель, Member, Manager или stakeholder; authenticated installation identity.
- **Trigger / основной поток:** открыть приложение/ссылку → показать разрешённые Context и опубликованный срез → проверить доступ повторно при загрузке detail.
- **Альтернативы:** без grant — eligible Team directory и access/join request по утверждённой policy D-13; viewer получает VIEW_PUBLISHED без превращения в ресурс, если утверждено D-06. Draft доступен только явным reviewer/manager capabilities.
- **Валидация:** user-controlled URL не даёт доступ к чужому Context; скрытые Jira issues и их производные totals не раскрываются.
- **Persisted state:** предпочтение последнего разрешённого Context; запрос доступа, если отправлен.
- **Permissions / ошибки:** минимальный directory view не включает людей/работы/причины отсутствий; ACCESS_PENDING, ACCESS_REVOKED, PLAN_NOT_PUBLISHED.
- **Результат / аудит и telemetry:** допустимый экран или понятный путь запроса; access request audit, entry/empty-state counts.

## Planning Context

### CTX-001 — Создание объекта планирования

- **Actor / предусловия:** Manager с правом создания Context и присоединения Team либо подтверждённый bootstrap actor.
- **Trigger / основной поток:** Create Context → название → явно связать Planning Team → sources/scope → horizon/timezone → сохранить Context в SETUP и продолжить readiness.
- **Альтернативы:** ручная Team без Atlassian identity; несколько Context одной Team как альтернативные scenarios/непересекающиеся commitments по D-03. MVP связывает одну Team, модель хранит явную связь.
- **Валидация:** уникальный внутренний Context ID; имя/project/board не являются ключом; связанная Team существует в tenant и доступна; создание не предоставляет новые grants неявно.
- **Persisted state:** Context, ContextTeam relation, configuration revisions, explicit grants, last-selected preference.
- **Permissions / ошибки:** create/link capabilities; TEAM_NOT_AUTHORIZED, INVALID_HORIZON, CONTEXT_INCOMPLETE.
- **Результат / аудит и telemetry:** отдельный Context с собственной конфигурацией; create/link audit и setup progression.

### CTX-002 — Переключение, возврат и archive

- **Actor / предусловия:** пользователь с доступом к двум Context или Manager для archive.
- **Trigger / основной поток:** выбрать Context B → очистить A из активного представления → загрузить B people/work/config/plan по одной revision envelope. Возврат в A восстанавливает его сохранённое состояние.
- **Альтернативы:** revoked/archived Context показывает состояние недоступности; несохранённый scenario требует save/discard decision. Archive сохраняет историю и запрещает новые runs/publish.
- **Валидация:** ответы A после switch отбрасываются; switching не меняет Team identity/membership/scope; все мутации проверяют исходный Context и revision.
- **Persisted state:** только preference при switch; archive status и audit при явном archive.
- **Permissions / ошибки:** read Context / manage lifecycle; LOAD_FAILED без смешанного экрана, STALE_CONTEXT, ARCHIVED.
- **Результат / аудит и telemetry:** согласованный B либо его error state; archive audit, switch latency/failure без customer names.

## Team и membership

### TEAM-001 — Стабильная Team identity и безопасная discovery

- **Actor / предусловия:** Manager; разрешённый Jira entry/source context и право связать Team.
- **Trigger / основной поток:** discover → показать external stable IDs с display names → явный выбор → resolve/create installation Planning Team по normalized externalTeamId → attach к Context.
- **Альтернативы:** нет candidates — manual setup; несколько — обязательный выбор; уже существующая Team требует её ACL, discovery не grants. Неоднозначное legacy binding направляется на review D-11.
- **Валидация:** bare/ARI нормализуются; одинаковое имя не объединяет ID; повторное обнаружение из другого проекта возвращает ту же Team. При смене обычной Team выбирается другой Context/создаётся явная связь, Team A не мутирует в B.
- **Persisted state:** Planning Team, external identity link, ContextTeam link; discovery result не membership.
- **Permissions / ошибки:** manage Context + attach Team; LEGACY_BINDING_REQUIRED, TEAM_NOT_AUTHORIZED, DISCOVERY_UNAVAILABLE.
- **Результат / аудит и telemetry:** стабильная идентичность независимо от доступности members API; link/binding audit, candidate counts.

### TEAM-002 — Platform sync и manual режим

- **Actor / предусловия:** Manager/System по разрешённой policy; source явно выбран.
- **Trigger / основной поток:** Retry sync → получить полный валидный snapshot → обновить platform membership для той же Team/source generation → объединить с одобренными additions, вычесть exclusions.
- **Альтернативы:** identity API недоступна — сохранить external identity и last-known-good, предложить ручной onboarding; MANUAL исключает platform слой, сохраняя manual overrides/history. Защитная явная смена source A→B инвалидирует A до B sync.
- **Валидация:** partial/failure не authoritative empty; same-Team failure не удаляет last-good; поздний A result не активирует A под B. GraphQL/scopes/auth workaround запрещён.
- **Persisted state:** source generation, sync status/freshness, platform relations и inactive history, manual overrides.
- **Permissions / ошибки:** manage membership или service sync capability; IDENTITY_SYNC_UNAVAILABLE, PARTIAL_SYNC, SOURCE_CHANGED.
- **Результат / аудит и telemetry:** корректное effective membership с видимой freshness; source-change audit, только безопасные classifications/scopes/correlation telemetry.

### TEAM-003 — Manager additions и exclusions

- **Actor / предусловия:** Manager данной Team; доступный Jira user search, выбранный человек с устойчивой identity.
- **Trigger / основной поток:** найти → проверить дубликат → добавить manual membership и явно согласованный RM_MEMBER access → отдельно настроить planning profile. Exclude создаёт исключение, оно выигрывает у любого inclusion.
- **Альтернативы:** уже включён — идемпотентный успех; снять exclusion требует явной операции; удаление членства не удаляет resource/history. По D-07 platform evidence само по себе не RM grant.
- **Валидация:** assignees не добавляются; принадлежность A не распространяется на B; утверждённое membership не означает роль Developer или ненулевую capacity.
- **Persisted state:** provenance MANAGER_ADDED, inclusion/exclusion, explicit grant operation, resource identity reference.
- **Permissions / ошибки:** manage Team membership, допустимый basic grant; USER_UNAVAILABLE, REVISION_CONFLICT, GRANT_FINALIZATION_PENDING.
- **Результат / аудит и telemetry:** Team-specific effective relation после завершения операции; actor/subject references только в защищённом audit, aggregate addition/exclusion counts.

### TEAM-004 — Self-enrollment с решением Manager

- **Actor / предусловия:** authenticated requester и Manager выбранной eligible Team; утверждённая directory/request policy D-13 допускает обращение.
- **Trigger / основной поток:** Request join → PENDING → Manager review → APPROVED или REJECTED. Approval завершает membership и RM_MEMBER grant одной возобновляемой операцией; UI не объявляет ACTIVE до её завершения.
- **Альтернативы:** withdraw до решения; duplicate pending возвращает исходный request; exclusion требует явного решения снять его, approval не обходит запрет автоматически.
- **Валидация:** requester не утверждает себя; Manager авторизован на момент решения; повтор approve/reject не создаёт дубликатов; source sync не approves.
- **Persisted state:** request lifecycle, decision reason, decision actor, operation completion, membership/grant references.
- **Permissions / ошибки:** submit own / review Team; REQUEST_CLOSED, EXCLUDED_MEMBER, APPROVAL_PENDING, FORBIDDEN.
- **Результат / аудит и telemetry:** pending/rejected не дают membership/access; approved завершённый join даёт RM_MEMBER; request/decision audit и in-app уведомление сторонам.

## Planning profiles, capacity и availability

### PROF-001 — Каталоги профессий и skills

- **Actor / предусловия:** Manager с catalog capability, governance для общих изменений по D-08.
- **Trigger / основной поток:** создать стабильный role/skill ID → настроить описание/архивацию → выбрать primary/additional roles и skills в Team resource profile и requirements работ.
- **Альтернативы:** сходные имена остаются разными ID до явного merge review; archived запись доступна для истории, запрещена новым назначениям.
- **Валидация:** planning role != RM role; не выводить профессию из Jira roles/Team roles/assignee. Переименование не меняет eligibility ID.
- **Persisted state:** versioned catalogs, Team profile references, Context requirement revisions.
- **Permissions / ошибки:** catalog management отдельно от своего proposal; ROLE_IN_USE, UNKNOWN_SKILL, CATALOG_CONFLICT.
- **Результат / аудит и telemetry:** объяснимые eligibility constraints; catalog/profile decision audit, unmapped requirements counts.

### PROF-002 — Профиль человека и review предложений

- **Actor / предусловия:** Member своей Team и её Manager; effective membership либо доступ к own onboarding profile.
- **Trigger / основной поток:** Member предлагает роли/skills → PENDING → Manager approve/reject → новая approved profile revision; менеджер может внести разрешённое явное изменение с audit.
- **Альтернативы:** старые approved данные остаются до решения; отклонённое предложение можно заменить новым; изменение membership не удаляет profile history.
- **Валидация:** чужой account/resource ID не меняет subject; proposal не меняет solver eligibility; одинаковый человек в A/B имеет независимые Team profiles.
- **Persisted state:** proposal, review, approved profile revisions, optional identity-level display data.
- **Permissions / ошибки:** edit own proposal / approve Team profile; STALE_PROFILE, INVALID_ROLE, FORBIDDEN.
- **Результат / аудит и telemetry:** понятный pending/approved/rejected статус; in-app handoff Member↔Manager, audit changes и review duration.

### CAP-001 — Рабочая capacity и распределение в Context

- **Actor / предусловия:** Manager; Team resource profile, календарь и horizon.
- **Trigger / основной поток:** задать normal-day fraction, рабочую неделю, holidays и Context allocation share → preview daily person-days → approve revision → readiness recompute.
- **Альтернативы:** отсутствует календарь/доля — NEEDS_INPUT; нулевая доступность допустима; D-03 блокирует конфликт committed horizons, пока нет общей reservation модели.
- **Валидация:** fraction в допустимом диапазоне, без двойного суммирования allocations; capacity уменьшается approved absences; effort задачи одинаков для всех eligible людей. Исторический календарь не перезаписывается.
- **Persisted state:** Team calendar/profile versions, Context resource allocation, compiled capacity reference.
- **Permissions / ошибки:** manage Team capacity и Context allocation; OVERALLOCATION, CAPACITY_MISSING, INVALID_DATE_RANGE.
- **Результат / аудит и telemetry:** daily capacity с объяснением источников; audit approvals, days/capacity aggregates без absence reasons.

### CAP-002 — Availability requests

- **Actor / предусловия:** Member и Manager Team; известны person identity и даты.
- **Trigger / основной поток:** собственный vacation/availability request → PENDING → review → APPROVED/REJECTED → approved event меняет будущую capacity и создаёт planning change для затронутого Context.
- **Альтернативы:** withdraw/cancel approved event через новую reviewed revision; overlap merge/validation без повторного вычитания; причина ограничена D-09, план показывает доступную capacity.
- **Валидация:** даты/timezone, диапазон fraction, ownership, конфликт revisions; pending не уменьшает capacity, уже опубликованный baseline не переписывается.
- **Persisted state:** requests, decisions, effective availability events, impact references.
- **Permissions / ошибки:** submit own / approve affected Team; INVALID_AVAILABILITY, STALE_REQUEST, FORBIDDEN.
- **Результат / аудит и telemetry:** подтверждённое изменение входов и уведомление Manager/Member; защищённый decision audit, counts/latency без причин отсутствия.

## Work Scope, ingestion и mappings

### SCOPE-001 — Явный Work Source и Work Scope

- **Actor / предусловия:** Manager Context, доступ к выбранным Jira sources.
- **Trigger / основной поток:** выбрать project set и ATLASSIAN_TEAM_FIELD либо BOARD_SCOPE → preview количества/границ → подтвердить scope revision. Team-field сравнивает stable external ID; board использует явно выбранный board, а не entry board.
- **Альтернативы:** manual membership с Team-field scope допустимо; CUSTOM_JQL обозначен future и не принимает произвольный payload. D-02 рекомендует board∩project set.
- **Валидация:** membership/assignee не являются фильтром; unassigned matching issues включены; source permission проверяется; отсутствующий stable field требует настройки, не fallback на имя/assignee.
- **Persisted state:** Context sources, versioned scope definition, field references, preview acknowledgement.
- **Permissions / ошибки:** manage scope + source visibility; TEAM_FIELD_UNRESOLVED, SOURCE_FORBIDDEN, INVALID_SCOPE.
- **Результат / аудит и telemetry:** воспроизводимое множество кандидатов; scope-change audit, bounded source/page counts.

### SCOPE-002 — Согласованный список работ и полнота чтения

- **Actor / предусловия:** authorized Context user; сохранённый scope.
- **Trigger / основной поток:** открыть work list/refresh → все страницы scope → показать inclusion provenance, hierarchy kind, фильтры отображения и freshness. Solver использует тот же approved scope revision, а не UI-фильтр.
- **Альтернативы:** read-only inspection до mapping; partial/source revoked показывает неполноту и блокирует новый trusted input set; старый snapshot явно помечен устаревшим.
- **Валидация:** project запуска не сужает Context; siblings/descendants не теряются из-за отображения только Epic; изменение display filter не меняет membership или scope.
- **Persisted state:** ingestion generation и completeness, snapshot reference; персональные UI-фильтры отдельно.
- **Permissions / ошибки:** read Context + Jira visibility; PARTIAL_SOURCE, STALE_SNAPSHOT, LOAD_FAILED.
- **Результат / аудит и telemetry:** согласованный список либо объяснение недоступности; page/completeness metrics, без issue contents.

### MAP-001 — Versioned Field Mapping

- **Actor / предусловия:** Manager Context; schema metadata каждого source project.
- **Trigger / основной поток:** сопоставить stable fields/status/type IDs с canonical effort/value/deadline/status/requirements → preview samples с доступом → validate → activate version для Context+source.
- **Альтернативы:** template копируется явно; один mapping не навязывается проектам с другим schema; unknown status/type требует review. Unmapped raw work можно безопасно смотреть.
- **Валидация:** effort unit PERSON_DAYS, допустимые transforms, числа/даты/status enums; имя поля не достаточный ключ; source-owned actual и manager override различаются.
- **Persisted state:** immutable mapping versions, activation pointer, validation results, transform provenance.
- **Permissions / ошибки:** manage mappings, source read; INVALID_MAPPING, INCOMPATIBLE_SCHEMA, STALE_PREVIEW.
- **Результат / аудит и telemetry:** выбранная воспроизводимая mapping revision; activation audit, error-code/field-category counts без samples.

### MAP-002 — Исправление canonical inputs

- **Actor / предусловия:** Manager, а Member — только предложить own change; импорт с missing/invalid inputs.
- **Trigger / основной поток:** открыть readiness finding → исправить mapping либо сделать явно помеченный Context override → preview impact → approve → новая input revision.
- **Альтернативы:** Jira edit выполняет пользователь вне RM; следующий refresh обнаруживает изменение. Pending proposal и rejected input не заменяют approved value.
- **Валидация:** нулевой effort не маскирует missing estimate; placeholder business value запрещён; ручное изменение не пишет Jira автоматически; конфликт source/override видим.
- **Persisted state:** overrides/proposals с provenance, approval, superseded revisions.
- **Permissions / ошибки:** manage inputs / propose own; INVALID_EFFORT, MISSING_VALUE, CONFLICTING_UPDATE.
- **Результат / аудит и telemetry:** исправлен конкретный finding либо сохранён блокер; reasoned audit, resolved-finding counts.

### ING-001 — Canonical ingestion и снимок источников

- **Actor / предусловия:** Manager-triggered System; утверждённые sources/scope/mappings и Jira access.
- **Trigger / основной поток:** создать generation → прочитать все страницы/relations → нормализовать по стабильному issue ID и platform site → сверить completeness → сохранить immutable source snapshot и Context projection → выставить readiness.
- **Альтернативы:** rate limit/retry возобновляет generation; partial не удаляет ранее известные записи; удалённые/out-of-scope issues после полного чтения получают явный state и impact, не cascade-delete history.
- **Валидация:** один issue может входить в разные Context с независимыми planning overlays; Jira key/name/project move не перепривязывает Team; hierarchy != dependency.
- **Persisted state:** source observations, snapshot, ContextWorkItems/relations, provenance, completeness cursor/manifest.
- **Permissions / ошибки:** service acting policy и Jira visibility; PARTIAL_INGESTION, SCHEMA_DRIFT, STALE_SCOPE.
- **Результат / аудит и telemetry:** inspectable canonical projection и факт изменений; generation audit, page/duration/retry counts.

## Work model, dependencies и readiness

### WORK-001 — Executable work и hierarchy

- **Actor / предусловия:** Manager; imported work and stable type IDs.
- **Trigger / основной поток:** назначить explicit EXECUTABLE/CONTAINER policy → inspect parent-child tree → определить обязательные canonical поля executable leaves/самостоятельных узлов.
- **Альтернативы:** Epic может быть executable; display-only ancestor вне scope показывается как context reference и не получает скрыто effort/value/selection.
- **Валидация:** hierarchy cycles/unknown kinds блокируют compilation; parent-child не создаёт F2S; siblings могут идти параллельно; containers не получают ресурсное назначение как executable task.
- **Persisted state:** Context structure policy versions, structure relations, work kind projection.
- **Permissions / ошибки:** manage planning model / read projection; AMBIGUOUS_KIND, HIERARCHY_CYCLE.
- **Результат / аудит и telemetry:** однозначный планируемый граф отдельно от дерева отображения; policy audit, kind/error counts.

### WORK-002 — Deliverable Value без двойного счёта

- **Actor / предусловия:** Manager; explicit structure и value scale.
- **Trigger / основной поток:** выбрать value-owner и executable descendants deliverable → preview aggregate → approve. Value группы засчитывается только при выполнении всей требуемой группы в target horizon.
- **Альтернативы:** atomic executable value без группы; незавершённая/overflow группа не приносит целевую delivered value. Nested overlaps требуют D-05 review, не автоматической суммы.
- **Валидация:** каждый учитываемый deliverable имеет однозначное ownership; descendants и group value не считаются дважды; значения допустимой точности/диапазона; DONE work не создаёт повторно будущую value.
- **Persisted state:** Context deliverable groups, value ownership, mapping/override revision.
- **Permissions / ошибки:** manage value model; OVERLAPPING_VALUE_OWNERS, INVALID_VALUE, INCOMPLETE_GROUP.
- **Результат / аудит и telemetry:** объяснимый total и blockers; ownership/value decision audit, overlap counts.

### DEP-001 — Явные внутренние dependencies

- **Actor / предусловия:** Manager/System; canonical issues и link mapping.
- **Trigger / основной поток:** импортировать/утвердить directed Blocks/prerequisite → preview → classify internal dependency, если оба executable endpoints входят в Context.
- **Альтернативы:** вручную предложенная зависимость проходит review; DONE prerequisite удовлетворён; endpoint вне Context направляется в DEP-002.
- **Валидация:** direction, self/cycle, отсутствующие endpoints; F2S означает старт после даты завершения prerequisite, с учётом daily capacity; parent relation не преобразуется в dependency.
- **Persisted state:** Context dependency versions, source link provenance, review outcome.
- **Permissions / ошибки:** manage dependencies / source read; DEPENDENCY_CYCLE, INVALID_ENDPOINT, AMBIGUOUS_LINK.
- **Результат / аудит и telemetry:** корректный precedence graph или actionable blocker; edit/classification audit, cycle/gate counts.

### DEP-002 — External Dependency Gates

- **Actor / предусловия:** Manager; dependency endpoint вне Context scope, доступная ограниченная source evidence.
- **Trigger / основной поток:** показать unresolved gate → подтвердить DONE evidence либо явно задать assumed completion date/reason → approve gate revision → preview affected work.
- **Альтернативы:** unknown gate остаётся blocker для требующей его работы; optional successor можно явно exclude/defer. Недоступный endpoint не считается DONE. Platform Due Date — подсказка, не автоматически разрешённый gate (D-10).
- **Валидация:** никакой скрытой даты; assumption имеет owner, дату, evidence/freshness; downstream starts строго после completion date; изменённый факт создаёт review.
- **Persisted state:** gate, resolution type/evidence, approved assumption, revisions/history.
- **Permissions / ошибки:** approve assumptions Context; UNRESOLVED_GATE, EXTERNAL_SOURCE_UNAVAILABLE, STALE_ASSUMPTION.
- **Результат / аудит и telemetry:** явно разрешённый gate либо блокер; обязательный assumption audit и безопасный unresolved count.

### READY-001 — Readiness и freeze согласованных входов

- **Actor / предусловия:** Manager; Context setup/import available.
- **Trigger / основной поток:** validate → findings по людям, capacity, scope completeness, mapping, graph, gates, horizon/policies → исправление → READY → freeze ApprovedPlanningInputSet с source/config revisions.
- **Альтернативы:** inspection доступна при NEEDS_INPUT; изменившаяся revision делает READY устаревшим; pending changes показываются отдельно и не входят в freeze.
- **Валидация:** отсутствие ошибок структуры/input обязательно; readiness не обещает solver feasibility. Удаление необязательной задачи — явное управленческое решение, не автоматическое сокрытие ошибок.
- **Persisted state:** findings с entity references, validation version, approved input manifest/hash.
- **Permissions / ошибки:** validate/read findings / approve inputs; NOT_READY, INPUT_CHANGED, SOURCE_INCOMPLETE.
- **Результат / аудит и telemetry:** reproducible approved input либо список действий; approval audit, readiness duration/blocker categories.

## Planning policies

### PLAN-001 — Горизонт, effort и календарные даты

- **Actor / предусловия:** Manager; Context и approved resources.
- **Trigger / основной поток:** задать start, targetEnd, scheduleEnd и timezone → preview daily calendar → approve. ScheduleEnd явно допускает truthful carryover после target.
- **Альтернативы:** слишком короткий schedule horizon даёт infeasibility для обязательств; менеджер расширяет его явно. Fractional effort округляется compiler по опубликованной точности, не по productivity.
- **Валидация:** start ≤ targetEnd ≤ scheduleEnd; canonical PERSON_DAYS, DAY resolution; effort 5 при capacity 0.5 требует 10 доступных рабочих дат; никаких скрытых overtime/horizon extension.
- **Persisted state:** horizon version, calendar/precision references, Context timezone.
- **Permissions / ошибки:** manage horizon; INVALID_HORIZON, UNSUPPORTED_PRECISION, CALENDAR_MISSING.
- **Результат / аудит и telemetry:** одинаковые даты для пользователей; horizon change audit, overflow counts.

### PLAN-002 — Hard work constraints и execution commitments

- **Actor / предусловия:** Manager; canonical work, approved execution snapshot при replanning.
- **Trigger / основной поток:** include/exclude, mandatory, assignment pin, deadline, remaining effort и допустимый handover → preview conflicts → approve constraint revision.
- **Альтернативы:** optional work может быть deferred; mandatory/IN_PROGRESS остаётся обязательным, DONE сохраняет history и не планирует будущий effort. Reassignment начатой работы включает явный handover effort.
- **Валидация:** selected deadline hard; pin требует eligible resource/capacity; exclude prerequisite обязательной работы — conflict; actual start не переписывается; baseline dates не превращаются автоматически в deadlines.
- **Persisted state:** constraints, execution evidence, approved changes, handover policy.
- **Permissions / ошибки:** manage constraints; CONFLICTING_CONSTRAINTS, INELIGIBLE_PIN, INVALID_REMAINING_EFFORT.
- **Результат / аудит и telemetry:** прозрачные обязательства без скрытого relaxation; constraint reason audit, conflict categories.

### PLAN-003 — SIMPLE и ADVANCED WIP

- **Actor / предусловия:** Manager; Team profiles/primary roles и Context policy.
- **Trigger / основной поток:** выбрать SIMPLE default либо ADVANCED overrides → preview effective limits по каждому resource/type → approve revision.
- **Альтернативы:** SIMPLE не уничтожает сохранённые advanced settings; при возврате они снова действуют. Defaults и inheritance показываются явно.
- **Валидация:** WIP hard; general precedence resource > primary role > default, type precedence resource-type > primary-role-type; general и type caps соблюдаются вместе. Активный интервал задачи занимает WIP даже в день без allocation.
- **Persisted state:** policy version, defaults/overrides, effective compiled limits.
- **Permissions / ошибки:** manage policies; INVALID_WIP_LIMIT, MISSING_PRIMARY_ROLE, POLICY_CONFLICT.
- **Результат / аудит и telemetry:** воспроизводимые concurrency constraints; policy audit, effective-limit counts.

## Optimization, scenario и publish

### OPT-001 — Асинхронный запуск Python solver

- **Actor / предусловия:** Manager Context; READY immutable approved input set и допустимый baseline.
- **Trigger / основной поток:** Optimize → создать run/idempotency key → QUEUED → worker RUNNING → versioned canonical input validation → CP-SAT → verified result → завершённый run и готовый Draft либо diagnostic outcome.
- **Альтернативы:** повтор клика возвращает run; refresh страницы восстанавливает progress; cancel/failure/retry сохраняет attempt history. Поздний result не меняет текущий Context и не публикуется.
- **Валидация:** tenant/Context/run/hash/schema match; browser knapsack не участвует; worker получает минимальный canonical input, без Jira credentials; invalid output не материализуется.
- **Persisted state:** run state, attempts, input hash/revisions, solver version/settings, output manifest, idempotency result.
- **Permissions / ошибки:** optimize Context, service execution token; QUEUE_FAILURE, INVALID_SOLVER_OUTPUT, RUN_CANCELLED, SERVICE_UNAVAILABLE.
- **Результат / аудит и telemetry:** inspectable run независимо от browser lifetime; submit/cancel/finalize audit, queue/run duration, retry/result classifications.

### OPT-002 — Feasible value portfolio и честный результат

- **Actor / предусловия:** System для Manager; валидный input и compute budget.
- **Trigger / основной поток:** решить hard constraints → lexicographic maximize target delivered value → minimize churn → prefer earlier completion → вернуть selected assignments/daily allocations, deferred/carryover и objective status.
- **Альтернативы:** ограниченный budget может дать FEASIBLE без доказательства optimality, либо UNKNOWN без usable solution; proven INFEASIBLE отдельно. Utilization выводится только как метрика.
- **Валидация:** objective order не меняется weighted sum; target value не включает позднюю поставку; no capacity/skill/WIP/deadline violations; данные stage bounds/termination не преувеличивают доказанность.
- **Persisted state:** validated solver result, objective vector, solve status/bounds when available, overflow metadata.
- **Permissions / ошибки:** service только свой run; CONTRACT_INVALID, OUTPUT_CONSTRAINT_VIOLATION, NO_SOLUTION_WITHIN_BUDGET.
- **Результат / аудит и telemetry:** честное feasible решение/класс отсутствия решения; solver version/termination audit, durations без customer work descriptions.

### OPT-003 — Actionable diagnostics

- **Actor / предусловия:** Manager; readiness failure, INFEASIBLE либо operational solver failure.
- **Trigger / основной поток:** открыть result → различить invalid input / infeasible / timeout / service error → показать проверяемые constraints и ссылки на correction screens → создать новый input revision/run после решения менеджера.
- **Альтернативы:** если минимальный conflict set не доказан, показать диагностированную причину и её пределы; необязательную работу можно явно defer; hard rules сами не ослабляются.
- **Валидация:** suggested correction не обещает feasibility без нового solve; не выдавать timeout за mathematically impossible; foreign work/absence reasons не раскрывать.
- **Persisted state:** diagnostic codes, safe entity references, analysis version, subsequent run lineage.
- **Permissions / ошибки:** read run + inputs allowed; DIAGNOSTIC_UNAVAILABLE с correlation ID, не raw stack.
- **Результат / аудит и telemetry:** менеджер знает следующее действие либо честно видит предел диагностики; resolution path и diagnostic usefulness metrics.

### SCEN-001 — Сравнение и review сценариев

- **Actor / предусловия:** Manager/reviewer; готовый Draft, baseline либо другой сопоставимый scenario.
- **Trigger / основной поток:** сравнить value/selection/deferred/carryover/assignee/dates/capacity/WIP/gates → просмотреть churn и bottlenecks → изменить scenario inputs → отдельный run → approve выбранный preview.
- **Альтернативы:** не сопоставимые horizon/value scale явно маркируются; новый snapshot делает Draft STALE; save/discard сохраняет baseline. Изменение таблицы не считается валидным solver schedule без новой проверки.
- **Валидация:** draft принадлежит Context; input lineage неизменна; preview содержит assumptions и отличает deadline/planned/forecast/actual; отсутствие решения не маскируется пустым успешным планом.
- **Persisted state:** scenario identity/revisions, run references, review acknowledgement, comparison metadata.
- **Permissions / ошибки:** review/modify scenario capabilities; STALE_DRAFT, INCOMPARABLE_SCENARIOS, INCOMPLETE_RESULT.
- **Результат / аудит и telemetry:** выбран проверенный кандидат либо требование изменения входов; scenario/review audit, review completion metrics.

### PUB-001 — Явная публикация версии

- **Actor / предусловия:** Manager с publish capability; READY Draft, пройден preview, согласованные inputs и baseline revision.
- **Trigger / основной поток:** Publish → повтор ACL/freshness/constraint/commitment checks → материализовать immutable complete version → атомарно продвинуть Context published pointer с ожидаемой revision → completion/notification.
- **Альтернативы:** duplicate publish возвращает version; concurrent publisher получает conflict; stale/new fact требует review/replan, не молчаливого overwrite. По D-03 конфликт committed capacity/work блокируется.
- **Валидация:** указатель не ссылается на неполный roadmap; ни optimize, ни profile approval не публикуют; Jira write-back отсутствует в MVP.
- **Persisted state:** immutable version/assignments, predecessor/baseline/input/run references, Context pointer revision, publish operation.
- **Permissions / ошибки:** publish Context, current grant; STALE_BASELINE, COMMITMENT_CONFLICT, PUBLISH_PENDING, FORBIDDEN.
- **Результат / аудит и telemetry:** одна authoritative версия Context; audit preview/publisher/lineage, resumable finalization, publish latency/failure.

### PUB-002 — Личный и stakeholder roadmap

- **Actor / предусловия:** Member, stakeholder, Manager; опубликована версия и есть соответствующий read grant.
- **Trigger / основной поток:** открыть свой/разрешённый план → assignments, даты, changes от предыдущей версии, deferred/carryover и допустимые assumptions → подтвердить просмотр/подать change request.
- **Альтернативы:** работы нет — явный empty state; revoked Jira permission скрывает детали и производные totals; draft не заменяет опубликованное представление.
- **Валидация:** actor/resource link серверный; отсутствие assignment не означает исключение из Team; Viewer не получает availability reasons или возможность изменения.
- **Persisted state:** version read marker при необходимости; underlying Published Roadmap неизменен.
- **Permissions / ошибки:** own plan / VIEW_PUBLISHED scope; PLAN_UNAVAILABLE, ACCESS_REVOKED.
- **Результат / аудит и telemetry:** понятные текущие обязательства и изменения; notification/read delivery state, агрегаты adoption без наблюдения за личной продуктивностью.

## Execution и change loop

### LOOP-001 — Execution Snapshot и обнаружение расхождений

- **Actor / предусловия:** Manager-triggered System в MVP D-01; published baseline, доступные Jira sources.
- **Trigger / основной поток:** Refresh execution → полный source snapshot → immutable facts/status/actual/remaining evidence → сравнить с baseline → inbox variances и freshness.
- **Альтернативы:** partial snapshot не доказывает удаление/завершение; scope changes и inaccessible work требуют отдельного review; автоматизация позже использует те же команды.
- **Валидация:** actual facts не переписываются плановыми датами; deadline lateness отдельно от baseline variance; baseline остаётся неизменной версией; assignee change не membership.
- **Persisted state:** execution snapshot, observation generation, detected changes/variances, completeness, baseline reference.
- **Permissions / ошибки:** refresh/read execution Context и Jira; PARTIAL_EXECUTION_SNAPSHOT, SOURCE_REVOKED.
- **Результат / аудит и telemetry:** факты и объяснимый impact inbox; snapshot lineage audit, freshness/change category metrics.

### LOOP-002 — Planning Change Request и review входов

- **Actor / предусловия:** Member own work/profile/availability, Manager либо System-detected change; known Context и base revisions.
- **Trigger / основной поток:** submit proposal/detected change → PENDING review → показать source evidence, impact и конфликты → approve/reject → новый approved planning input set для будущего run.
- **Альтернативы:** manager не «отклоняет реальность» DONE/actual start: подтверждённые факты сохраняются, он решает planning response/override. Withdraw, supersede и конфликтующий новый факт дают отдельные состояния.
- **Валидация:** approval не publish; pending proposal не действует; reason нужен для override/assumption; latest factual snapshot нельзя скрыто заменить устаревшим согласованным планом.
- **Persisted state:** request/detected change, decision, source/base revisions, approved input set, impact links.
- **Permissions / ошибки:** propose own / manage affected Context; STALE_CHANGE, FACT_CONFLICT, FORBIDDEN.
- **Результат / аудит и telemetry:** решение и уведомление автору, воспроизводимые новые inputs; protected decision audit и review queue latency.

### LOOP-003 — Replan с сохранением baseline lineage

- **Actor / предусловия:** Manager; complete execution snapshot, reviewed changes и published baseline.
- **Trigger / основной поток:** Re-optimize approved inputs → новый run с baseline → сохранить DONE/history/IN_PROGRESS remaining и handover → preview difference → Publish новой версии через PUB-001.
- **Альтернативы:** infeasible оставляет прежний published pointer; cancelled/stale draft не меняет обязательства; scope/member departure показывает impact и требует явного решения оставшейся работы.
- **Валидация:** прошлое не пересчитывается как свободная capacity; churn сравнивается с указанным baseline; каждая версия ссылается на predecessor, snapshot и inputs; поздний result не перескакивает новый baseline.
- **Persisted state:** run/scenario/version lineage, immutable history и publish operation.
- **Permissions / ошибки:** optimize/review/publish отдельно; STALE_BASELINE, UNREVIEWED_CHANGES, IN_PROGRESS_CONFLICT.
- **Результат / аудит и telemetry:** новый объяснимый baseline или actionable problem; replan/publish audit и closed-loop completion metric.

## Diagnostics и эксплуатационные качества

### OPS-001 — Поддержка без раскрытия customer content

- **Actor / предусловия:** пользователь с проблемой, authorized support/Admin с отдельной diagnostic capability.
- **Trigger / основной поток:** показать safe error code/correlation ID → inspect allowlisted technical events и tenant-authorized audit → определить failed stage/retry path.
- **Альтернативы:** более детальная customer evidence запрашивается отдельным согласованным процессом; отсутствие telemetry не изменяет бизнес-результат.
- **Валидация:** не логировать raw Jira/GraphQL bodies/messages, account IDs, токены, issue titles, absence reasons; diagnostic access не universal plan read; сроки хранения D-09/D-12.
- **Persisted state:** bounded diagnostics, access audit, retention configuration; бизнес-audit хранится отдельно.
- **Permissions / ошибки:** explicit diagnostics capability; DIAGNOSTIC_EXPIRED, DIAGNOSTIC_ACCESS_DENIED.
- **Результат / аудит и telemetry:** воспроизводимый безопасный support trail; access/retention audit, aggregate reliability metrics.

### OPS-002 — Восстановление, freshness и управляемые лимиты

- **Actor / предусловия:** System и authorized Manager/Admin; interrupted operations или рост объёма.
- **Trigger / основной поток:** обнаружить незавершённую operation → проверить lease/revision → идемпотентно продолжить или пометить failed → показать пользователю status/retry. Benchmark-defined limits проверяются до очереди.
- **Альтернативы:** partial persistence остаётся invisible/uncommitted; backend unavailable показывает freshness последнего разрешённого состояния; data repair сохраняет provenance.
- **Валидация:** нет предположения о multi-table transaction; stale worker не побеждает newer generation; лимиты никогда не превращают первые N issues в «полный scope»; time budget != infeasible.
- **Persisted state:** operation/attempt state, manifests, revision fences, retention и service policy.
- **Permissions / ошибки:** service recovery по tenant; LIMIT_EXCEEDED, RETRY_PENDING, RECOVERY_REQUIRED.
- **Результат / аудит и telemetry:** завершение либо понятный блокер без повреждения истории; recovery audit, queue/backlog/retry metrics. Численные SLA и инфраструктура ещё не утверждены.

## Трассировка и решения

Каждый ID проверяется в [acceptance criteria](ACCEPTANCE_CRITERIA.md) и распределяется по [delivery milestones](../implementation/DELIVERY_ROADMAP.md). Изменение требования сохраняет ID и фиксирует revision документа; удалённый ID не переиспользуется.

Разрешение D-решений выполняется в едином [реестре owner approval](PRODUCT_VISION.md#product-decisions-requiring-owner-approval), затем согласованно обновляет CJM, требования, acceptance и модель. Ни один draft здесь не разрешает менять scopes, deploy или реализовывать authorization до отдельного implementation milestone.
