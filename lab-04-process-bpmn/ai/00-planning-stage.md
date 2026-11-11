# Planning Stage

## Prompt

```
проаналізуй lab-01-er/spec.md та запропонуй кроки процесу відповідно до вже існуючої sequence-діаграми
```

## Response

### 1. Аналіз `lab-01-er/spec.md` у контексті процесу реєстрації та оплати

У процесі індивідуальної реєстрації на турнір з онлайн-оплатою задіяні чотири сутності з ER-моделі (`lab-01-er/spec.md`):

1. **`users` та `player` (зв'язок `1:0..1` через `player.person_id -> users.user_id`):**
   - Ініціатором процесу виступає автентифікований користувач (`users.deleted_at IS NULL`, `users.role = 'USER'`), який має активний спортивний профіль `player` (`player_id`).
2. **`tournament`:**
   - Визначає умови допуску до реєстрації:
     - `tournament_status` — має бути у стані `'REGISTRATION_OPEN'`;
     - `registration_deadline` — дедлайн прийому заявок (`now() <= registration_deadline`);
     - `max_participants` — ліміт учасників турніру;
     - `entry_fee` — розмір вступного внеску (`numeric`).
3. **`tournament_registration`:**
   - Фіксує стан заявки конкретного гравця на турнір:
     - XOR-інваріант учасника: для індивідуального турніру заповнюється `player_id`, а `team_id = NULL`;
     - `seed_number` — при створенні заявки ініціалізується як `NULL` (визначається пізніше під час посіву сітки);
     - `registered_at` — часова мітка створення/оновлення броні слота (використовується для розрахунку 15-хвилинного вікна резервування TTL);
     - `payment_status` — канонічний перелік станів оплати (`'PENDING'`, `'PAID'`, `'WAIVED'`, `'REFUNDED'`). Оскільки статус `'CANCELLED'` у схемі відсутній, прострочені заявки залишаються у `'PENDING'`, але не враховуються в активній квоті `max_participants`.

---

### 2. Пропоновані кроки процесу (відповідно до Sequence-діаграм із Лабораторної №3)

У Лабораторній №3 взаємодія була поділена на 3 окремі Sequence-діаграми (*Happy Path*, *Payment Failure & Retry*, *Validation Error / Resume PENDING*). У BPMN-процесі вони об'єднуються в єдиний потік керування через шлюзи (`XOR`, `Event-Based`, `AND`):

#### Етап 1. Ініціація заявки та перевірка умов допуску
1. **Ініціація запиту гравцем (`User Task`):**
   - Гравець натискає «Зареєструватися» в інтерфейсі (`clickRegister(tournamentId)`).
   - Надсилається повідомлення (`Message Flow`): `POST /api/tournaments/{id}/registrations` із JWT-токеном до `Billiard CRM`.
2. **Автентифікація та перевірка ролі (`Service Task`):**
   - Перевірка валідності JWT-токена та наявності `player_id` у профілі (`verifyJwtAndRole`).
3. **Перевірка інваріантів турніру та квоти в БД (`Service Task`):**
   - Читання параметрів турніру (`SELECT ... FOR UPDATE`) та підрахунок активних заявок: `payment_status = 'PAID'` або (`payment_status = 'PENDING'` і `registered_at + 15 min > NOW()`), а також пошук існуючої заявки гравця `existing_reg`.
4. **Розгалуження за результатами валідації (`Exclusive Gateway / XOR`):**
   - **Гілка 1 — Відмова (порушення інваріантів):**
     - Умови: `now() > registration_deadline` (`403 Forbidden`), АБО `active_count >= max_participants` за відсутності активної броні (`409 Conflict`), АБО `existing_reg.payment_status == 'PAID'` (`409 Conflict`).
     - Дія (`Service Task`): повернення відповідної помилки без виконання `INSERT`.
     - Повідомлення (`Message Flow`): передача помилки гравцю $\rightarrow$ відображення причини відмови $\rightarrow$ завершення процесу (`End Event`).
   - **Гілка 2 — Відновлення існуючої заявки `PENDING`:**
     - Умова: гравець уже має заявку зі статусом `'PENDING'` і квота дозволяє участь.
     - Дія (`Service Task`): якщо `registered_at + 15 min < now()`, виконується `UPDATE tournament_registration SET registered_at = NOW()`. Повторний `INSERT` не виконується; потік переходить до отримання сесії оплати.
   - **Гілка 3 — Створення нової заявки:**
     - Умова: всі перевірки пройдено успішно, попередньої заявки немає.
     - Дія (`Service Task`): `INSERT INTO tournament_registration (tournament_id, player_id, team_id=NULL, payment_status='PENDING', seed_number=NULL, registered_at=NOW())`.

#### Етап 2. Перевірка платності та створення Checkout-сесії
5. **Перевірка розміру внеску (`Exclusive Gateway / XOR` за умовою `entry_fee > 0`):**
   - **Якщо `entry_fee == 0` (безкоштовний турнір):** перехід одразу до Етапу 4 (фіксація статусу `'PAID'` та сповіщення).
   - **Якщо `entry_fee > 0` (платний турнір):**
     - Дія (`Service Task`): виклик `initiateCheckout(regId, amount)`.
     - Повідомлення (`Message Flow` до `Payment Gateway`): `POST /v1/checkout/sessions {amount, reg_id, expires_in: min(15m, deadline)}`.
     - Повідомлення (`Message Flow` від `Payment Gateway`): `200 OK {session_id, checkout_url}`.
     - Повідомлення (`Message Flow` до гравця): `201 Created / 200 OK {checkout_url}` (перенаправлення на сторінку оплати).

#### Етап 3. Проведення оплати, повторні спроби (Retry) та очікування подій
6. **Подання платіжних даних (`User Task` у пулі гравця):**
   - Гравець вводить реквізити картки на сторінці шлюзу та відправляє форму (`submitPayment(cardDetails, session_id)` $\xrightarrow{\text{Message Flow}}$ `Payment Gateway`).
7. **Очікування результату транзакції або тайм-ауту (`Event-Based Gateway` у `Billiard CRM`):**
   - **Гілка А — Успішне списання (`Intermediate Message Catch Event: Webhook charge.successful`):**
     - Надходить `POST /api/webhooks/payment {event: 'charge.successful'}` від `Payment Gateway`.
     - Дія (`Service Task`): криптографічна перевірка `verifyHmacSignature` $\rightarrow$ перехід до Етапу 4.
   - **Гілка Б — Помилка списання (`Intermediate Message Catch Event: Webhook charge.failed`):**
     - Надходить `POST /api/webhooks/payment {event: 'charge.failed'}` від `Payment Gateway`.
     - Дія (`Service Task`): перевірка підпису вебхука; статус заявки в БД залишається `'PENDING'`; повернення `200 OK` шлюзу.
     - Гравець отримує повідомлення про відхилення оплати (`redirect failure_url` / `showPaymentError`).
     - **Розгалуження (`Exclusive Gateway / XOR` — повторна спроба):**
       - Якщо гравець обирає повторну спробу і виконується умова `attempts < 3 AND now() <= min(deadline, registered_at + 15 min)`:
         - Дія гравця (`User Task`): `clickRetryPayment` $\xrightarrow{\text{Message Flow: POST /retry-payment}}$ `Billiard CRM`.
         - Дія системи (`Service Task`): `createRetrySession` $\xrightarrow{\text{Message Flow}}$ `Payment Gateway` $\rightarrow$ отримання нового `checkout_url` $\rightarrow$ повернення до кроку 6.
       - Якщо ліміт спроб вичерпано або гравець відмовляється: перехід до завершення без оплати.
   - **Гілка В — Відсутність оплати / спливання часу броні (`Intermediate Timer Catch Event: min(15m, deadline)`):**
     - Спрацьовує таймер резервування слота.
     - Дія (`Service Task`): заявка залишається в БД зі статусом `'PENDING'` (слот автоматично вивільняється у розрахунку квоти), система надсилає гравцю повідомлення `pushEvent("SESSION_EXPIRED")` $\rightarrow$ завершення процесу за тайм-аутом (`End Event`).

#### Етап 4. Паралельне підтвердження участі та сповіщення
8. **Розщеплення потоку після успішної оплати (`Parallel Gateway / AND — Fork`):**
   - **Паралельна гілка 1 (`Service Task`):** оновлення запису в БД — `UPDATE tournament_registration SET payment_status = 'PAID' WHERE tournament_registration_id = ? AND payment_status = 'PENDING'`.
   - **Паралельна гілка 2 (`Service Task`):** асинхронне сповіщення клієнта — `pushEvent("REGISTRATION_CONFIRMED")` $\xrightarrow{\text{Message Flow}}$ пул гравця.
9. **Синхронізація (`Parallel Gateway / AND — Join`):**
   - Після завершення обох гілок система повертає `200 OK (Webhook Acknowledged)` у `Payment Gateway` (`Message Flow`) та завершує процес успіхом (`End Event`).
   - У пулі гравця: отримання підтвердження (`showNotification`) $\rightarrow$ успішне завершення процесу (`End Event`).

---

### 3. Питання для узгодження плану:
1. **Декомпозиція пулу `Billiard CRM` на доріжки (Lanes):** Чи виділяємо всередині пулу `Billiard CRM` окремі доріжки для сервісів (наприклад, `Registration Service` та `Payment Service`), чи залишаємо `Billiard CRM` однією суцільною доріжкою?
2. **Умови валідації (Етап 1):** Чи об'єднуємо перевірки дедлайну, квоти та дубліката в один `Service Task` + один `Exclusive Gateway (XOR)` із 3 вихідними потоками (відмова / відновлення `PENDING` / нова заявка), чи розбиваємо їх на послідовні шлюзи?
3. **Гілка безкоштовного турніру (`entry_fee == 0`):** Чи залишаємо шлюз перевірки `entry_fee > 0`, чи в межах даного BPMN-процесу розглядаємо виключно платні турніри (оскільки `UC-06` підключено через обов'язковий `«include»`)?

---

## Prompt

```
1. залишаємо Billiard CRM як суцільну доріжку
2. який сенс у рознесенні перевірки на послідовні шлюзи?
3. xor шлюз обов'язковий, оскільки повинна бути повна відповідність sequence діаграмі
```

## Response

### 1. Пул `Billiard CRM` та шлюз `entry_fee > 0`
Зафіксовано:
- Пул **`Billiard CRM`** залишається суцільним (без внутрішніх під-доріжок / Lanes).
- `Exclusive Gateway (XOR)` для перевірки **`entry_fee > 0`** обов'язково зберігається для повної відповідності фрагменту `opt [entry_fee > 0]` зі Сценарію 1 Sequence-діаграми (при `entry_fee == 0` потік оминає звернення до `Payment Gateway` і переходить напряму до паралельного блоку фіксації `PAID` та сповіщення гравця).

---

### 2. Чи є сенс розносити перевірки валідації на послідовні шлюзи?

У BPMN рознесення умов на каскад послідовних шлюзів («драбинку» ромбів) має сенс лише у двох випадках:
1. **Поетапне відсікання важких запитів (Fail-Fast між різними `Service Task`):** наприклад, спочатку перевірити дешеву умову в пам'яті, і лише якщо вона істинна — робити дорогий запит до БД чи зовнішнього сервісу. Проте в нашій Sequence-діаграмі (`model-03-validation-error.mmd`) обидва `SELECT` до БД виконуються **до** входу в блок `alt`.
2. **Принципово різна бізнес-реакція на кожну помилку:** наприклад, якби перевищення квоти запускало окремий підпроцес додавання у Waitlist, а дублікат заявки ініціював повернення коштів. У нашій же системі всі три порушення інваріантів (`now() > registration_deadline` $\rightarrow$ `403`, `active_count >= max_participants` $\rightarrow$ `409`, `existing_reg.status == 'PAID'` $\rightarrow$ `409`) ведуть до однієї дії — відмови без `INSERT` та повернення клієнту повідомлення про помилку.

Тому в нашому процесі каскад послідовних шлюзів лише перевантажить граф зайвими ромбами. Натомість **один `Exclusive Gateway (XOR)` після читання даних із БД** є прямим структурним відповідником комбінованого фрагмента `alt ... else ...` зі Сценарію 3 Sequence-діаграми й має 3 чіткі виходи:
1. **Порушено інваріант допуску** (`now() > deadline` АБО `active_count >= max` без активної броні АБО `existing_reg.status == 'PAID'`) $\rightarrow$ `Service Task` формування помилки (`403` / `409`) $\rightarrow$ `Message Flow` гравцю $\rightarrow$ завершення процесу.
2. **Існує заявка у статусі `'PENDING'`** (за наявності місць) $\rightarrow$ вкладений `XOR` (відповідник `opt [existing_reg.registered_at + 15m < now()]` у Сценарії 3): якщо 15-хв TTL минув, виконується `Service Task` `UPDATE registered_at = NOW()`, після чого потік переходить до отримання сесії оплати без нового `INSERT`.
3. **Нова валідна заявка** (`existing_reg == NULL`) $\rightarrow$ `Service Task` `INSERT INTO tournament_registration (status='PENDING', seed_number=NULL)` $\rightarrow$ перехід до шлюзу `entry_fee > 0`.
