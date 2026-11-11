# spec.md - Процес / поведінка (Activity, BPMN або машина станів)

## Намір

Описати один бізнес-процес Billiard CRM однією точною формою - BPMN діаграмою.<br/>Обраний Use-Case - **`UC-04` (Зареєструватися в індивідуальному турнірі)** з обов'язковим включенням (`«include»`) **`UC-06` (Оплатити турнірний внесок)** із Лабораторної №2.

## Пули та лейни

Пули розташовуються вертикально (колонками зліва направо: Користувач, Billiard CRM, Payment Gateway); внутрішні потоки керування (`Sequence Flow`) розгортаються згори донизу, а міжпулові повідомлення (`Message Flow`) передаються горизонтально.

1. Користувач
2. Billiard CRM
3. Payment Gateway (black box pool) - пул, внутрішня логіка якого свідомо не моделюється

## Кроки процесу

### Етап 1. Ініціація заявки та перевірка умов допуску
1. **Ініціація запиту (`User Task` у пулі Користувача):** натискання «Зареєструватися» (`clickRegister(tournamentId)`) та надсилання `Message Flow`: `POST /api/tournaments/{id}/registrations` із JWT-токеном до `Billiard CRM`.
2. **Автентифікація (`Service Task` у `Billiard CRM`):** валідація JWT-токена та ролі гравця (`verifyJwtAndRole`).
3. **Перевірка інваріантів у БД (`Service Task`):** блокуюче читання параметрів турніру (`SELECT ... FOR UPDATE`), підрахунок активних заявок (`payment_status = 'PAID'` або `'PENDING'` із `registered_at + 15 min > NOW()`) та пошук існуючої заявки `existing_reg`.
4. **Шлюз валідації (`Exclusive Gateway / XOR`):**
   - **Відмова** (`now() > registration_deadline` $\rightarrow$ `403`, або `active_count >= max_participants` без активної броні $\rightarrow$ `409`, або `existing_reg.payment_status == 'PAID'` $\rightarrow$ `409`): повернення помилки без `INSERT` (`Send Task`) $\xrightarrow{\text{Message Flow}}$ відображення помилки користувачу $\rightarrow$ `End Event`.
   - **Існуюча заявка `'PENDING'`** (за наявності місць): вкладений `XOR` (`registered_at + 15 min < now()`) — за потреби оновлення `UPDATE registered_at = NOW()` (`Service Task`, оновлює `Data Object: tournament_registration [PENDING]`), перехід до отримання сесії оплати без повторного `INSERT`.
   - **Нова валідна заявка:** `INSERT INTO tournament_registration (player_id, team_id=NULL, payment_status='PENDING', seed_number=NULL, registered_at=NOW())` (`Service Task`, створює `Data Object: tournament_registration [PENDING]`).

### Етап 2. Перевірка платності та створення Checkout-сесії
5. **Шлюз вартості участі (`Exclusive Gateway / XOR` за умовою `entry_fee > 0`):**
   - **`entry_fee == 0`:** перехід напряму до Етапу 4 (фіксація `'PAID'` та сповіщення).
   - **`entry_fee > 0`:** виклик `initiateCheckout` (`Service Task`, читає `Data Object: tournament_registration [PENDING]`) $\xrightarrow{\text{Message Flow: POST /v1/checkout/sessions}}$ `Payment Gateway` $\xrightarrow{\text{Message Flow: 200 OK \{session\_id, checkout\_url\}}}$ `Billiard CRM`, після чого окремий `Send Task` (`Надіслати checkout_url користувачу`) передає `201 / 200 {checkout_url}` $\xrightarrow{\text{Message Flow}}$ користувачу для переходу на Checkout.

### Етап 3. Оплата, повторні спроби (Retry) та очікування подій
6. **Подання платіжних даних (`User Task` у пулі Користувача):** введення реквізитів картки та відправка `submitPayment(cardDetails, session_id)` $\xrightarrow{\text{Message Flow}}$ `Payment Gateway`.
7. **Очікування результату транзакції або тайм-ауту (`Event-Based Gateway` у `Billiard CRM` на 3 гілки):**
   - **Webhook `charge.successful` (`Message Catch Event` від `Payment Gateway`):** перевірка HMAC-підпису (`verifyHmacSignature`, `Service Task`) $\rightarrow$ миттєве підтвердження вебхука `200 OK (Webhook Ack)` (`Send Task`) $\xrightarrow{\text{Message Flow}}$ `Payment Gateway` $\rightarrow$ перехід до Етапу 4.
   - **Webhook `charge.failed` (`Message Catch Event` від `Payment Gateway`):** перевірка підпису, збереження статусу `'PENDING'`, відповідь `200 OK` шлюзу (`Service Task`), тоді як користувач отримує повідомлення про помилку оплати від шлюзу (`redirect failure_url`). Далі у `Billiard CRM` очікується вхідна подія **`Отримано POST /retry-payment` (`Message Catch Event` від `clickRetryPayment` користувача)**, після якої запускається перевірка на шлюзі `Exclusive Gateway (XOR)` (`attempts < 3 AND now() <= min(deadline, registered_at + 15 min)`):
     - **Так (дозволено повторну спробу):** `createRetrySession` (`Service Task`) $\xrightarrow{\text{Message Flow}}$ `Payment Gateway` $\rightarrow$ `Send Task` (`Надіслати новий checkout_url користувачу`) $\xrightarrow{\text{Message Flow}}$ повернення до очікування оплати (та до кроку 6 у пулі Користувача).
     - **Ні (перевищено ліміт спроб або сплив TTL):** перехід до завершення сесії (`SESSION_EXPIRED`).
   - **Тайм-аут броні $\min(\text{registered\_at} + 15\text{ хв}, \text{deadline})$ (`Timer Catch Event` із семантикою фіксованого дедлайну `timeDate`):** єдиний таймер на весь процес оплати, прив'язаний до початкового `registered_at` (не скидається на нові 15 хв при повторних спробах); за його спрацювання перевірка підпису не виконується, заявка залишається `'PENDING'` (слот вивільняється у квоті), надсилання `pushEvent("SESSION_EXPIRED")` (`Send Task`) $\xrightarrow{\text{Message Flow}}$ користувачу $\rightarrow$ `End Event`.

### Етап 4. Паралельне підтвердження реєстрації в БД та сповіщення клієнта
8. **Злиття шляхів та розщеплення потоку (`XOR Merge` $\rightarrow$ `Parallel Gateway / AND — Fork`):** об'єднує гілку підтвердженого вебхука та гілку безкоштовної участі (`entry_fee == 0`), після чого паралельно запускає:
   - **Гілка 1 (`Service Task`):** атомарний `UPDATE tournament_registration SET payment_status = 'PAID' WHERE id = ? AND payment_status = 'PENDING'`, що формує `Data Object: tournament_registration [PAID]`.
   - **Гілка 2 (`Send Task`):** асинхронне надсилання `pushEvent("REGISTRATION_CONFIRMED")` $\xrightarrow{\text{Message Flow}}$ користувачу (`showNotification`).
9. **Синхронізація (`Parallel Gateway / AND — Join`):** об'єднання паралельних гілок та успішне завершення процесу (`End Event`) у пулах `Billiard CRM` і Користувача.

## Критерії прийняття
- [ ] **Артефакт (текст + рендер)** - BPMN діаграма (BPMN 2.0 XML з bpmn.io)
- [ ] є ≥1 розгалуження й ≥1 паралельна гілка
- [ ] узгодженість з UC/Sequence
- [ ] **Вертикальна орієнтація:** пули розташовані вертикально (колонками зліва направо), внутрішній потік керування (Sequence Flow) спрямований згори донизу, а міжпулові повідомлення (Message Flow) передаються горизонтально.
- [ ] **Локалізація активностей:** жодна активність не висить у повітрі - кожен елемент належить конкретному лейну.
- [ ] **Чорна скринька:** Payment Gateway змодельовано як Black Box Pool (порожній пул, взаємодія лише через Message Flow).
- [ ] **Типізація задач:** суворе розрізнення типів - User Task (дія користувача у системі), Manual Task (поза системою), Service Task (автоматична внутрішня обробка / синхронний виклик), Send Task (асинхронна відправка повідомлення без очікування синхронної відповіді).
- [ ] **Елементи даних (Data Objects):** явно відображено життєвий цикл об'єкта даних `tournament_registration [PENDING]` та `tournament_registration [PAID]` (`bpmn:dataObjectReference`) із пунктирними асоціаціями даних.
- [ ] **Заборонені типи задач:** Script Task та Business Rule Task відсутні в моделі.
- [ ] **Обов'язкова комунікація:** наявне ≥1 повідомлення (Message Flow) між пулами з обробкою відповіді та її відсутності.
- [ ] **Семантика сповіщень (Message vs Signal):** адресні виклики - Message (конверт), широкомовні події - Signal (трикутник).
