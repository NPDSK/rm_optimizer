# RM Optimizer — Product Vision

Статус: DESIGN DRAFT, 12 сентября 2026. Основание аудита: `980eb79`.
Документ описывает целевой продукт, не факт доступности функций в установленном приложении.
Обязательные принципы из запроса считаются заданными. Рекомендации с D-ID требуют утверждения владельца; они не дают разрешения на реализацию или deploy.

## Навигация и приоритет

1. [Personas and CJM](PERSONAS_AND_CJM.md) — опыт людей и передачи ответственности.
2. [Product requirements](PRODUCT_REQUIREMENTS.md) — контракты возможностей.
3. [Acceptance criteria](ACCEPTANCE_CRITERIA.md) — наблюдаемые результаты.
4. [Target domain](../architecture/DOMAIN_MODEL_TARGET.md) — значения сущностей и инварианты.
5. [Target architecture](../architecture/TARGET_ARCHITECTURE.md) — границы исполнения.
6. [Current-to-target gap](../architecture/CURRENT_TO_TARGET_GAP.md) — доказательства из репозитория.
7. [Delivery roadmap](../implementation/DELIVERY_ROADMAP.md) — крупные вертикальные этапы.

Этот комплект задаёт проект целевой модели. Старые `docs/domain-model.md`, `application-architecture.md`, `field-mapping.md` и SQL-дизайн остаются описанием предшествующих решений, а не основанием сохранять неверную абстракцию. Изменения публичных контрактов требуют отдельного версионирования; этот комплект не меняет действующие JSON schemas.

## Проблема и клиент

Менеджеру нужно договориться, какой набор работ реально завершить в квартале, кто сможет его выполнить и что придётся отложить. Список Jira issues и сумма estimates не отвечают на этот вопрос: роли, доступность, зависимости, уже начатая работа и обязательства конкурируют за одни и те же даты.

Целевой клиент — организация с Jira Cloud, несколькими источниками работ и командами, где менеджер отвечает за план поставки, сотрудники уточняют профиль/доступность, а бизнесу нужен объяснимый опубликованный baseline. MVP обслуживает одну установку и один Planning Team на Planning Context. Несколько Jira-проектов могут поставлять работу в один контекст; это отдельная задача от объединения нескольких команд в одном solver run.

## Jobs to be done

- Когда формируется период, менеджер собирает допустимый набор работ и ресурсные ограничения, чтобы выбрать максимальную достижимую бизнес-ценность, а не только заполнить календарь.
- Когда оптимального для всех желаний плана нет, менеджер видит конфликт и цену возможного изменения, чтобы осознанно согласовать обязательства.
- Когда план опубликован, сотрудник понимает своё назначение и изменение дат, может сообщить новые факты и предложить коррекцию.
- Когда реальность расходится с baseline, команда сохраняет факты исполнения и перепланирует будущее без переписывания истории.
- Когда меняется менеджер или права, администратор сохраняет управляемость и изоляцию данных без автоматического захвата Team.
- Когда бизнес смотрит план, он отличает обещание, ограничение и прогноз, видит Deferred/Carryover и допущения.

## Предложение ценности

RM Optimizer превращает явно определённый объект планирования в проверяемый план: люди + scope + одобренные данные + правила + горизонт → feasible portfolio → рассмотренный и опубликованный roadmap → контролируемое перепланирование.

Planning Context отвечает «что и в каких границах планируем». Planning Team отвечает «какая организационная команда участвует». Atlassian Team — внешний идентификатор/возможный источник членства. Jira Project/Board — источник/entry point. Ни один из них не подменяет остальные.

## Предлагаемый целостный MVP

Предложение границ MVP требует D-01. Это завершённый путь, а не текущее состояние кода:

- безопасный bootstrap, RM ACL, явный Context с одной Team;
- manual manager additions и self-enrollment с обязательным review;
- одобренные роли, навыки, календарь, отпуск и выделенная capacity;
- явный project set и ATLASSIAN_TEAM_FIELD либо BOARD_SCOPE, все страницы доступных issues;
- mappings с PERSON_DAYS, readiness, hierarchy отдельно от Blocks, containers и внешние gates;
- Python CP-SAT через асинхронную service boundary, hard constraints и три lexicographic objectives;
- сравнение сценариев, объяснение infeasibility, preview, публикация внутри RM;
- опубликованный личный план, обнаружение изменений по инициированному refresh и manager review/replan;
- аудит значимых решений, ограниченные безопасные diagnostics.

MVP не считается завершённым на кнопке Optimize. Внешние Jira write-back, email/Slack delivery, автоматический фоновой мониторинг и hosting technology не подразумеваются этим определением.

## Не-MVP

Multi-Team portfolio solve; несколько независимых опубликованных планов, конкурирующих за одну capacity без резервирования; CUSTOM_JQL; HR/calendar интеграции; ML estimates; employee productivity multipliers; несколько исполнителей одной executable задачи; внутридневные зависимости; произвольные типы precedence; сложный drag-and-drop; глобальная кросс-tenant аналитика. Productivity не является согласованным «следующим шагом»: текущий принцип требует одинаковый effort для всех eligible исполнителей.

## Принципы

1. Membership, authorization, profession и skills — четыре разных оси.
2. Идентичность стабильна при смене проекта, board, display name и горизонта.
3. Context switch — чтение другого контекста, не переименование/мутация предыдущего.
4. Membership не выводится из assignee; scope не выводится из membership.
5. Только явно подтверждённый scope определяет включённые данные. UI-фильтр не изменяет solver input.
6. Неполные/недоступные данные маркируются; пустой результат не равен успешному полному sync.
7. Deadline — hard business constraint, planned end — baseline, forecast — текущая оценка, actual — факт.
8. PERSON_DAYS и DAY resolution; доступность влияет на capacity, не на effort.
9. Parent-child не означает Finish-to-Start. Epic не является container по названию.
10. Solver никогда не исправляет противоречия скрытым снятием обязательств. Utilization — отчёт, не главная цель.
11. Черновик не виден как обязательство; публикация явная и версионированная.
12. Jira permissions и RM ACL применяются совместно; discovery не выдаёт права.
13. Неподдерживаемая nested identity API не обходится токеном, Basic Auth, Teams REST или добавлением запрещённого scope.

## Метрики успеха

Численные launch thresholds утверждаются в D-12; ниже определения, а не придуманные SLA.

| Метрика | Измерение и интерпретация |
|---|---|
| Time to first trusted plan | Медиана и p90 от разрешённого первого входа менеджера до первой публикации; отдельно ожидание ACL/member review и работа пользователя |
| Setup completion | Доля начавших Context setup, дошедших до READY; причины остановки по доменам без содержимого customer issues |
| Diagnostic usefulness | Доля INFEASIBLE/NEEDS_INPUT, после которых менеджер смог исправить конкретные входы и получить feasible run |
| Replan adoption | Доля активных контекстов, завершивших хотя бы один цикл snapshot → review → replan → publish |
| Plan trust | Ошибки контекста/прав/двойного счёта Value, несанкционированные публикации и silent data loss — release blockers, а не допустимая оптимизация метрики |
| Delivered value | Plan vs actual в одной шкале Context; не складывать несопоставимые шкалы разных бизнес-контекстов |
| Stability | Изменение selection/assignee/dates относительно baseline, отдельно от физического handover effort |
| Reliability | Время queue/run, доли completion/retry, FEASIBLE vs доказанный OPTIMAL; timeout не объявлять INFEASIBLE |
| Capacity visibility | Обнаруженные перегрузки и их разрешение; не KPI продуктивности сотрудников |

## Терминология

| Термин | Смысл |
|---|---|
| Installation | Tenant приложения с собственными идентичностями, ACL, данными и сроками хранения |
| Planning Team | Стабильная организационная команда; внешняя Atlassian identity опциональна |
| Planning Context | Самостоятельный объект планирования с Team links, sources, scope, horizon/policies, scenarios и своим published pointer |
| Work Source / Work Scope | Где читаем / какое множество работ из источников планируем |
| Resource | Планируемый человек; наличие учётной записи не создаёт membership или grant |
| Approved input set | Неизменяемый снимок согласованных входов, по которому выполнен run |
| Deferred / Carryover | Не выбранная работа / выбранная работа, завершённая за target horizon, но в schedule horizon |
| Draft / Published baseline | Кандидат решения / явно опубликованная версия; оба не переписывают факты Jira |

## PRODUCT DECISIONS REQUIRING OWNER APPROVAL

Все D-ID ниже OPEN. Использование рекомендации в остальных документах обозначает проектное предложение, а не молчаливое утверждение. Принципы выше не являются предметом выбора.

| ID / вопрос | Почему важно | Option A | Option B | Рекомендация и последствия |
|---|---|---|---|---|
| D-01. Где граница MVP closed loop? | Определяет реальную завершённость продукта | Refresh пользователем, inbox, ручной replan и RM publish | Плюс автоматические background triggers и внешние уведомления | A: полный управляемый цикл раньше; автоматизация отдельным этапом, пользователь видит freshness |
| D-02. Как задавать cross-project scope? | Иначе «все работы Cools» неоднозначно и меняется с вызывающим проектом | Явный project set Context; board results пересекаются с ним | Все доступные проекты динамически | A: воспроизводимость и понятный доступ; новые проекты добавляются явно, preview показывает исключённые источники |
| D-03. Как избежать двойного резервирования людей/работ? | Несколько Context могут независимо обещать одну capacity | В MVP один committed Context на Team/пересекающийся горизонт; общие person/work identities также защищены от второго commitment; альтернативы — scenarios | Сразу fine-grained multi-context capacity reservations | A: проще безопасная публикация; overlapping commitments блокируются, в том числе для одного человека в разных Teams. Это ограничивает concurrent planning, но сохраняет multi-Team модель связей |
| D-04. Кто инициирует первый RM_ADMIN/Manager bootstrap? | Project admin не доказывает владение организационной Team | Проверенный installation/site authority назначает RM_ADMIN; он подтверждает managers | Project admin инициирует nomination неуправляемой Team, authority явно проверяет и подтверждает её | A: единый governance entry; B требует дополнительного request/review пути. Ни один вариант не допускает автоматическую роль или takeover. Проверка credential требует исследования |
| D-05. Как учитывать nested container Value? | Сложение Epic и Story может удвоить ценность | Явно выбрать один value-owner для набора descendants, overlaps блокировать | Автоматически выбрать deepest/outermost container | A: объяснимость; понадобится разрешить конфликт ownership в readiness |
| D-06. Какой доступ у stakeholder? | Доступ к плану не должен делать человека ресурсом | Отдельный scoped VIEW_PUBLISHED capability, без нового admin/member статуса | Назначать RM_MEMBER всем viewers | A: оси не смешиваются; MVP требует read grant и проверки Jira visibility |
| D-07. Что даёт platform membership при восстановлении API? | Членство и доступ различаются | Сохранять platform membership evidence, RM grants — явным правилом/одобрением | Автоматически выдавать базовый RM_MEMBER каждому platform member | A: текущая ручная схема безопасна; approved join уже явно создаёт RM_MEMBER, platform sync не выдаёт Manager |
| D-08. Где живут роли/skills? | Разные проекты называют профессию по-разному | Installation catalogs + Team resource profiles; Context выбирает требования | Независимые catalogs для каждого Context | A: стабильные ID и переиспользование; миграция не объединяет одинаковые имена автоматически |
| D-09. Сроки хранения и видимость отсутствий? | История нужна для объяснения планов, причины отсутствия чувствительны | Хранить минимум: доля/даты; причины ограничены владельцем/уполномоченным manager | Широкая Team-видимость причин и длительная история | A: меньше раскрытия; конкретные сроки для audit, snapshots и diagnostics нужно утвердить отдельно |
| D-10. Как вводить допущение о дате внешнего gate? | Deadline не доказывает будущую готовность prerequisite | Manager явно вводит assumption/date/reason; DONE используется как evidence | Предлагать Due Date как незадействованную подсказку и отдельно запрашивать explicit approve | A: меньше риска принять подсказку за факт; B ускоряет review, но требует ясного pending состояния. Оба варианта запрещают automatic PLATFORM_DEADLINE fallback, его удаление не является открытым выбором |
| D-11. Как решать неоднозначные legacy данные Cools/Legal? | Источник мог меняться без истории владения ручными данными | Сохранить ID/снимок и запросить явное распределение в A/B | Сохранить legacy state архивным Context и явно создать чистые A/B без переноса неоднозначных настроек | A: меньше повторной настройки, но нужен binding review. B проще отделяет новую работу, сохраняя старую историю отдельно. Ни один вариант не угадывает identity по имени/последнему source |
| D-12. Какие лимиты, timezone и уведомления утверждаем? | Влияет на ожидания и корректность дат | Явная timezone Context, in-app notifications, лимиты после benchmarks/pilot | Единая installation timezone, внешние уведомления и заранее утверждённые SLA | A: одинаковые даты внутри Context и гибкость разных рабочих календарей; значения лимитов и delivery latency утверждаются после измерений |
| D-13. Кто видит Team в directory и может запросить join? | Jira discovery не означает согласие раскрыть RM roster или принимать заявки | Team owner явно разрешает discoverability/request policy для eligible installation users; минимальные metadata | Любой installation user может запросить любую обнаруживаемую через доступный Jira контекст Team | A: контролируемый onboarding; нужен directory/request policy screen и owner decision. Оба варианта сохраняют PENDING/review, не показывают roster/work и не выдают права автоматически |

Внешний worker hosting, service-to-service authentication, queue/callback и доказательство installation authority — технические решения с обязательной проверкой возможностей, см. TARGET_ARCHITECTURE. Этот документ не утверждает конкретный Forge runtime для Python.
