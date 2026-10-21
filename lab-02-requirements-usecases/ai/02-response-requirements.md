# Відповідь ШІ 02: Точкове оновлення вимог та Use Case діаграми (Ітерація 2)

Виконано точкові зміни у `model.mmd` (`+10/-10` рядків) та `requirements.md` (`+6/-6` рядків) строго за 5 пунктами інженерного аудиту `ai/02-feedback-audit.md`:

1. **`0377bf9` (`UC-07 -> UC-05`):** у `model.mmd` додано `UC07 -.->|"«extend»"| UC05` для скасування неоплаченої командної заявки капітаном (`Captain`).
2. **`fcac044` (`include`/`extend` та шеврони Mermaid):** у `model.mmd` видалено `UC02 -> UC01`, змінено зв'язок до `UC06` на `«include»`, а всі `<<...>>` замінено на `«include»`, `«extend»`, `«generalize»`.
3. **`1e233bc` (`seed_number`):** у `REQ-03` та першому Gherkin-сценарії вказано ініціалізацію `seed_number = NULL` при реєстрації; у `REQ-09` уточнено збереження `NULL` для форматів без посіву (*round-robin*).
4. **`909309c` (схема БД Лаб №1):** у `REQ-03` та Gherkin замінено `registration` на `tournament_registration` (`player_id` XOR `team_id`), а в `REQ-08` та Gherkin замінено `match.score` на `match_participant` (`frames_won` XOR `points`).
5. **`c281790` (EARS тестованість):** у `REQ-03` додано умови `'REGISTRATION_OPEN'` та `< max_participants`; у `REQ-10` задано час відгуку `≤ 500 мс` (p95) для `щонайменше 1000 одночасних користувачів` та `HTTP 429` при перевантаженні.
