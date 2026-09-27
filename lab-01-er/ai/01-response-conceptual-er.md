# Відповідь моделі: Концептуальна ER-модель CRM системи більярдного клубу

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
    Team }o--o{ Player : "consists of"
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
    }

    Tournament_registration {
        int tournament_registration_id PK
        int tournament_id FK
        int player_id FK "nullable"
        int team_id FK "nullable"
        int seed_number
        int placement
        datetime registered_at
    }

    Match {
        int match_id PK
        int tournament_id FK
        int table_id FK "nullable"
        int race_to
        int time_limit_minutes
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
        boolean is_winner
    }
```

## Опис сутностей та зв'язків

1. **User (Користувач)**:
   - Базова сутність для автентифікації та профілю.
   - Зв'язок 1:0..1 з `Player` (користувач може мати профіль гравця).
   - Зв'язок 1:0..N з `Tournament` як директор/організатор (`director_id` посилається на `User`).

2. **Player (Гравець)**:
   - Спортивний профіль із рейтингом та спортивним званням (`SportsTitle`).
   - Унікальний зв'язок 1:1 з `User` (`user_id` FK unique).

3. **Club (Клуб)**:
   - Локація та організаційна база з адресою та назвою.
   - Зв'язок 1:1..N зі столами (`Billiard_table`).
   - Зв'язок 1:0..N із турнірами (`Tournament`).

4. **Billiard_table (Більярдний стіл)**:
   - Стіл з номером, типом дисципліни (`TableType`) та оперативним статусом (`TableStatus`).
   - Прив'язаний до клубу. Зв'язок 0..1:0..N із матчами (`Match`), оскільки стіл призначається на матч або під час проведення турніру.

5. **Tournament (Турнір)**:
   - Змагання з дисципліною (`Discipline`), внеском, призовим фондом, клубом проведення та призначеним директором.
   - Має множину реєстрацій (`Tournament_registration`) та матчів (`Match`).

6. **Tournament_registration (Реєстрація на турнір)**:
   - Асоціативна сутність для реєстрації учасників.
   - Підтримує як індивідуальні турніри (`player_id`), так і командні (`team_id`) через nullable FK.
   - Фіксує посів (`seed_number`) та фінальне місце (`placement`).

7. **Team (Команда)**:
   - Команда для командних змагань.
   - Зв'язок M:N з гравцями (`Player`).
   - Може реєструватися на турніри (`Tournament_registration`) та брати участь у матчах (`Match_participant`).

8. **Match (Матч)**:
   - Зустріч у межах турніру з параметрами регламенту (`race_to`, `time_limit_minutes`, часові мітки).
   - Призначається на вільний стіл (`table_id` nullable).

9. **Match_participant (Учасник матчу)**:
   - Учасник зустрічі (гравець або команда).
   - Зв'язок 1:2..N з матчем (у більярді зазвичай 2 суперники).
   - Зберігає набрані очки (`score`) та ознаку перемоги (`is_winner`) для фіксації суддею.
