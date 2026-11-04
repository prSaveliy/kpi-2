# Відповідь моделі: Генерація Sequence-діаграм для прецеденту онлайн-реєстрації

Відповідно до специфікації `spec.md` згенеровано декларативні моделі Sequence-діаграм у синтаксисі Mermaid для трьох сценаріїв прецеденту `UC-04` (Реєстрація на індивідуальний турнір) з включенням `UC-06` (Оплата внеску).

---

## 1. Сценарій 1 (Happy Path): Успішна реєстрація з онлайн-оплатою (`model/model-01-success.mmd`)

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

    RegSvc ->> DB: SELECT COUNT(*) FROM tournament_registration WHERE tournament_id = 5 AND payment_status != 'CANCELLED'
    activate DB
    DB -->> RegSvc: count = 14
    deactivate DB

    RegSvc ->> DB: INSERT INTO tournament_registration (tournament_id, player_id, team_id, payment_status, seed_number)<br/>VALUES (5, 42, NULL, 'PENDING', NULL) RETURNING tournament_registration_id
    activate DB
    DB -->> RegSvc: tournament_registration_id = 101
    deactivate DB

    opt [entry_fee > 0]
        RegSvc ->> PaySvc: initiateCheckout(regId=101, amount=500.00)
        activate PaySvc
        PaySvc ->> Gateway: POST /v1/checkout/sessions<br/>{amount: 500.00, reg_id: 101, return_url}
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
        RegSvc ->> DB: UPDATE tournament_registration<br/>SET payment_status = 'PAID'<br/>WHERE tournament_registration_id = 101
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

## 2. Сценарій 2 (Alternative / Retry): Помилка списання коштів та повторна спроба (`model/model-02-payment-failure.mmd`)

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
    PaySvc ->> DB: SELECT payment_status FROM tournament_registration WHERE tournament_registration_id = 101
    activate DB
    DB -->> PaySvc: status = 'PENDING'
    deactivate DB
    Note over PaySvc, DB: Заявка залишається 'PENDING', щоб дати можливість повторити платіж до дедлайну
    PaySvc -->> RegCtrl: webhookAcknowledged
    deactivate PaySvc
    RegCtrl -->> Gateway: 200 OK (Webhook Acknowledged)
    deactivate RegCtrl

    Gateway -->> WebUI: redirect(failure_url?reg_id=101&reason=CARD_DECLINED)
    deactivate Gateway
    activate WebUI
    WebUI -->> Player: showPaymentError("Картку відхилено. Бажаєте спробувати знову?")
    deactivate WebUI

    loop [поки attempts < 3 та now() <= registration_deadline]
        Player ->> WebUI: clickRetryPayment(registrationId=101)
        activate WebUI
        WebUI ->> Auth: POST /api/registrations/101/retry-payment<br/>Header: Authorization: Bearer jwt_token
        activate Auth
        Auth ->> Auth: verifyJwtAndRole(jwt)
        Auth ->> RegCtrl: forward(req, AuthContext(userId=10, playerId=42))
        activate RegCtrl
        RegCtrl ->> PaySvc: createRetrySession(registrationId=101, playerId=42)
        activate PaySvc
        PaySvc ->> DB: SELECT tr.payment_status, t.registration_deadline, t.entry_fee FROM tournament_registration tr JOIN tournament t ON tr.tournament_id = t.tournament_id WHERE tr.tournament_registration_id = 101
        activate DB
        DB -->> PaySvc: status='PENDING', deadline_valid=true, fee=500.00
        deactivate DB
        PaySvc ->> Gateway: POST /v1/checkout/sessions<br/>{amount: 500.00, reg_id: 101, retry_attempt: attempt_count}
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
            RegSvc ->> DB: UPDATE tournament_registration<br/>SET payment_status = 'PAID'<br/>WHERE tournament_registration_id = 101
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
        else [тайм-аут або вичерпання ліміту спроб]
            RegCtrl ->> RegSvc: expireRegistration(registrationId=101)
            activate RegCtrl
            activate RegSvc
            RegSvc ->> DB: UPDATE tournament_registration<br/>SET payment_status = 'CANCELLED'<br/>WHERE tournament_registration_id = 101
            activate DB
            DB -->> RegSvc: updated (1 row)
            deactivate DB
            RegSvc -) WebUI: pushEvent("REGISTRATION_EXPIRED", {regId: 101})
            activate WebUI
            WebUI -->> Player: showNotification("Час на оплату вичерпано. Заявку анульовано.")
            deactivate WebUI
            RegSvc -->> RegCtrl: registrationCancelled
            deactivate RegSvc
            deactivate RegCtrl
        end
    end
```

---

## 3. Сценарій 3 (Exception / Validation): Відхилення заявки через порушення інваріантів (`model/model-03-validation-error.mmd`)

```mermaid
sequenceDiagram
    autonumber
    actor Player as Player (Гравець)
    participant WebUI as Frontend (WebUI)
    participant Auth as AuthMiddleware
    participant RegCtrl as RegistrationController
    participant RegSvc as RegistrationService
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

    RegSvc ->> DB: SELECT COUNT(*) as active_count,<br/>EXISTS(SELECT 1 FROM tournament_registration WHERE tournament_id = 5 AND player_id = 42 AND payment_status != 'CANCELLED') as has_registered<br/>FROM tournament_registration WHERE tournament_id = 5 AND payment_status != 'CANCELLED'
    activate DB
    DB -->> RegSvc: queryResult(active_count, has_registered)
    deactivate DB

    alt [now() > registration_deadline]
        RegSvc -->> RegCtrl: throw RegistrationClosedException()
        RegCtrl -->> Auth: 403 Forbidden {error: "REGISTRATION_CLOSED", message: "Дедлайн минув"}
        Auth -->> WebUI: 403 Forbidden (REGISTRATION_CLOSED)
        WebUI -->> Player: showErrorMessage("Відхилено: час реєстрації на турнір минув")
    else [active_count >= max_participants]
        RegSvc -->> RegCtrl: throw TournamentQuotaExceededException()
        RegCtrl -->> Auth: 409 Conflict {error: "QUOTA_EXCEEDED", message: "Всі місця зайнято"}
        Auth -->> WebUI: 409 Conflict (QUOTA_EXCEEDED)
        WebUI -->> Player: showErrorMessage("Відхилено: всі 32 місця на турнір вже заповнені")
    else [has_registered == true]
        RegSvc -->> RegCtrl: throw DuplicateRegistrationException()
        RegCtrl -->> Auth: 409 Conflict {error: "DUPLICATE_REGISTRATION", message: "Заявка вже існує"}
        Auth -->> WebUI: 409 Conflict (DUPLICATE_REGISTRATION)
        WebUI -->> Player: showErrorMessage("Відхилено: ви вже подали заявку на цей турнір")
    end

    Note over RegSvc, DB: Інваріант цілісності: INSERT не викликається, стан бази даних залишається незмінним

    deactivate RegSvc
    deactivate RegCtrl
    deactivate Auth
    deactivate WebUI
```
