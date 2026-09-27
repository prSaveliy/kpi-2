# spec.md - Реалізація (сутності, атрибути, зв'язки)
Цей файл містить інформацію для реалізації ER діаграми CRM системи.

## Сутності та атрибути

### User
- **User** - узагальнююча таблиця для акторів: Player та Director (директор турніру). Director не має власних полів,
тому це буде реалізовано через через зберігання турнірів як атрибуту в таблиці User і зовнішнього
ключа User'а в таблиці Tournament

Атрибути: 
 - user_id Int autoincrement PK
 - first_name String
 - last_name String
 - email String unique
 - password String
 - date_of_birth Date 

### Player
- **Player** - профіль гравця. 

Атрибути:
 - player_id Int autoincrement PK
 - user_id Int unique (FK)
 - rating Int
 - sports_title SportsTitle <!-- FIRST_CATEGORY, CANDIDATE_MASTER_OF_SPORTS, MASTER_OF_SPORTS etc.  --> 

### Club
- **Club** - Модель більярдного клубу. з ним будуть пов'язані столи та турніри.

Атрибути:
 - club_id Int autoincrement PK
 - name String
 - city String
 - address String

### Tournament
- **Tournament** - модель турніру

Атрибути:
 - tournament_id Int autoincrement PK
 - club_id Int FK
 - director_id Int FK
 - name String
 - entry_fee Decimal
 - prize_pool Decimal
 - discipline Discipline <!--POOL_9, POOL_8, COMBINED_PYRAMID, SNOOKER etc.-->
 - format Format <!--SINGLE_ELIMINATION, DOUBLE_ELIMINATION, ROUND_ROBIN-->
 - status TournamentStatus <!--ANNOUNCED, REGISTRATION_OPEN, ONGOING, FINISHED-->
 - start_date Datetime
 - end_date Datetime

### Tournament_registration
- **Tournament_registration** - асоціативна сутність між гравцем та турніром.

Атрибути:
 - tournament_registration_id Int autoincrement PK
 - tournament_id Int FK
 - player_id Int FK nullable
 - team_id Int FK nullable
 - seed_number Int
 - placement Int
 - payment_status PaymentStatus <!--PENDING, PAID, REFUNDED-->
 - prize_won Decimal
 - registered_at Datetime

### Match 
 - **Match** - сутність матчу.

Атрибути:
 - match_id Int autoincrement PK
 - tournament_id Int FK
 - table_id Int FK nullable
 - stage MatchStage <!--QUALIFICATION, ROUND_OF_16, QUARTER_FINAL, SEMI_FINAL, FINAL-->
 - race_to Int
 - time_limit_minutes Int
 - status MatchStatus <!--SCHEDULED, IN_PROGRESS, FINISHED, CANCELLED-->
 - scheduled_at Datetime
 - started_at Datetime
 - finished_at Datetime

### Match_participant
- **Match_participant** - асоціативна сутність учасника / сторони матчу.

Атрибути:
 - match_participant_id Int autoincrement PK
 - match_id Int FK
 - player_id Int FK nullable
 - team_id Int FK nullable
 - score Int
 - side Int <!--1 або 2-->
 - is_winner Boolean

## TEAM
- **Team** - сутність команди.
Атрибути:
 - team_id Int autoincrement PK
 - name String
 - created_at Datetime

## Billiard_table
- **Billard-table** - сутністсь більярдного столу
Атрибути:
 - table_id int autoincrement PK
 - club_id int FK
 - table_number int
 - table_type TableType <!--PYRAMID_12FT, PYRAMID_10FT, POOL_9FT, SNOOKER_12FT etc.--> 
 - table_status TableStatus <!--OCCUPIED, AVAILABLE, RESERVED, MAINTENANCE -->


## Зв'язки

- User - Player (1:0..1)
- User - Tournament (1:0..N). можна бути директором багатьох турнірів. турнір має одного директора
- Club - Billiard_table (1:N)
- CLub - Tournament (1:0..N)
- Tournament - Tournament_registration (1:0..N)
- Player - Tournament_registration (0..1:0..N)
- Team - Tournament_registration (0..1:0..N)
- Team - Player (M:1..N)
- Match - Match_participant (1:2..N)
- Player - Match_participant (0..1:0..N)
- Team - Match_participant (0..1:0..N)
- Tournament -  Match (1:0..N)
- Billiard_table - Match (0..1:0..N)

## Критерії прийняття (Acceptance Criteria)

1. **Валідність синтаксису**: Mermaid-код ER-діаграми успішно валідується та рендериться без синтаксичних помилок.
2. **Артефакт - діаграма**: Наявність актуальної концептуальної ER-діаграми у файлі `model.mmd` та експортованого графічного зображення.
3. **Коміти за правилами з AGENTS.md**: Дотримання конвенції атомарних семантичних комітів (`spec:`, `model:`, `ai:`, `audit:`, `docs(adr):`).
4. **Слід промптів / виходу AI / фідбеку**: Збереження прозорого аудиторського сліду розробки в каталозі `ai/` (промпти, сирі генерації, аудит-фідбек).
5. **3NF (Третя нормальна форма)**: Концептуальна модель відповідає вимогам 3NF - відсутні транзитивні та часткові функціональні залежності.
6. **Чисті M:N без асоціативних сутностей**: Зв'язки типу багато-до-багатьох, які не мають власних бізнес-атрибутів, моделюються як прямі зв'язки сутностей без введення технічних з'єднувальних таблиць. Асоціативні сутності використовуються виключно тоді, коли сам факт зв'язку несе бізнес-стан.
7. **Узгоджені типи ідентифікаторів**: Первинні та зовнішні ключі типізовані, уніфіковані за стандартом `<entity>_id` (зокрема `table_id` для `Billiard_table`).
8. **Назви полів у діаграмі й spec не розходяться**: Повна тотожність імен сутностей, атрибутів та кардинальностей між специфікацією та схемою.
