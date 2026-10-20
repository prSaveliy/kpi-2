# Завдання: Синхронізація ER-моделі з оновленою схемою реляційної БД (Ітерація 3)

## Мета
Оновити та синхронізувати концептуальну ER-модель (`model.mmd`) з уточненою фізичною схемою бази даних, спроектованою у межах лабораторної роботи №2 з курсу «Бази даних» (`~/forks/db-intro-course/labs/2 - DDL/schema.png` та `schema.mmd`), згідно з оновленим `spec.md`.

## Вхідні дані
- Оновлена технічна специфікація: `spec.md` (коміт `ceb254b`)
- Фізична схема та DDL: `~/forks/db-intro-course/labs/2 - DDL/schema.png`, `schema.mmd`, `create_tables.sql`
- Звіт розробки DDL: `~/forks/db-intro-course/labs/2 - DDL/lab_2_report.md`
- Правила взаємодії та стандарти проекту: `AGENTS.md`

## Вимоги до синхронізації схеми
1. **Фізичне розкриття зв'язку M:N Team – Player**:
   - Додати асоціативну таблицю `team_member` зі складеним первинним ключем `(team_id, player_id)` та атрибутом `joined_at`.
   - Додати зовнішній ключ `captain_id` у таблицю `team`, що посилається на `player(player_id)`.
2. **Уніфікація назв сутностей та ідентифікаторів**:
   - Використати snake_case та точні імена таблиць: `users`, `player`, `club`, `billiard_table`, `team`, `team_member`, `tournament`, `tournament_registration`, `match`, `match_participant`.
   - У таблиці `player` зовнішній ключ до `users` іменується як `person_id FK,UK`.
   - У таблицях `club`, `team`, `tournament` назви колонок уніфіковано: `club_name`, `team_name`, `tournament_name`.
3. **Диференціація метрик гри та деталізація атрибутів**:
   - У сутності `match`: розділити `race_to` на взаємовиключні стовпці `race_to_frames` та `race_to_points`.
   - У сутності `match_participant`: розділити загальний рахунок `score` на `frames_won` та `points`, типізувати сторону `side` перелічуваним типом `participant_side`.
   - У сутності `tournament`: додати атрибути `max_participants`, `registration_deadline`.
   - Додати системні атрибути аудиту та життєвого циклу: `created_at` для сутностей, `deleted_at` для `users`.
4. **Формат виходу**:
   - Валідний код Mermaid `erDiagram`.
   - Повний опис відповідності критеріям прийняття.
