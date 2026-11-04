# Відповідь моделі: Регенерація Sequence-діаграм після інженерного аудиту

На основі зауважень аудиту ([ai/01-feedback-sequence.md](./01-feedback-sequence.md)) та результатів уточнюючого інтерв'ю ([ai/02-prompt-sequence.md](./02-prompt-sequence.md)):

1. **Виправлено галюцинацію статусу `payment_status`:**
   - Повністю вилучено неіснуючий статус `'CANCELLED'`.
   - У скоупі взаємодії використовуються виключно валідні значення ENUM із Лабораторної №1: `'PENDING'` та `'PAID'`.
   - При вичерпанні спроб оплати чи тайм-ауті сесії статус залишається `'PENDING'` у БД (відповідає вимозі REQ-06), а слот автоматично вивільняється після 15 хвилин за правилом TTL.
2. **Впроваджено 15-хвилинне вікно резервування (Slot Hold TTL):**
   - Розрахунок зайнятих місць враховує лише оплачені або актуальні броні:
     `SELECT COUNT(*) FROM tournament_registration WHERE tournament_id = 5 AND (payment_status = 'PAID' OR (payment_status = 'PENDING' AND registered_at + INTERVAL '15 minutes' > NOW()))`.
   - Час броні суворо обмежується дедлайном турніру: $\min(15\text{ хв}, \text{час до дедлайну})$.
3. **Уточнено логіку валідації дублікатів (Сценарій 3):**
   - Помилка `409 Conflict` повертається лише у разі, якщо гравець уже оплатив турнір (`payment_status == 'PAID'`).
   - Якщо статус `'PENDING'` і бронь активна ($\le 15$ хв) — система повертає `200 OK` з існуючим посиланням на оплату.
   - Якщо статус `'PENDING'`, але бронь спливла ($> 15$ хв) — система перевіряє вільні місця: якщо є, оновлює `registered_at = NOW()` та повертає нову платіжну форму; якщо місць немає — повертає `409 Conflict («Бронь вичерпано, вільних місць немає»)`.
4. **Очищено кореневу структуру:**
   - Видалено депрекейтовані файли `model.mmd` та `render.png` у корені. Усі актуальні артефакти розміщено у `model/` та `render/`.

---

## 1. Сценарій 1 (Happy Path): `model/model-01-success.mmd`

```mermaid
sequenceDiagram
    autonumber
    actor Player as Player (Гравець)
    participant WebUI as Frontend (WebUI)
    participant Auth as AuthMiddleware
    participant RegCtrl as RegistrationController
    participant RegSvc as RegistrationService
    participant PaySvc as PaymentService
    actor Gateway as Payment Gateway
    participant DB as Database (PostgreSQL)

    Player ->> WebUI: clickRegister(tournamentId=5)
    activate WebUI
    WebUI ->> Auth: POST /api/tournaments/5/registrations<br/>Header: Authorization: Bearer jwt_token
    activate Auth
    Auth ->> Auth: verifyJwtAndRole(jwt)<br/>[claims: user_id=10, role=USER, player_id=42]
    Auth ->> RegCtrl: forward(req, AuthContext(userId=10, playerId=42))
    activate RegCtrl
    RegCtrl ->> RegSvc: registerPlayer(playerId=42, tournamentId=5)
    activate RegSvc

    RegSvc ->> DB: SELECT * FROM tournament WHERE tournament_id = 5 FOR UPDATE
    activate DB
    DB -->> RegSvc: tournament(status='REGISTRATION_OPEN', deadline, max=32, fee=500.00)
    deactivate DB

    RegSvc ->> DB: SELECT COUNT(*) FROM tournament_registration<br/>WHERE tournament_id = 5 AND (payment_status = 'PAID' OR (payment_status = 'PENDING' AND registered_at + INTERVAL '15 minutes' > NOW()))
    activate DB
    DB -->> RegSvc: active_count = 14
    deactivate DB

    RegSvc ->> DB: INSERT INTO tournament_registration (tournament_id, player_id, team_id, payment_status, seed_number, registered_at)<br/>VALUES (5, 42, NULL, 'PENDING', NULL, NOW()) RETURNING tournament_registration_id
    activate DB
    DB -->> RegSvc: tournament_registration_id = 101
    deactivate DB

    opt [entry_fee > 0]
        RegSvc ->> PaySvc: initiateCheckout(regId=101, amount=500.00)
        activate PaySvc
        PaySvc ->> Gateway: POST /v1/checkout/sessions<br/>{amount: 500.00, reg_id: 101, expires_in: min(15m, deadline)}
        activate Gateway
        Gateway -->> PaySvc: 200 OK {session_id: 'cs_101', checkout_url}
        deactivate Gateway
        PaySvc -->> RegSvc: checkoutUrl = 'https://pay.gateway.com/pay/cs_101'
        deactivate PaySvc
    end

    RegSvc -->> RegCtrl: RegistrationResult(regId=101, status='PENDING', checkoutUrl)
    deactivate RegSvc
    RegCtrl -->> Auth: 201 Created {registration_id: 101, status: 'PENDING', checkout_url}
    deactivate RegCtrl
    Auth -->> WebUI: 201 Created (checkout_url)
    deactivate Auth
    WebUI -->> Player: redirect(checkoutUrl)
    deactivate WebUI

    Note over Player, Gateway: Асинхронна фаза оплати та Webhook

    Player ->> Gateway: submitPayment(cardDetails, session_id='cs_101')
    activate Gateway
    Gateway ->> Gateway: processCharge(500.00) -> SUCCESS
    Gateway -) RegCtrl: POST /api/webhooks/payment<br/>Header: Stripe-Signature<br/>Payload: {event: 'charge.successful', reg_id: 101}
    deactivate Gateway
    activate RegCtrl

    RegCtrl ->> PaySvc: handleWebhook(payload, signature)
    activate PaySvc
    PaySvc ->> PaySvc: verifyHmacSignature(payload, signature, secret)
    PaySvc ->> RegSvc: confirmPayment(registrationId=101)
    activate RegSvc

    par [Оновлення статусу заявки в БД]
        RegSvc ->> DB: UPDATE tournament_registration<br/>SET payment_status = 'PAID'<br/>WHERE tournament_registration_id = 101 AND payment_status = 'PENDING'
        activate DB
        DB -->> RegSvc: updated (1 row)
        deactivate DB
    and [Сповіщення клієнта через SSE / WebSocket]
        RegSvc -) WebUI: pushEvent("REGISTRATION_CONFIRMED", {regId: 101, status: 'PAID'})
        activate WebUI
        WebUI -->> Player: showNotification("Оплату підтверджено! Вас включено до турніру")
        deactivate WebUI
    end

    RegSvc -->> PaySvc: paymentConfirmed(status='PAID')
    deactivate RegSvc
    PaySvc -->> RegCtrl: webhookProcessed
    deactivate PaySvc
    RegCtrl -->> Gateway: 200 OK (Webhook Acknowledged)
    deactivate RegCtrl
```

---

## 2. Сценарій 2 (Alternative / Retry): `model/model-02-payment-failure.mmd`

```mermaid
sequenceDiagram
    autonumber
    actor Player as Player (Гравець)
    participant WebUI as Frontend (WebUI)
    participant Auth as AuthMiddleware
    participant RegCtrl as RegistrationController
    participant RegSvc as RegistrationService
    participant PaySvc as PaymentService
    actor Gateway as Payment Gateway
    participant DB as Database (PostgreSQL)

    Note over Player, Gateway: Первинна спроба списання у шлюзі зазнає невдачі

    Player ->> Gateway: submitPayment(cardDetails, session_id='cs_101')
    activate Gateway
    Gateway ->> Gateway: processCharge(500.00) -> FAILED (Card Declined)
    Gateway -) RegCtrl: POST /api/webhooks/payment<br/>Header: Stripe-Signature<br/>Payload: {event: 'charge.failed', reg_id: 101, reason: 'CARD_DECLINED'}
    activate RegCtrl
    RegCtrl ->> PaySvc: handleWebhook(payload, signature)
    activate PaySvc
    PaySvc ->> PaySvc: verifyHmacSignature(payload, signature, secret)
    PaySvc ->> DB: SELECT payment_status, registered_at FROM tournament_registration WHERE tournament_registration_id = 101
    activate DB
    DB -->> PaySvc: status = 'PENDING', registered_at
    deactivate DB
    Note over PaySvc, DB: Заявка залишається 'PENDING' (REQ-06) — діє 15-хвилинне вікно резервування
    PaySvc -->> RegCtrl: webhookAcknowledged
    deactivate PaySvc
    RegCtrl -->> Gateway: 200 OK (Webhook Acknowledged)
    deactivate RegCtrl

    Gateway -->> WebUI: redirect(failure_url?reg_id=101&reason=CARD_DECLINED)
    deactivate Gateway
    activate WebUI
    WebUI -->> Player: showPaymentError("Картку відхилено. Бажаєте спробувати знову?")
    deactivate WebUI

    loop [поки attempts < 3 та now() <= min(deadline, registered_at + 15 min)]
        Player ->> WebUI: clickRetryPayment(registrationId=101)
        activate WebUI
        WebUI ->> Auth: POST /api/registrations/101/retry-payment<br/>Header: Authorization: Bearer jwt_token
        activate Auth
        Auth ->> Auth: verifyJwtAndRole(jwt)
        Auth ->> RegCtrl: forward(req, AuthContext(userId=10, playerId=42))
        activate RegCtrl
        RegCtrl ->> PaySvc: createRetrySession(registrationId=101, playerId=42)
        activate PaySvc
        PaySvc ->> DB: SELECT tr.payment_status, tr.registered_at, t.registration_deadline, t.entry_fee<br/>FROM tournament_registration tr JOIN tournament t ON tr.tournament_id = t.tournament_id<br/>WHERE tr.tournament_registration_id = 101
        activate DB
        DB -->> PaySvc: status='PENDING', registered_at, deadline, fee=500.00
        deactivate DB
        PaySvc ->> Gateway: POST /v1/checkout/sessions<br/>{amount: 500.00, reg_id: 101, expires_in: min(15m, deadline)}
        activate Gateway
        Gateway -->> PaySvc: 200 OK {session_id: 'cs_101_retry', checkout_url}
        deactivate Gateway
        PaySvc -->> RegCtrl: RetrySession(checkoutUrl='https://pay.gateway.com/pay/cs_101_retry')
        deactivate PaySvc
        RegCtrl -->> Auth: 200 OK {checkout_url}
        deactivate RegCtrl
        Auth -->> WebUI: 200 OK (checkout_url)
        deactivate Auth
        WebUI -->> Player: redirect(newCheckoutUrl)
        deactivate WebUI

        alt [успіх повторного списання]
            Player ->> Gateway: submitPayment(newCardDetails, cs_101_retry)
            activate Gateway
            Gateway ->> Gateway: processCharge(500.00) -> SUCCESS
            Gateway -) RegCtrl: POST /api/webhooks/payment<br/>Payload: {event: 'charge.successful', reg_id: 101}
            deactivate Gateway
            activate RegCtrl
            RegCtrl ->> RegSvc: confirmPayment(registrationId=101)
            activate RegSvc
            RegSvc ->> DB: UPDATE tournament_registration<br/>SET payment_status = 'PAID'<br/>WHERE tournament_registration_id = 101 AND payment_status = 'PENDING'
            activate DB
            DB -->> RegSvc: updated (1 row)
            deactivate DB
            RegSvc -) WebUI: pushEvent("REGISTRATION_CONFIRMED", {regId: 101, status: 'PAID'})
            activate WebUI
            WebUI -->> Player: showNotification("Оплату успішно підтверджено!")
            deactivate WebUI
            RegSvc -->> RegCtrl: paymentConfirmed
            deactivate RegSvc
            RegCtrl -->> Gateway: 200 OK
            deactivate RegCtrl
        else [тайм-аут сесії або вичерпання спроб]
            activate RegCtrl
            RegCtrl -) WebUI: pushEvent("SESSION_EXPIRED", {regId: 101})
            activate WebUI
            WebUI -->> Player: showPaymentError("Час сесії оплати вичерпано. Бронь знято, статус заявки залишається PENDING.")
            deactivate WebUI
            Note over RegCtrl, DB: Статус залишається 'PENDING' (без CANCELLED), 15-хв TTL спливає, слот стає доступним іншим
            deactivate RegCtrl
        end
    end
```

---

## 3. Сценарій 3 (Exception / Validation): `model/model-03-validation-error.mmd`

```mermaid
sequenceDiagram
    autonumber
    actor Player as Player (Гравець)
    participant WebUI as Frontend (WebUI)
    participant Auth as AuthMiddleware
    participant RegCtrl as RegistrationController
    participant RegSvc as RegistrationService
    participant PaySvc as PaymentService
    participant DB as Database (PostgreSQL)

    Player ->> WebUI: clickRegister(tournamentId=5)
    activate WebUI
    WebUI ->> Auth: POST /api/tournaments/5/registrations<br/>Header: Authorization: Bearer jwt_token
    activate Auth
    Auth ->> Auth: verifyJwtAndRole(jwt)<br/>[claims: user_id=10, role=USER, player_id=42]
    Auth ->> RegCtrl: forward(req, AuthContext(userId=10, playerId=42))
    activate RegCtrl
    RegCtrl ->> RegSvc: registerPlayer(playerId=42, tournamentId=5)
    activate RegSvc

    RegSvc ->> DB: SELECT status, registration_deadline, max_participants FROM tournament WHERE tournament_id = 5
    activate DB
    DB -->> RegSvc: tournament(status='REGISTRATION_OPEN', deadline, max=32)
    deactivate DB

    RegSvc ->> DB: SELECT COUNT(*) as active_count,<br/>(SELECT payment_status, registered_at FROM tournament_registration WHERE tournament_id = 5 AND player_id = 42) as existing_reg<br/>FROM tournament_registration WHERE tournament_id = 5 AND (payment_status = 'PAID' OR (payment_status = 'PENDING' AND registered_at + INTERVAL '15 minutes' > NOW()))
    activate DB
    DB -->> RegSvc: queryResult(active_count, existing_reg)
    deactivate DB

    alt [now() > registration_deadline]
        RegSvc -->> RegCtrl: throw RegistrationClosedException()
        RegCtrl -->> Auth: 403 Forbidden {error: "REGISTRATION_CLOSED", message: "Дедлайн минув"}
        Auth -->> WebUI: 403 Forbidden (REGISTRATION_CLOSED)
        WebUI -->> Player: showErrorMessage("Відхилено: час реєстрації на турнір минув")
    else [active_count >= max_participants AND (existing_reg is NULL OR (existing_reg.status == 'PENDING' AND existing_reg.registered_at + 15m < now()))]
        RegSvc -->> RegCtrl: throw TournamentQuotaExceededException()
        RegCtrl -->> Auth: 409 Conflict {error: "QUOTA_EXCEEDED", message: "Всі місця зайнято"}
        Auth -->> WebUI: 409 Conflict (QUOTA_EXCEEDED)
        WebUI -->> Player: showErrorMessage("Відхилено: всі 32 місця на турнір заповнені (або бронь спливла)")
    else [existing_reg.status == 'PAID']
        RegSvc -->> RegCtrl: throw DuplicateRegistrationException()
        RegCtrl -->> Auth: 409 Conflict {error: "DUPLICATE_PAID", message: "Участь вже оплачено"}
        Auth -->> WebUI: 409 Conflict (DUPLICATE_PAID)
        WebUI -->> Player: showErrorMessage("Відхилено: ви вже зареєстровані та оплатили участь")
    else [existing_reg.status == 'PENDING']
        Note over RegSvc, DB: Заявка у PENDING: відновлення сесії без 409 помилки
        opt [existing_reg.registered_at + 15m < now()]
            RegSvc ->> DB: UPDATE tournament_registration SET registered_at = NOW() WHERE tournament_registration_id = 101
            activate DB
            DB -->> RegSvc: updated (1 row)
            deactivate DB
        end
        RegSvc ->> PaySvc: getCheckoutSession(regId=101)
        activate PaySvc
        PaySvc -->> RegSvc: checkoutUrl
        deactivate PaySvc
        RegSvc -->> RegCtrl: ResumeResult(status='PENDING', checkoutUrl)
        RegCtrl -->> Auth: 200 OK {status: 'PENDING', checkout_url}
        Auth -->> WebUI: 200 OK (checkout_url)
        WebUI -->> Player: redirect(checkoutUrl)
    end

    Note over RegSvc, DB: Інваріант цілісності: новий INSERT не викликається при відмові

    deactivate RegSvc
    deactivate RegCtrl
    deactivate Auth
    deactivate WebUI
```
