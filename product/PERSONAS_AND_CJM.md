# Personas and Customer Journey Maps

Статус: DESIGN DRAFT, 12 сентября 2026. Общие принципы и открытые D-ID: [Product Vision](PRODUCT_VISION.md). Формальные use cases: [Requirements](PRODUCT_REQUIREMENTS.md).

## Проверка списка персон

Пять человеческих перспектив и один системный актор. Один человек может совмещать несколько персон, но права складываются только из явных scoped grants. Jira Project Admin — временный credential/handoff actor, а не ещё одна RM business role. Portfolio stakeholder и plan viewer объединены: в MVP они читают согласованный результат и обсуждают его; право менять бизнес-входы требует отдельного manager grant. Resource — объект capacity, не гарантия интерактивной учётной записи или доступа.

### P-ADMIN — installation / RM administrator

- **Цель:** легитимное владение приложением, непрерывность управления, отзыв доступа и поддержка.
- **Вход:** установка/страница governance; support reference; запрос передачи ownership. Конкретный Forge module не выбран.
- **Права:** RM_ADMIN installation governance; не автоматический доступ к содержимому всех проектов/планов (D-04/D-06).
- **Первый опыт:** проверить installation credential, явно принять RM_ADMIN, назначить первого manager.
- **Регулярно:** manager lifecycle, audit, operational health, review access requests.
- **Исключения:** некому передать ownership; credential недоступен; удалён последний admin; пользователь потерял Jira access; uninstall.
- **Решения:** кому дать/отозвать управление, как разрешить legacy ownership, какие support данные разрешить раскрыть.
- **Смотрит / меняет:** ACL/audit/health / grants, governance settings, согласованные lifecycle операции; не исправляет бизнес-Value под видом поддержки.
- **Уведомления:** unresolved ownership, failed infrastructure, revocation, recovery; передача менеджеру сопровождается audit и уведомлением о новых полномочиях.
- **Передача:** manager получает управляемую Team/Context; support — очищенную диагностику.
- **Успех:** нет осиротевших владений, ни одного несанкционированного повышения прав; решения прослеживаются.

### P-MANAGER — planning manager

- **Цель:** объяснимо согласовать лучший выполнимый portfolio и своевременно пересматривать его.
- **Вход:** Jira action как entry/discovery; доступный Context; inbox/deep link к run или draft.
- **Права:** RM_MANAGER только своих Team/Context; членство/ресурсы Team и planning operations Context требуют соответствующих scopes ACL.
- **Первый опыт:** запросить полномочия, определить объект планирования, подготовить людей и данные, опубликовать первый baseline.
- **Регулярно:** review enrollment/availability/change requests, readiness, сценарии, publication и replan.
- **Исключения:** identity sync unavailable, неизвестная legacy Team, недоступные issues, infeasible обязательства, устаревший draft.
- **Решения:** состав, planning eligibility, allocation, scope, mapping, horizon, обязательность/пины, допущения, выбор сценария и publish.
- **Смотрит / меняет:** разрешённые people/work/hierarchy/capacity/diagnostics / свои Team profiles и Context planning inputs. Не меняет actual history.
- **Уведомления:** pending requests, data drift, failed/infeasible run, готовый draft; одно событие — одна ссылка/предмет решения.
- **Передача:** admin → grant; member → утверждение proposal; stakeholder → согласование trade-off; member/viewer ← опубликованные изменения.
- **Успех:** план выполним, scope понятен, Deferred/Carryover объяснены, публикация осознанна.

### P-MEMBER — сотрудник / планируемый ресурс

- **Цель:** попасть в правильную Team, видеть реалистичные назначения, влиять на точность профиля/доступности.
- **Вход:** приложение, eligible Team, ссылка на собственный plan/request.
- **Права:** до approval — только безопасный discovery и собственный enrollment; RM_MEMBER после approval — свой профиль, предложения/доступность, доступный published plan. Нет setup/optimization/publish.
- **Первый опыт:** запрос → ожидание → решение manager → профиль/календарь → первый личный план.
- **Регулярно:** работать в Jira, смотреть изменения baseline, предлагать коррекции и следить за review.
- **Исключения:** pending/rejected/excluded; отпуск конфликтует с планом; assignment изменён; доступ отозван.
- **Решения:** куда запросить вступление, какие навыки/даты подтвердить, какой remaining effort предложить; не принимает решение об общей обязательности.
- **Смотрит / меняет:** свои requests, accepted profile, план и разрешённые Jira issues / proposal draft, request/withdrawal; accepted inputs меняются только через уполномоченное review.
- **Уведомления:** принято/отклонено с причиной, публикация собственного изменения, вопрос manager. Причина отсутствия не раскрывается всей Team.
- **Передача:** member → manager pending; manager → member decision/changed plan; Jira → system факты исполнения.
- **Успех:** понятен статус каждого запроса и действующий план; ожидание review не выглядит как потеря данных.

### P-VIEWER — business stakeholder / plan viewer

- **Цель:** понять достижимые результаты, обязательства и trade-offs без настройки модели.
- **Вход:** разрешённый Context/published roadmap, in-app уведомление о новой версии.
- **Права:** scoped VIEW_PUBLISHED по D-06; не RM_MEMBER автоматически, не ресурс. Доступ к source details ограничен Jira permissions.
- **Первый опыт:** получить read grant, выбрать понятный Context, увидеть baseline, assumptions и horizon.
- **Регулярно:** сравнивать новые публикации, сроки и actual/forecast, задавать вопросы manager.
- **Исключения:** нет опубликованного плана, часть данных restricted, версия superseded, результат не успевает к бизнес-дате.
- **Решения:** подтвердить приоритеты организационно или попросить manager пересмотреть; MVP не даёт скрытого veto/publish permission.
- **Смотрит / меняет:** разрешённые summaries/deliverables/deferred reasons / только свои view filters; бизнес-изменение оформляет manager.
- **Уведомления/передача:** manager публикует → viewer читает; viewer запрашивает изменение → manager формализует change request. Не предполагается email/Slack интеграция.
- **Успех:** published, forecast и deadline различимы, нет трактовки draft как обещания.

### P-PROJECT-ADMIN — Jira project admin как кандидат bootstrap

- **Цель:** подключить проект и найти легитимного RM-владельца; не захватить внешнюю Team.
- **Вход:** Jira action/permission request при первом открытии.
- **Права:** Jira permission может быть доказательством для bootstrap request; до RM grant права обычного пользователя.
- **Первый опыт:** посмотреть ownership status без утечки содержимого, предъявить проверяемый credential, запросить назначение manager.
- **Регулярно:** восстановить Browse/field permissions и передавать вопросы governance RM_ADMIN.
- **Исключения:** существующий RM owner, одновременные claims, project admin другого проекта видит ту же Atlassian Team.
- **Решения:** подключать ли источник, кого номинировать, какие Jira permissions исправить.
- **Смотрит / меняет:** состояние собственной заявки/доступа / заявку; настройки Jira за пределами RM меняет отдельно.
- **Уведомления/передача:** запрос → RM_ADMIN; назначение/отказ → заявитель; существующий manager получает запрос, а не заменяется.
- **Успех:** project credential не превращается в автоматический организационный ownership.

## Матрица ожидаемых разрешений

Все grants tenant-scoped; Team/Context scope обязателен. RM_ADMIN не означает Jira admin. Предложения D-04/D-06 требуют утверждения.

| Операция | RM_ADMIN | RM_MANAGER | RM_MEMBER | Viewer capability | Project Admin без RM grant |
|---|---|---|---|---|---|
| Управлять RM ACL / назначать managers | Да, с audit | Нет эскалации; запрос передачи | Нет | Нет | Только bootstrap request |
| Создать Context / связать Team | Через отдельное business grant | Да, в разрешённой области | Нет | Нет | Нет |
| Смотреть Team roster | Только при content grant | Своя Team | По разрешённой visibility policy | Нет по умолчанию | Нет |
| Add/remove/exclude / review joins | Только при manager grant | Своя Team | Только собственный request | Нет | Нет |
| Roles/skills/capacity: прямое изменение | Только при manager grant | Своя Team/Context | Собственные proposals | Нет | Нет |
| Scope, mapping, constraints | Только при manager grant | Свой Context | Только view/proposal | Только опубликованное | Нет |
| Optimize / сравнить drafts / publish | Только при manager grant | Свой Context | Нет | Нет | Нет |
| Published / My work | По явному read grant | Свой Context | Доступный Context / своё | Доступный Context | Нет по credential |
| Support diagnostics | Operational, очищенные | Свои correlation references | Свои ошибки | Свои ошибки | Своя заявка |

## CJM менеджера — от первого входа до повторной публикации

Ожидаемый переход ощущений: «непонятно, что планируется» → «границы подтверждены» → «данным можно доверять» → «компромисс объясним» → «изменения управляемы». Таблица показывает именно переходы опыта, а не набор кнопок. Каждый выход передаёт данные следующему этапу.

| Этап / ожидание | Действие и touchpoint | Системный переход / данные | Решение пользователя | Сбой и восстановление / handoff | Требования |
|---|---|---|---|---|---|
| M01 Первый контакт: понять назначение | Jira action → welcome с entry project/board | Видит product purpose и ownership status; user OBSERVED без прав | Продолжить/запросить доступ | Installation не готова → статус и RM admin request | AUTH-001, AUTH-002 |
| M02 Получить законные полномочия | Bootstrap/access request | PENDING → grant или REJECTED; membership не создаётся | Принять scoped responsibility | Team уже управляется → обратиться к owner, не claim | AUTH-001, AUTH-003 |
| M03 Назвать объект планирования | Context chooser/create | Context SETUP, стабильный ID, собственное имя | Какой результат/период планируем | Имя проекта не становится Team; отмена без побочных изменений | CTX-001 |
| M04 Подключить команду | Team connection preview | ContextTeam link; Atlassian identity отдельно от membership | Существующая Team или новая явная manual Team | Legacy ambiguity → admin decision; несколько кандидатов → выбор | TEAM-001, CTX-002 |
| M05 Проверить реальный состав | Roster + user search | Manual additions; provenance, exclusions, dates | Кто входит; кого исключить | Graph identity unavailable → manual путь, identity сохранена | TEAM-002, TEAM-003 |
| M06 Рассмотреть заявки | Membership inbox | PENDING → APPROVING → APPROVED либо REJECTED; completion выдаёт RM_MEMBER | Подтвердить конкретного человека или отказать с причиной | Exclusion/conflicting review → явное решение; notify member | TEAM-004 |
| M07 Уточнить eligibility | Profiles, role/skill catalog | Accepted profiles с revision | Primary/additional roles и skills | Неизвестный role ID → repair catalog, без догадки по Jira Role | PROF-001, PROF-002 |
| M08 Проверить календарь | Capacity grid, availability inbox | Working week, allocation, approved exceptions | Реальная доступность, approval отпуска | Перегрузка/пересечение → capacity conflict; member уточняет | CAP-001, CAP-002 |
| M09 Зафиксировать scope | Sources + scope preview | Versioned project set, Team-field/board predicate, counts | Подтвердить множество работ | Недоступный проект/board → PARTIAL, не «0 issues» | SCOPE-001, SCOPE-002 |
| M10 Увидеть исходную работу | Scope explorer с provenance | Полные страницы, ancestors для навигации отдельно | Проверить, те ли работы и уровни | Только Epic/Story → inspect scope/kind/coverage, не менять Team по имени | ING-001, WORK-001 |
| M11 Согласовать смысл полей | Mapping editor + preview samples | DRAFT → VALIDATED → ACTIVE | Effort conversion, Value, deadline, roles/skills | Missing field/type → NEEDS_REVIEW; safe viewing доступно | MAP-001, MAP-002 |
| M12 Проверить структуру | Hierarchy/dependency inspector | Explicit planning kind; group ownership; Blocks graph | Что executable, какой container владеет Value | Overlap/cycle/missing descendant → readiness item | WORK-002, DEP-001 |
| M13 Разрешить внешние gates | Prerequisite detail | DONE evidence либо approved assumption/date | Принять допущение с основанием | Нет даты/недоступен источник → blocking issue, no auto due-date assumption | DEP-002 |
| M14 Исправить readiness | Checklist с переходом к полю/человеку | Пообъектные NEEDS_INPUT → READY | Уточнить/явно исключить optional incomplete work | Не исключать IN_PROGRESS автоматически; request владельцу данных | READY-001 |
| M15 Выбрать горизонт | Horizon editor | Optimization date, targetEnd, scheduleEnd, timezone version | Реалистичный target и конечный overflow | Невалидные даты → inline error; не растягивать молча | PLAN-001 |
| M16 Утвердить hard policies | Policy review | WIP simple/advanced, pins, must-have, exclusions, handover | Принять ограничения и зафиксировать входы | Contradiction → INPUT_CONFLICT; нет hidden precedence | PLAN-002, PLAN-003 |
| M17 Запустить и продолжить работу | Optimize → run status | ApprovedInputSet + run QUEUED/RUNNING | Запустить/отменить ожидание | Повтор click → тот же run; transport failure retry; revoked rights stop action | OPT-001, OPT-002 |
| M18 Разобрать невозможность | Diagnostic panel linked to assumptions | INFEASIBLE отделён от ERROR/TIME_LIMIT | Изменить конкретное одобренное ограничение, не «починить solver» | No minimal core не выдаётся за доказательство; новый scenario/run | OPT-003 |
| M19 Сравнить варианты | Scenario comparison | Assignments, dates, deferred, carryover, handover, value/churn | Выбрать компромисс; pins → новый run | Разные входы показываются; stale baseline → rebase/replan | SCEN-001 |
| M20 Опубликовать обязательство | Preview → explicit Publish | Immutable version + Context published pointer + lineage | Подтвердить видимые изменения | CAS conflict/partial write → reconcile; old publication remains authoritative | PUB-001 |
| M21 Передать план людям | Published/My work | Member/viewer читают опубликованную версию | Объяснить изменения и assumptions | Нет Jira access → restricted detail, без утечки | PUB-002 |
| M22 Следить за реальностью | Refresh execution → inbox | Immutable snapshot, detected changes, stale marker | Какие proposals принять; actuals не «отклоняются» | Incomplete sync → last-good snapshot, freshness warning | LOOP-001, LOOP-002 |
| M23 Перепланировать будущее | Review → re-optimize → compare/publish | Новый approved set и run, parent baseline сохранён | Принять новый план или оставить старый со stale warning | Capacity/deadline conflict → M18; member/viewer получают changed-plan notification | LOOP-003 |

## CJM сотрудника

| Этап / ожидание | Touchpoint и действие | Состояние / решение | Исключение и следующий шаг | Передача / требования |
|---|---|---|---|---|
| E01 Найти себя | Welcome, eligible Team discovery | OBSERVED user видит только metadata по directory policy D-13 | Team не eligible → request access, не поиск закрытых roster | AUTH-004, TEAM-004 |
| E02 Попросить вступление | Join form с Team identity | Один PENDING request; отправить/отменить | Повтор отправки возвращает тот же pending; excluded требует manager review | Manager inbox, TEAM-004 |
| E03 Понять ожидание | My requests | Pending не означает членство/доступ; cancel возможно до review | Reject содержит безопасную причину; повтор по правилам eligibility | Decision notification, TEAM-004 |
| E04 Принять результат | Approval detail → My profile | После завершённого approval membership + RM_MEMBER активны | Частично выполненное approval остаётся APPROVING, не полуправа | AUTH-002, TEAM-004 |
| E05 Уточнить профессию | Accepted profile и отдельный proposal editor | Видны принятые и предлагаемые role/skill значения | Нельзя править чужой профиль или свои accepted поля напрямую | Manager review, PROF-002 |
| E06 Сообщить отсутствие | My availability calendar | DRAFT → PENDING_REVIEW; proposed capacity preview | Overlap/invalid dates → исправить; pending не уменьшает solver capacity | CAP-002 |
| E07 Посмотреть личный план | My work, published version label | Назначение, даты, причины изменения, source link | Пока нет публикации — empty state, не выдавать draft за план | PUB-002 |
| E08 Выполнять в Jira | Jira issue → RM snapshot refresh | Actual facts фиксируются, статус плана может стать stale | Нет доступа к issue → restricted; RM не подменяет Jira статус | LOOP-001 |
| E09 Предложить изменение | Remaining effort/assignment proposal | PENDING_REVIEW с причиной и expected revision | Изменившийся объект → CONFLICT; заново просмотреть before/after | LOOP-002 |
| E10 Получить новый baseline | Notification → changed assignments | Сравнение собственной старой/новой работы | Rejected proposal остаётся в истории; старый план не исчезает без публикации | Manager → member, LOOP-003 |
| E11 Покинуть/потерять доступ | Membership/access status | Effective membership прекращается по явному решению; history retained | Назначенная будущая работа создаёт planning impact, не удаляется | TEAM-003, AUTH-003 |

## CJM администратора

| Этап | Действие / ожидание | Переход и данные | Failure/recovery | Handoff / требования |
|---|---|---|---|---|
| A01 Installation | Проверить app/site и запрошенные permissions | PROVISIONING → ACTIVE либо требуется действие | Не запускать setup поверх неполной схемы; показывать correlation | AUTH-001 |
| A02 Authority | Предъявить разрешённый credential, принять ответственность | Bootstrap REQUESTED → VERIFIED → COMMITTED | Прерванная запись/два claims → один winner, второй review | AUTH-001, D-04 |
| A03 Назначение manager | Проверить identity и Team/Context scope | Grant PENDING_FINALIZATION → ACTIVE, audit reason | Нельзя grant по display name; non-existing user → resolve | Manager M02, AUTH-003 |
| A04 Governance | Смотреть owners и lifecycle | Review grants, orphan risks, retention | Нельзя отозвать последнего owner без replacement/recovery | AUTH-003 |
| A05 Поддержка | Найти correlation, увидеть safe health | Diagnostic case без raw Graph messages/tokens | Jira access/scopes различаются; не предлагать unsupported identity workaround | OPS-001 |
| A06 Смена manager | Назначить successor, затем revoke predecessor | Переход владения не меняет membership/capacity | Concurrent revoke / pending publish → recheck ACL | Manager handoff, AUTH-003 |
| A07 Security | Отозвать grant, обработать deactivation | Немедленный отказ новых операций; in-flight outputs quarantined as needed | Кэш permissions не даёт продолжить publish | AUTH-003, OPT-001 |
| A08 Uninstall/recovery | Явное lifecycle решение | Suspended/removed; политика retention D-09 | Reinstall не приписывает старые данные новой installation автоматически | OPS-002 |

## CJM stakeholder / viewer

| Этап | Действие и ожидание | Состояние/решение | Исключение/передача | Требования |
|---|---|---|---|---|
| V01 Доступ | Получить scoped read grant | Viewer не становится ресурсом | Нет grant → запрос manager/admin | AUTH-002 |
| V02 Выбор | Открыть Context с понятными Team/source/horizon | Подтвердить, что это нужный объект | Одинаковые имена → показать тип и безопасную identity metadata | CTX-001 |
| V03 Первый план | Читать опубликованные deliverables и assumptions | Отделить planned/deadline/forecast | NO_PUBLISHED_PLAN — ждать публикации; drafts недоступны | PUB-002 |
| V04 Обсуждение | Найти deferred/carryover и причину | Попросить пересмотр приоритета у manager | Нельзя снять hard deadline ради красивой картинки | SCEN-001, LOOP-002 |
| V05 Новая версия | Открыть notification и diff | Понять, какие обязательства изменились | Restricted issues не раскрываются через summaries/exports | PUB-002 |
| V06 Регулярный обзор | Сравнивать baseline с actual/forecast | Обсудить необходимость replan | Несвежие данные явно помечены | LOOP-001, LOOP-003 |

## CJM Jira Project Admin

| Этап | Действие | Состояние/решение | Исключение и handoff | Требования |
|---|---|---|---|---|
| J01 Entry | Открыть RM из своего проекта | Получить только разрешённую стартовую информацию | Видеть Team != владеть Team | AUTH-001 |
| J02 Проверка | Отправить bootstrap nomination с server-verified credential | PENDING; не RM_MANAGER | Existing RM ownership → approval у RM_ADMIN, без takeover | AUTH-001, AUTH-003 |
| J03 Решение | Получить grant либо отказ | При grant начинается manager journey | При отказе Jira admin права сохраняются, RM права не появляются | M02, AUTH-003 |
| J04 Подключение source | Исправить Browse/field visibility в Jira | Повторный read/preview в RM | Не делать scopes upgrade наугад | SCOPE-001, MAP-001 |
| J05 Передача | Завершить настройку проекта | RM ACL дальше authoritative | Потеря Jira admin не должна автоматически отбирать отдельный RM grant; source access всё равно проверяется | AUTH-004 |

## S-SYSTEM — sync / worker / notification actor

Не человеческая persona и не RM_ADMIN. Работает от конкретной служебной capability с tenant, Context и operation bounds.

| Фаза | Вход → действие → выход | Recovery / ограничения |
|---|---|---|
| S01 Refresh | Authorized refresh/event → source read → import batch | Pagination/rate limit retry с cursor; incomplete не удаляет last-good |
| S02 Detect | Snapshot delta → change detection → deduplicated inbox items | Observed actuals не ждут approval как будто их можно отменить |
| S03 Compile | Approved revision → validation → immutable input set | Pending proposals исключены; stale/permission loss блокируют run |
| S04 Solve | Run envelope → worker → validated result | At-least-once delivery, single materialized result, late/cancelled outputs не публикуются |
| S05 Publish recovery | Authorized publish command → pointer operation → reconciliation | Служба не инициирует publish без явной команды manager |
| S06 Notify | Durable event → in-app inbox item → read status | Dedupe по event/recipient; нет массового повтора unchanged state; внешние каналы D-01/D-12 |

## Общие переходы и экраны

- Welcome / access request → governance inbox → Context chooser.
- Context workspace содержит единый header: Context, Team, sources, horizon, revision/freshness; roster и work panels используют один contextId.
- Раздельные inbox tabs: membership, profile/availability, planning-impact changes; review actions не взаимозаменяемы.
- Readiness links ведут к конкретному полю, policy, member или gate и сохраняют контекст.
- Run page показывает QUEUED/RUNNING/результат; draft review показывает различия inputs и baseline, не только календарь.
- Опубликованный roadmap и My work содержат version/lineage; draft не подменяет их.
- Пусто, недоступно, неполно, не настроено, вычисляется, infeasible и infrastructure failure — разные состояния интерфейса.

Самый важный handoff: manager утверждает не только состав или входы, но и новую версию обязательств. Ни successful sync, ни accepted proposal, ни successful solve не заменяют Publish.
