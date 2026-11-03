# spec.md - Взаємодія (Sequence Diagrams)

## 1. Намір та системні межі
Відобразити внутрішню динаміку системи Billiard CRM для всіх сценаріїв прецеденту онлайн-реєстрації у декларативному синтаксисі Mermaid `sequenceDiagram`.

## 2. Обраний Use Case
- **`UC-04` (Зареєструватися в індивідуальному турнірі)** з обов'язковим включенням (`«include»`) **`UC-06` (Оплатити турнірний внесок)** із Лабораторної №2.

## 3. Учасники взаємодії

### Актори
- **`Player`**: автентифікований спортсмен із активним профілем (`users.role = 'USER'`, `player.person_id = users.user_id`).
- **`Payment Gateway`**: зовнішній платіжний провайдер (Stripe/LiqPay) для онлайн-транзакцій та асинхронних вебхуків.

### Компоненти / Об'єкти
- **`Frontend (WebUI)`**: клієнтський SPA-інтерфейс спортсмена.
- **`AuthMiddleware`**: шар автентифікації та авторизації (перевірка JWT, `exp`, вилучення `user_id` та ролі `Player`).
- **`RegistrationController`**: REST API контролер для прийому клієнтських запитів та вебхуків.
- **`RegistrationService`**: сервіс бізнес-правил (перевірка квоти, дедлайну, дублікатів, XOR-інваріанта).
- **`PaymentService`**: сервіс взаємодії з платіжним шлюзом (створення сесій, верифікація підписів вебхуків).
- **`Database`**: реляційна СУБД PostgreSQL (`tournament`, `tournament_registration`, `player`, `users`).

---

## 4. Сценарії та покрокова взаємодія

### Сценарій 1 (Happy Path): Успішна реєстрація з онлайн-оплатою
- **Кроки:**
  1. `Player -> Frontend`: запит на реєстрацію `clickRegister(tournamentId)`.
  2. `Frontend -> AuthMiddleware`: `POST /api/tournaments/{id}/registrations` із `Bearer <jwt>`.
  3. `AuthMiddleware -> RegistrationController`: токен валідовано, передача `AuthContext(userId, playerId)`.
  4. `RegistrationController -> RegistrationService`: `registerPlayer(playerId, tournamentId)`.
  5. `RegistrationService -> Database`: блокуюче читання `SELECT ... FOR UPDATE` (дедлайн, статус `'REGISTRATION_OPEN'`, квота).
  6. `RegistrationService -> Database`: `INSERT INTO tournament_registration (status='PENDING', seed_number=NULL)`.
  7. `RegistrationService -> PaymentService`: `initiateCheckout(registrationId, amount)`.
  8. `PaymentService -> Payment Gateway`: `POST /v1/checkout/sessions`.
  9. `RegistrationController -> Frontend`: `201 Created (checkoutUrl)`.
  10. `Frontend -> Player`: перенаправлення на Hosted Checkout.
  11. `Payment Gateway -> RegistrationController`: асинхронний Webhook `POST /api/webhooks/payment` із сигнатурою.
  12. `PaymentService -> RegistrationService`: підпис підтверджено, `confirmPayment(registrationId)`.
  13. `RegistrationService -> Database`: `UPDATE tournament_registration SET payment_status = 'PAID'`.
  14. `RegistrationController -> Payment Gateway`: `200 OK`.
- **Комбіновані фрагменти:**
  - `opt [entry_fee > 0]`: створення сесії оплати (для безкоштовного турніру одразу призначається `'PAID'`).
  - `par [Оновлення статусу в БД || Сповіщення клієнта в WebUI]`.

### Сценарій 2 (Alternative / Retry): Помилка списання коштів та повторна спроба
- **Кроки:**
  1. `Player -> Payment Gateway`: введення реквізитів на платіжній сторінці.
  2. `Payment Gateway -> RegistrationController`: асинхронний Webhook `payment.failed` (відхилення картки).
  3. `PaymentService -> Database`: фіксація помилки, статус заявки залишається `'PENDING'` (активна до дедлайну).
  4. `Payment Gateway -> Frontend`: повернення на сторінку відмови (`failure_url`).
  5. `Frontend -> Player`: повідомлення про відхилення та відображення кнопки «Спробувати знову».
  6. Гравець натискає «Спробувати знову».
  7. `Frontend -> AuthMiddleware -> RegistrationController`: `POST /api/registrations/{id}/retry-payment`.
  8. `PaymentService -> Payment Gateway`: генерація нової Checkout-сесії.
  9. `Frontend -> Player`: перенаправлення на оновлену форму оплати.
- **Комбіновані фрагменти:**
  - `loop [поки attempts < 3 та now() <= registration_deadline]`: цикл повторних спроб оплати.
  - `alt [успіх повторного списання] else [тайм-аут або вичерпання ліміту спроб]`.

### Сценарій 3 (Exception / Validation): Відхилення заявки через порушення інваріантів
- **Кроки:**
  1. `Player -> Frontend`: запит на реєстрацію.
  2. `Frontend -> AuthMiddleware`: `POST /api/tournaments/{id}/registrations`.
  3. `AuthMiddleware -> RegistrationController`: передача контексту запиту.
  4. `RegistrationController -> RegistrationService`: `registerPlayer(playerId, tournamentId)`.
  5. `RegistrationService -> Database`: перевірка параметрів турніру та існуючих заявок гравця.
  6. Валідація бізнес-правил без створення запису (`INSERT` відсутній):
     - `now() > registration_deadline` -> `403 Forbidden` (*«Дедлайн реєстрації минув»*).
     - `current_count >= max_participants` -> `409 Conflict` (*«Квоту місць вичерпано»*).
     - `existing_registration != null` -> `409 Conflict` (*«Гравець вже зареєстрований»*).
  7. `RegistrationController -> Frontend`: повернення помилки.
  8. `Frontend -> Player`: повідомлення про причину відхилення.
- **Комбіновані фрагменти:**
  - `alt [now() > deadline] else [count >= max] else [duplicate registration]`.

---

## 5. Формальні критерії прийняття
1. **Покриття сценаріїв:** для кожного з 3 сценаріїв розроблено окрему Mermaid-діаграму у `model/` (`model-01-success.mmd`, `model-02-payment-failure.mmd`, `model-03-validation-error.mmd`).
2. **Графічний рендер:** збережено окремі експорти у `render/` та зведено у підсумковий `render.png`.
3. **Комбіновані фрагменти:** кожен сценарій містить щонайменше один фрагмент (`opt`, `loop`, `alt`, `par`).
4. **Неперервність Activation Bars:** смуги активності починаються до умовних блоків і завершуються після них.
5. **Наскрізна цілісність:** узгодженість із [Лабораторною №1](../lab-01-er/model.mmd) та [Лабораторною №2](../lab-02-requirements-usecases/requirements.md).
