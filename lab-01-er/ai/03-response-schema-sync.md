# Відповідь моделі: Синхронізація ER-моделі з реляційною схемою PostgreSQL (Ітерація 3)

## Огляд змін третьої ітерації
У цій ітерації модель повністю синхронізована з фізичною схемою PostgreSQL, розробленою в межах курсу баз даних:
1. **Розкриття M:N через `team_member`**:
   - Зв'язок між `team` та `player` реалізовано через таблицю-з'єднання `team_member` зі складеним ключем `(team_id, player_id)` та міткою часу приєднання `joined_at`.
   - Додано відношення капітанства: `player ||--o{ team : "captains (captain_id)"`.
2. **Уточнення ідентифікаторів та полів**:
   - `player.person_id` посилається на `users.user_id` (зв'язок 1:1 зі збереженням SSOT та 3NF).
   - Атрибути імен сутностей перейменовано на недвозначні: `club_name`, `team_name`, `tournament_name`.
   - Поле `password` замінено на `password_hash` (безпека збереження облікових записів).
3. **Диференціація правил більярдних дисциплін**:
   - `race_to_frames` (для пулу, снукеру, піраміди) та `race_to_points` (для карамболю/straight pool) у `match`.
   - `frames_won` та `points` у `match_participant`.
   - `participant_side` ENUM для позиції за столом.
4. **Аудит та життєвий цикл**:
   - Додано `created_at timestamptz` до всіх таблиць.
   - Додано `deleted_at timestamptz nullable` у таблицю `users` для забезпечення м'якого видалення (soft delete).

---

## Mermaid ER Діаграма

```mermaid
erDiagram
    users ||--o| player : "has profile (person_id)"
    users ||--o{ tournament : "directs (director_id)"
    club ||--|{ billiard_table : "owns (club_id)"
    club ||--o{ tournament : "hosts (club_id)"
    tournament ||--o{ tournament_registration : "has (tournament_id)"
    tournament ||--o{ match : "organizes (tournament_id)"
    billiard_table |o--o{ match : "assigned to (table_id)"
    player |o--o{ tournament_registration : "registers (player_id)"
    team |o--o{ tournament_registration : "registers (team_id)"
    player ||--o{ team : "captains (captain_id)"
    team ||--|{ team_member : "contains (team_id)"
    player ||--o{ team_member : "belongs to (player_id)"
    match ||--|{ match_participant : "features (match_id)"
    player |o--o{ match_participant : "participates as (player_id)"
    team |o--o{ match_participant : "participates as (team_id)"

    users {
        int user_id PK
        varchar first_name
        varchar last_name
        varchar email "UK"
        varchar password_hash
        date date_of_birth
        timestamptz created_at
        timestamptz deleted_at "nullable"
    }

    player {
        int player_id PK
        int person_id FK,UK "1:1 with users"
        int rating "nullable"
        sports_title sports_title
        timestamptz created_at
    }

    club {
        int club_id PK
        varchar club_name
        varchar city
        varchar address
        timestamptz created_at
    }

    billiard_table {
        int table_id PK
        int club_id FK
        smallint table_number "UK with club_id"
        table_type table_type
        table_status table_status
        timestamptz created_at
    }

    team {
        int team_id PK
        varchar team_name "UK"
        int captain_id FK "references player"
        timestamptz created_at
    }

    team_member {
        int team_id PK,FK
        int player_id PK,FK
        timestamptz joined_at
    }

    tournament {
        int tournament_id PK
        int club_id FK
        int director_id FK "references users"
        varchar tournament_name
        smallint max_participants
        numeric entry_fee
        numeric prize_pool
        discipline discipline
        format format
        tournament_status tournament_status
        timestamptz registration_deadline
        timestamptz start_date
        timestamptz end_date "nullable"
        timestamptz created_at
    }

    tournament_registration {
        int tournament_registration_id PK
        int tournament_id FK "UK with player_id, team_id, seed"
        int player_id FK "nullable, XOR team_id"
        int team_id FK "nullable, XOR player_id"
        smallint seed_number "nullable"
        payment_status payment_status
        smallint final_placement "nullable"
        numeric prize_won
        timestamptz registered_at
    }

    match {
        int match_id PK
        int tournament_id FK
        int table_id FK "nullable"
        match_stage stage
        smallint race_to_frames "nullable, XOR race_to_points"
        smallint race_to_points "nullable, XOR race_to_frames"
        smallint time_limit_minutes "nullable"
        match_status match_status
        timestamptz scheduled_at "nullable"
        timestamptz started_at "nullable"
        timestamptz finished_at "nullable"
        timestamptz created_at
    }

    match_participant {
        int match_participant_id PK
        int match_id FK "UK with side, player_id, team_id"
        int player_id FK "nullable, XOR team_id"
        int team_id FK "nullable, XOR player_id"
        smallint frames_won "nullable, XOR points"
        smallint points "nullable, XOR frames_won"
        participant_side side
        boolean is_winner
        timestamptz created_at
    }
```

## Відповідність критеріям прийняття (Acceptance Criteria)
1. **Валідність синтаксису**: Mermaid-код валідний, успішно проходить валідацію.
2. **Артефакт - діаграма**: Файл `model.mmd` та рендер `render.png` повністю відповідають реляційній схемі.
3. **Семантичні коміти**: Робота виконується за конвенцією Conventional Commits.
4. **Аудиторський слід**: Зафіксовано промпт `ai/03-prompt-schema-sync.md` та результат `ai/03-response-schema-sync.md`.
5. **3NF**: Схема нормалізована до 3NF, зв'язок M:N коректно розкрито через сполучну таблицю `team_member`.
6. **Уніфікація PK/FK**: Усі первинні та зовнішні ключі узгоджені зі схемою DDL.
7. **Повна тотожність**: Імена сутностей, атрибутів та зв'язків тотожні `spec.md` та `schema.mmd`.
