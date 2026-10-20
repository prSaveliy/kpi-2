# spec.md - Специфікація ER-моделі (сутності, атрибути, зв'язки)
Цей файл містить формалізовану специфікацію реляційної ER-моделі бази даних CRM-системи більярдного клубу та турнірів.

## Сутності та атрибути

### users
Узагальнююча сутність облікових записів акторів системи (гравці, директори турнірів, адміністратори).
- `user_id` int autoincrement PK
- `first_name` varchar
- `last_name` varchar
- `email` varchar unique
- `password_hash` varchar
- `date_of_birth` date
- `created_at` timestamptz
- `deleted_at` timestamptz nullable

### player
Спортивний профіль гравця (розширення базового акаунту `users`).
- `player_id` int autoincrement PK
- `person_id` int unique FK (посилання на `users.user_id`, 1:1)
- `rating` int nullable
- `sports_title` sports_title ENUM
- `created_at` timestamptz

### club
Більярдний клуб (локація проведення турнірів та розміщення столів).
- `club_id` int autoincrement PK
- `club_name` varchar
- `city` varchar
- `address` varchar
- `created_at` timestamptz

### billiard_table
Більярдний стіл у клубі.
- `table_id` int autoincrement PK
- `club_id` int FK
- `table_number` smallint (unique разом із `club_id`)
- `table_type` table_type ENUM
- `table_status` table_status ENUM
- `created_at` timestamptz

### team
Команда для участі у командних турнірах із визначеним капітаном.
- `team_id` int autoincrement PK
- `team_name` varchar unique
- `captain_id` int FK (посилання на `player.player_id`)
- `created_at` timestamptz

### team_member
Асоціативна сутність членства гравців у командах (розкриття зв'язку M:N).
- `team_id` int PK,FK
- `player_id` int PK,FK
- `joined_at` timestamptz

### tournament
Турнір з більярдного спорту.
- `tournament_id` int autoincrement PK
- `club_id` int FK
- `director_id` int FK (посилання на `users.user_id`)
- `tournament_name` varchar
- `max_participants` smallint
- `entry_fee` numeric
- `prize_pool` numeric
- `discipline` discipline ENUM
- `format` format ENUM
- `tournament_status` tournament_status ENUM
- `registration_deadline` timestamptz
- `start_date` timestamptz
- `end_date` timestamptz nullable
- `created_at` timestamptz

### tournament_registration
Асоціативна сутність заявки на участь у турнірі (одиночній або командній).
- `tournament_registration_id` int autoincrement PK
- `tournament_id` int FK
- `player_id` int FK nullable (XOR `team_id`)
- `team_id` int FK nullable (XOR `player_id`)
- `seed_number` smallint nullable
- `payment_status` payment_status ENUM
- `final_placement` smallint nullable
- `prize_won` numeric
- `registered_at` timestamptz

### match
Матч у межах турніру.
- `match_id` int autoincrement PK
- `tournament_id` int FK
- `table_id` int FK nullable
- `stage` match_stage ENUM
- `race_to_frames` smallint nullable (XOR `race_to_points`)
- `race_to_points` smallint nullable (XOR `race_to_frames`)
- `time_limit_minutes` smallint nullable
- `match_status` match_status ENUM
- `scheduled_at` timestamptz nullable
- `started_at` timestamptz nullable
- `finished_at` timestamptz nullable
- `created_at` timestamptz

### match_participant
Асоціативна сутність сторони / учасника зустрічі.
- `match_participant_id` int autoincrement PK
- `match_id` int FK
- `player_id` int FK nullable (XOR `team_id`)
- `team_id` int FK nullable (XOR `player_id`)
- `frames_won` smallint nullable (XOR `points`)
- `points` smallint nullable (XOR `frames_won`)
- `side` participant_side ENUM
- `is_winner` boolean
- `created_at` timestamptz

## Зв'язки (Relationships)

- `users` - `player` (1:0..1) — профіль спортсмена (`person_id`)
- `users` - `tournament` (1:0..N) — директор турніру (`director_id`)
- `club` - `billiard_table` (1:1..N) — столи клубу (`club_id`)
- `club` - `tournament` (1:0..N) — проведення турніру в клубі (`club_id`)
- `tournament` - `tournament_registration` (1:0..N) — реєстрації учасників (`tournament_id`)
- `tournament` - `match` (1:0..N) — матчі турніру (`tournament_id`)
- `billiard_table` - `match` (0..1:0..N) — призначення столу на матч (`table_id`)
- `player` - `tournament_registration` (0..1:0..N) — реєстрація індивідуального гравця (`player_id`)
- `team` - `tournament_registration` (0..1:0..N) — реєстрація команди (`team_id`)
- `player` - `team` (1:0..N) — капітан команди (`captain_id`)
- `team` - `team_member` (1:1..N) — склад команди (`team_id`)
- `player` - `team_member` (1:0..N) — членство гравця у командах (`player_id`)
- `match` - `match_participant` (1:2..N) — сторони матчу (`match_id`)
- `player` - `match_participant` (0..1:0..N) — участь індивідуального гравця (`player_id`)
- `team` - `match_participant` (0..1:0..N) — участь команди (`team_id`)

## Критерії прийняття (Acceptance Criteria)

1. **Валідність синтаксису**: Mermaid-код ER-діаграми успішно валідується та рендериться без синтаксичних помилок.
2. **Артефакт - діаграма**: Наявність актуальної ER-діаграми у файлі `model.mmd` та експортованого графічного зображення `render.png`.
3. **Коміти за правилами з AGENTS.md**: Дотримання конвенції атомарних семантичних комітів (`spec:`, `model:`, `ai:`, `audit:`, `docs(adr):`, `render:`).
4. **Слід промптів / виходу AI / фідбеку**: Збереження прозорого аудиторського сліду розробки в каталозі `ai/` (промпти, сирі генерації, аудит-фідбек).
5. **3NF (Третя нормальна форма)**: Фізична модель відповідає вимогам 3NF — відсутні транзитивні та часткові функціональні залежності.
6. **Фізична реалізація зв'язку M:N**: Зв'язок `team` – `player` фізично розкривається через таблицю `team_member` зі складеним первинним ключем `(team_id, player_id)` та атрибутом `joined_at`; таблиця `team` містить зовнішній ключ `captain_id FK -> player`.
7. **Узгоджені типи ідентифікаторів та полів**: Первинні та зовнішні ключі типізовані, уніфіковані за стандартом `<entity>_id` (з урахуванням `person_id` як FK до `users(user_id)`).
8. **Повна тотожність імен у діаграмі та spec**: Повна узгодженість імен таблиць, атрибутів, перелічуваних типів (ENUM) та кардинальностей між специфікацією та схемою.
