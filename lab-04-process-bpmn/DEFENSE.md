# DEFENSE — Завдання 4: Процес / поведінка (BPMN 2.0)

**Намір**: Описати наскрізний бізнес-процес Billiard CRM для прецеденту **`UC-04` (Зареєструватися в індивідуальному турнірі)** з обов'язковим включенням (`«include»`) **`UC-06` (Оплатити турнірний внесок)** однією точною декларативною формою — у нотації **BPMN 2.0 XML** (критерії готовності та прийняття: [spec.md](./spec.md)).

## Взаємодія з AI

1. **Стадія планування**

- [00-planning-stage.md](./ai/00-planning-stage.md) — початковий запит із чернеткою `spec.md`, декомпозиція 4 етапів процесу між пулами (`Користувач`, `Billiard CRM`, `Payment Gateway`), опрацювання граничних випадків (поновлення простроченої броні `PENDING`, конкурентний таймер `min(15 хв, deadline)`) та критеріїв якості BPMN 2.0.

2. **Стадія імплементації**

- [01-prompt-bpmn.md](./ai/01-prompt-bpmn.md) та [01-response-bpmn.md](./ai/01-response-bpmn.md) — генерація першої версії декларативної моделі `model.bpmn` та рендеру `render.png`.
- [02-prompt-vertical-layout.md](./ai/02-prompt-vertical-layout.md) та [02-response-vertical-layout.md](./ai/02-response-vertical-layout.md) — перебудова орієнтації пулів на вертикальні колонки з потоком керування згори донизу та горизонтальними `Message Flow`.
- [03-feedback-audit.md](./ai/03-feedback-audit.md) — серія інженерних аудитів та правок через специфікацію: відокремлення `Send Task` від `Service Task`, додавання `Data Objects` (`tournament_registration [PENDING]` / `[PAID]`), винесення `200 OK (Webhook Ack)` перед `XOR`-злиттям і паралельним форком, виправлення моменту перевірки `attempts < 3 та TTL дійсний?` після події `Отримано POST /retry-payment`, фіксація абсолютного дедлайну `min(registered_at + 15 хв, deadline)` та синхронізація очікування відповіді в пулі `Користувач`.

## Топ-3 розбіжності

| #   | Що було не так (проти якого пункту spec)                                                                                                                                         | Мій слід (промпт / фідбек / дод. spec)                                                     | Виправлення (коміт)                                        |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| 1   | **Блокування `200 OK (Webhook Ack)` паралельним форком та хибний ACK при `entry_fee == 0`** (Етапи 3–4, кроки 7–8: затримка пуша блокувала відповідь шлюзу, а при `entry_fee == 0` надсилався зайвий ACK). | [03-feedback-audit.md](./ai/03-feedback-audit.md)<br/>[AUDIT.md](./AUDIT.md)               | `cbdeec3`, `374af8e` (spec)<br/>`1ed7d3d`, `fe78e78` (model) |
| 2   | **Передчасна перевірка `attempts < 3 та TTL дійсний?` до запиту `POST /retry-payment`, відносний таймер та ризик Overbooking** (Етап 1 крок 3 та Етап 3 крок 7: розсинхронізація з 15-хвилинним резервом від `registered_at`). | [03-feedback-audit.md](./ai/03-feedback-audit.md)<br/>[AUDIT.md](./AUDIT.md)               | `6646f33`, `eb80347`, `3cde4c3`, `4916744` (spec)<br/>`9a9128b`, `6a53323`, `8fbef84`, `99e4656` (model) |
| 3   | **Змішування синхронних `Service Task` і асинхронних `Send Task` та відсутність `Data Object` `tournament_registration`** (Критерії прийняття щодо типізації задач та елементів даних із Лаби 1). | [03-feedback-audit.md](./ai/03-feedback-audit.md)<br/>[AUDIT.md](./AUDIT.md)               | `cbdeec3`, `4916744` (spec)<br/>`1ed7d3d`, `99e4656` (model) |

## Ключове рішення

- [0001-bpmn-over-activity-and-state-machine.md](./adr/0001-bpmn-over-activity-and-state-machine.md) (`0821d18`) — Вибір нотації BPMN 2.0 замість UML Activity Diagram та State Machine

### Розглянуті варіанти

1. Машина станів (State Machine / Mermaid `stateDiagram-v2`)
2. UML Activity Diagram (Mermaid `flowchart` / PlantUML Activity)
3. BPMN 2.0 (Business Process Model and Notation — XML)

Обрано `варіант 3`. Наслідки:

Позитивні:

- **Явна типізація задач (Task Types):** суворе розрізнення `User Task` (дія гравця в UI), `Service Task` (транзакція БД або синхронний виклик), `Send Task` (асинхронна відправка повідомлення/відповіді без очікування) та дій поза системою.
- **Чітка семантика повідомлень та асинхронної взаємодії:** пунктирні `Message Flow` між пулами відокремлені від внутрішніх `Sequence Flow`, а `Intermediate Message Catch Event` явно фіксують точки очікування вебхуків і запитів.
- **Відображення артефактів даних (`Data Objects`):** елементи `tournament_registration [PENDING]` $\rightarrow$ `tournament_registration [PAID]` прямо пов'язують кроки процесу з реляційною схемою Лабораторної №1.
- **Розгалуження при очікуванні (`Event-Based Gateway`) і таймери (`Timer Catch Event`):** декларативне моделювання гонитви між `Webhook charge.successful`, `Webhook charge.failed` та абсолютним дедлайном $\min(\text{registered\_at} + 15\text{ хв}, \text{deadline})$.
- **Моделювання дій поза системою (`Black Box Pool`):** зовнішній `Payment Gateway` представлено як згорнутий пул, взаємодія з яким описується виключно через `Message Flow`.

Негативні:

- Вища складність декларативного опису (BPMN 2.0 XML включає як логічний граф процесу `bpmn:process` / `bpmn:collaboration`, так і координатну сітку `bpmndi:BPMNDiagram`).
- Вимагає суворого дотримання правил нотації (заборона `Sequence Flow` між пулами, обов'язкова симетрія `Message Flow` із `Send Task` / `Message Catch Event`).

## Перевірка
- розроблено єдину декларативну модель процесу у форматі BPMN 2.0 XML ([model.bpmn](./model.bpmn)) та згенеровано графічний рендер високої роздільної здатності ([render.png](./render.png)).
- пули розташовані вертикально (`Користувач`, `Billiard CRM`, `Payment Gateway` як Black Box Pool), потоки керування (`Sequence Flow`) розгортаються згори донизу, а міжпулові повідомлення (`Message Flow`) передаються горизонтально.
- наявні умовні розгалуження (`Exclusive Gateway / XOR`), подієві розгалуження очікування (`Event-Based Gateway`) та паралельне розщеплення і синхронізація (`Parallel Gateway / AND — Fork & Join`).
- для кожного міжпулового повідомлення змодельовано відправника, отримувача та реакцію процесу на відсутність відповіді через конкурентний таймер `min(registered_at + 15 хв, deadline)`.
- повна наскрізна узгодженість із моделлю даних [lab-01-er](../lab-01-er/) (`tournament_registration`, статуси `PENDING` / `PAID`), вимогами [lab-02-requirements-usecases](../lab-02-requirements-usecases/) (`UC-04`, `UC-06`, `REQ-06`) та повідомленнями Sequence-діаграм із Лабораторної №3.
