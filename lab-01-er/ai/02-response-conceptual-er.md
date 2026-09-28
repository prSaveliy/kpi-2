# Відповідь моделі: Регенерована концептуальна ER-модель CRM системи більярдного клубу (Ітерація 2)

## Огляд змін другої ітерації
У цій ітерації модель повністю синхронізована з оновленими вимогами `spec.md` та усунено всі зауваження з `ai/02-feedback-audit.md`:
1. **Зв'язок Team – Player (M:1..N)**: зв'язок оновлено до `Team }o--|{ Player : "consists of"`. Це гарантує правило домену: кожна команда повинна містити щонайменше одного гравця, гравець може входити до довільної кількості команд або грати сольно, зв'язок прямий M:N без асоціативної сутності відповідно до 3NF.
2. **PK сутності Billiard_table**: підтверджено відповідність `table_id PK` загальній системній конвенції `<entity>_id`.
3. **Повна синхронізація атрибутів**:
   - `Tournament`: додано `Format format`, `TournamentStatus status`, `datetime start_date`, `datetime end_date`.
   - `Tournament_registration`: додано `PaymentStatus payment_status`, `decimal prize_won`.
   - `Match`: додано `MatchStage stage`, `MatchStatus status`, `datetime scheduled_at`.
   - `Match_participant`: додано `int side`.

---

## Mermaid ER Діаграма

```mermaid
erDiagram
    User ||--o| Player : "has profile"
    User ||--o{ Tournament : "directs"
    Club ||--|{ Billiard_table : "owns"
    Club ||--o{ Tournament : "hosts"
    Tournament ||--o{ Tournament_registration : "has"
    Tournament ||--o{ Match : "organizes"
    Player |o--o{ Tournament_registration : "registers"
    Team |o--o{ Tournament_registration : "registers"
    Team }o--|{ Player : "consists of"
    Match ||--|{ Match_participant : "features"
    Player |o--o{ Match_participant : "participates as"
    Team |o--o{ Match_participant : "participates as"
    Billiard_table |o--o{ Match : "assigned to"

    User {
        int user_id PK
        string first_name
        string last_name
        string email "unique"
        string password
        date date_of_birth
    }

    Player {
        int player_id PK
        int user_id FK "unique"
        int rating
        SportsTitle sports_title
    }

    Club {
        int club_id PK
        string name
        string city
        string address
    }

    Tournament {
        int tournament_id PK
        int club_id FK
        int director_id FK "references User"
        string name
        decimal entry_fee
        decimal prize_pool
        Discipline discipline
        Format format
        TournamentStatus status
        datetime start_date
        datetime end_date
    }

    Tournament_registration {
        int tournament_registration_id PK
        int tournament_id FK
        int player_id FK "nullable"
        int team_id FK "nullable"
        int seed_number
        int placement
        PaymentStatus payment_status
        decimal prize_won
        datetime registered_at
    }

    Match {
        int match_id PK
        int tournament_id FK
        int table_id FK "nullable"
        MatchStage stage
        int race_to
        int time_limit_minutes
        MatchStatus status
        datetime scheduled_at
        datetime started_at
        datetime finished_at
    }

    Team {
        int team_id PK
        string name
        datetime created_at
    }

    Billiard_table {
        int table_id PK
        int club_id FK
        int table_number
        TableType table_type
        TableStatus table_status
    }

    Match_participant {
        int match_participant_id PK
        int match_id FK
        int player_id FK "nullable"
        int team_id FK "nullable"
        int score
        int side
        boolean is_winner
    }
```

## Відповідність критеріям прийняття (Acceptance Criteria)
1. **Валідність синтаксису**: Код Mermaid ER відповідає офіційній специфікації Mermaid.
2. **Артефакт - діаграма**: Файл `model.mmd` містить актуальну модель.
3. **Семантичні коміти**: Зміни документуються за правилами `AGENTS.md` (`ai:`, `model:`).
4. **Аудиторський слід**: Зафіксовано промпт `ai/02-prompt-conceptual-er.md` та результат `ai/02-response-conceptual-er.md`.
5. **3NF**: Усі неключові атрибути функціонально залежать виключно від повного первинного ключа.
6. **M:N без зайвих сутностей**: Прямий зв'язок між `Team` та `Player` без введення порожньої таблиці з'єднання.
7. **Уніфікація PK/FK**: Усі ідентифікатори узгоджені (`table_id`, `user_id`, `tournament_id` тощо).
8. **Повна тотожність імен**: Імена сутностей, атрибутів та кардинальності повністю синхронізовані зі `spec.md`.
