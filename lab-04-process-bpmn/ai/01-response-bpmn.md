# Відповідь моделі: Генерація BPMN 2.0 діаграми для прецеденту онлайн-реєстрації та оплати

Відповідно до специфікації `lab-04-process-bpmn/spec.md` згенеровано декларативну модель бізнес-процесу у стандарті **BPMN 2.0 XML** (`model.bpmn`, сумісну з `bpmn.io` / Camunda Modeler) та її графічний експорт високої роздільної здатності (`render.png`).

---

## 1. Структура пулів та учасників (`bpmn:collaboration`)

1. **`Користувач` (`Participant_User` — White Box Pool):**
   - Описує клієнтський потік дій користувача через задачі типу `bpmn:userTask` та очікування асинхронних відповідей системи за допомогою `bpmn:eventBasedGateway` і `bpmn:intermediateCatchEvent` (`messageEventDefinition`).
2. **`Billiard CRM` (`Participant_CRM` — White Box Pool, суцільна доріжка):**
   - Описує автоматизовану оркестрацію бізнес-правил на стороні сервера через задачі типу `bpmn:serviceTask`, умовні шлюзи (`bpmn:exclusiveGateway`), подієвий шлюз (`bpmn:eventBasedGateway` із `messageEventDefinition` та `timerEventDefinition`) та паралельний шлюз (`bpmn:parallelGateway` Fork/Join).
3. **`Payment Gateway` (`Participant_Gateway` — Black Box Pool):**
   - Змодельовано як згорнутий пул без внутрішнього `processRef`. Уся взаємодія здійснюється виключно через міжпулові потоки повідомлень (`bpmn:messageFlow`).

---

## 2. Реалізація 4 етапів процесу в `model.bpmn`

### Етап 1. Ініціація заявки та перевірка умов допуску
- **`Task_User_ClickRegister` (`userTask`)** $\xrightarrow{\text{MF\_RegisterReq: POST /registrations (JWT)}}$ **`StartEvent_CRM_RegReq` (`messageStartEvent`)**.
- **`Task_CRM_Auth` (`serviceTask`):** валідація JWT-токена та ролі гравця (`verifyJwtAndRole`).
- **`Task_CRM_CheckDB` (`serviceTask`):** блокуюче читання параметрів турніру (`SELECT ... FOR UPDATE`), розрахунок активної квоти з урахуванням 15-хвилинного TTL та перевірка існуючої заявки `existing_reg`.
- **`Gateway_CRM_Validate` (`exclusiveGateway`):**
  - *Порушено інваріант (`403` / `409`)* $\rightarrow$ **`Task_CRM_SendReject` (`serviceTask`)** $\xrightarrow{\text{MF\_RejectResp}}$ **`Event_User_CatchReject`** $\rightarrow$ **`Task_User_ViewError` (`userTask`)** $\rightarrow$ завершення процесу без виконання `INSERT`.
  - *Існує заявка `'PENDING'`* $\rightarrow$ вкладений **`Gateway_CRM_CheckTtlExpired` (`exclusiveGateway`)**: при `TTL > 15 хв` виконується **`Task_CRM_UpdateTtl` (`serviceTask`: `UPDATE registered_at = NOW()`)**, після чого потік переходить до перевірки вартості участі без дублювання `INSERT`.
  - *Нова валідна заявка* $\rightarrow$ **`Task_CRM_InsertPending` (`serviceTask`: `INSERT INTO tournament_registration`)**.

### Етап 2. Перевірка платності та створення Checkout-сесії
- **`Gateway_CRM_CheckFee` (`exclusiveGateway`: `entry_fee > 0?`):**
  - При `entry_fee == 0` потік оминає платіжний шлюз і переходить одразу до Етапу 4 (`Gateway_CRM_MergeToConfirm`).
  - При `entry_fee > 0` виконується **`Task_CRM_InitCheckout` (`serviceTask`)**:
    - `MF_CreateSession`: `POST /v1/checkout/sessions` $\rightarrow$ `Payment Gateway`;
    - `MF_SessionCreated`: `200 OK {session_id, checkout_url}` $\leftarrow$ `Payment Gateway`;
    - `MF_CheckoutUrl`: `201 / 200 {checkout_url}` $\rightarrow$ `Event_User_CatchCheckout` у пулі Користувача.

### Етап 3. Оплата, повторні спроби (Retry) та очікування подій
- **`Task_User_SubmitPayment` (`userTask`)** $\xrightarrow{\text{MF\_SubmitCard: submitPayment(cardDetails)}}$ **`Payment Gateway`**.
- **`Gateway_CRM_WaitEvents` (`eventBasedGateway` у `Billiard CRM`):**
  1. **`Event_CRM_WebhookSuccess` (`messageCatchEvent`)** $\leftarrow$ `MF_WebhookSuccess`:
     - **`Task_CRM_VerifySuccessSig` (`serviceTask`: `verifyHmacSignature`)** $\rightarrow$ перехід до Етапу 4.
  2. **`Event_CRM_WebhookFailed` (`messageCatchEvent`)** $\leftarrow$ `MF_WebhookFailed`:
     - **`Task_CRM_HandleFail` (`serviceTask`)** перевіряє підпис, залишає статус `'PENDING'` та повертає `200 OK (Ack)` шлюзу (`MF_WebhookFailAck`).
     - Користувач отримує `redirect(failure_url)` (`MF_RedirectFail` $\rightarrow$ `Event_User_CatchPayError`) і на шлюзі **`Gateway_User_RetryChoice`** (`attempts < 3`) обирає повторну спробу (**`Task_User_ClickRetry` (`userTask`)** $\xrightarrow{\text{MF\_RetryReq: POST /retry-payment}}$ **`Event_CRM_CatchRetry`**).
     - **`Task_CRM_CreateRetrySession` (`serviceTask`)** створює нову сесію в `Payment Gateway` (`MF_RetrySessionReq` / `MF_RetrySessionResp`), повертає новий `checkout_url` користувачу (`MF_RetryCheckoutUrl`) та повертає потік у цикл очікування.
     - Якщо ліміт спроб вичерпано (`Gateway_CRM_CheckRetry` $\rightarrow$ `Ні`), потік переходить до завершення сесії за тайм-аутом.
  3. **`Event_CRM_TimerTtl` (`timerCatchEvent`: `min(15 хв, deadline)`):**
     - **`Task_CRM_ExpireSession` (`serviceTask`)** залишає заявку в статусі `'PENDING'` (слот вивільняється у розрахунку квоти) та надсилає `pushEvent(SESSION_EXPIRED)` (`MF_PushExpired` $\rightarrow$ `Event_User_CatchExpired`) $\rightarrow$ завершення процесу за тайм-аутом.

### Етап 4. Паралельне підтвердження участі та сповіщення
- **`Gateway_CRM_ParFork` (`parallelGateway` — Fork):**
  - *Гілка 1:* **`Task_CRM_UpdatePaid` (`serviceTask`)** — `UPDATE tournament_registration SET payment_status = 'PAID'`.
  - *Гілка 2:* **`Task_CRM_PushConfirmed` (`serviceTask`)** — `pushEvent(REGISTRATION_CONFIRMED)` (`MF_PushConfirmed` $\rightarrow$ `Event_User_CatchConfirmed` $\rightarrow$ **`Task_User_ViewSuccess` (`userTask`)** $\rightarrow$ `EndEvent_User_Success`).
- **`Gateway_CRM_ParJoin` (`parallelGateway` — Join)** синхронізує обидві гілки $\rightarrow$ **`Task_CRM_AckWebhook` (`serviceTask`)** повертає `200 OK (Webhook Ack)` (`MF_WebhookSuccessAck`) у `Payment Gateway` $\rightarrow$ `EndEvent_CRM_Success`.

---

## 3. Згенеровані файли
- Декларативна модель BPMN 2.0 XML: [`lab-04-process-bpmn/model.bpmn`](../model.bpmn)
- Графічний рендер діаграми: [`lab-04-process-bpmn/render.png`](../render.png)
