# Модель данных PostgreSQL - словарь и ER-диаграмма

Документ описывает целевую структуру БД для сервиса купонов на PostgreSQL. Определяет состав таблиц, типы данных, ключи, стратегию партиционирования и правила маппинга из Oracle. Служит основой для DDL-скриптов и контрактов API.

## 1. Словарь данных

### 1.1. COUPON - купоны

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание | Маппинг из Oracle |
|---|---|---|---|---|---|
| coupon_num | varchar(20) | Да | PK (составной) | Номер купона | VARCHAR2(20) -> varchar(20) |
| promo_id | bigint | Да | FK -> PROMO | Акция, по которой активирован | NUMBER(10) -> bigint |
| status | char(1) | Да | | A (активен), D (погашен), E (ошибка) | CHAR(1) -> char(1) |
| activation_date | timestamp(0) | Да | PK (партиц.) | Дата и время активации | DATE -> timestamp(0) |
| redemption_date | timestamp(0) | Нет | | Дата и время гашения | DATE -> timestamp(0) |

**Партиционирование:** RANGE по `activation_date`.  
**Первичный ключ:** `(coupon_num, activation_date)` - ключ партиционирования должен входить в PK (ограничение PostgreSQL).

### 1.2. COUPON_EVENT - события по купонам

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание | Маппинг из Oracle |
|---|---|---|---|---|---|
| event_id | bigint | Да | PK (составной) | ID события (генерируется) | NUMBER(19) -> bigint |
| event_date | timestamp(0) | Да | PK (партиц.) | Дата и время события | DATE -> timestamp(0) |
| coupon_num | varchar(20) | Да | FK -> COUPON | Номер купона | VARCHAR2(20) -> varchar(20) |
| event_type | varchar(20) | Да | | ACTIVATE, REDEEM, RESTORE, ERROR_RESOLVE | VARCHAR2(10) -> varchar(20) (расширено) |
| promo_id | bigint | Да | FK -> PROMO | Акция операции | NUMBER(10) -> bigint |
| store_code | varchar(10) | Нет | | Код ТТ; NULL для ручной операции | VARCHAR2(10) -> varchar(10) |
| pos_num | int | Нет | | Номер кассы; NULL для ручной | NUMBER(5) -> int |
| is_success | char(1) | Да | | Y (успешно), N (ошибка) | CHAR(1) -> char(1) |
| error_desc | varchar(4000) | Нет | | Описание ошибки (при IS_SUCCESS=N) | VARCHAR2(4000) -> varchar(4000) |

**Партиционирование:** RANGE по `event_date`.  
**Первичный ключ:** `(event_id, event_date)`.

### 1.3. PROMO - описание акций

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание | Маппинг из Oracle |
|---|---|---|---|---|---|
| promo_id | bigint | Да | PK | Номер акции | NUMBER(10) -> bigint |
| promo_guid | uuid | Да | UK | Код акции (GUID) | RAW(16) -> uuid |
| created_date | timestamp(0) | Да | | Дата создания акции | DATE -> timestamp(0) |
| description | varchar(1000) | Нет | | Описание акции | VARCHAR2(1000) -> varchar(1000) |
| load_date | timestamp(0) | Да | | Когда акция пришла | DATE -> timestamp(0) |

### 1.4. PROMO_STORE - акции по ТТ и датам

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание | Маппинг из Oracle |
|---|---|---|---|---|---|
| promo_id | bigint | Да | PK, FK -> PROMO | Номер акции | NUMBER(10) -> bigint |
| store_code | varchar(10) | Да | PK | Код торговой точки | VARCHAR2(10) -> varchar(10) |
| date_from | date | Да | PK | Дата начала действия | DATE -> date |
| date_to | date | Да | | Дата окончания действия | DATE -> date |

### 1.5. POS_KEY - ключи касс

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание | Маппинг из Oracle |
|---|---|---|---|---|---|
| store_code | varchar(10) | Да | PK | Код торговой точки | VARCHAR2(10) -> varchar(10) |
| pos_num | int | Да | PK | Номер кассы | NUMBER(5) -> int |
| valid_from | date | Да | PK | Дата начала действия | DATE -> date |
| secret_key_hash | varchar(128) | Да | | Хеш ключа безопасности | VARCHAR2(128) -> varchar(128) |
| valid_to | date | Нет | | Дата окончания действия | DATE -> date |

*Примечание: при ротации ключей на одну кассу создаётся несколько записей (текущий и следующий ключ) с разными VALID_FROM/VALID_TO.*

### 1.6. MANUAL_OPERATION_TASK - задачи массовых операций (новая)

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание |
|---|---|---|---|---|
| task_id | uuid | Да | PK | Идентификатор задачи |
| operation_type | varchar(20) | Да | | ACTIVATE, REDEEM, RESTORE |
| total_count | int | Да | | Общее количество купонов в загрузке |
| processed_count | int | Да | | Обработано (прогресс) |
| status | varchar(30) | Да | | in_queue, running, completed_success, completed_error |
| result_summary | jsonb | Нет | | Агрегаты (успешно/неуспешно) и перечень ошибок |
| created_at | timestamp(0) | Да | | Дата и время постановки в очередь |
| completed_at | timestamp(0) | Нет | | Дата и время завершения |

### 1.7. IDEMPOTENCY_KEY - ключи идемпотентности (новая)

| Поле | Тип PostgreSQL | Обязательное | Ключ | Описание |
|---|---|---|---|---|
| idempotency_key | varchar(64) | Да | PK | Уникальный ключ операции от кассы |
| coupon_num | varchar(20) | Да | | Номер купона |
| operation_type | varchar(20) | Да | | Тип операции (активация/гашение) |
| result_code | varchar(20) | Да | | Код результата первого запроса |
| created_at | timestamp(0) | Да | | Время создания (для TTL/очистки) |

## 2. ER-диаграмма

```mermaid
erDiagram
    PROMO ||--o{ COUPON : "promo_id"
    PROMO ||--o{ COUPON_EVENT : "promo_id"
    PROMO ||--o{ PROMO_STORE : "promo_id"
    COUPON ||--o{ COUPON_EVENT : "coupon_num"

    COUPON {
        varchar(20) coupon_num PK
        bigint promo_id FK
        char(1) status
        timestamp(0) activation_date PK
        timestamp(0) redemption_date
    }

    COUPON_EVENT {
        bigint event_id PK
        timestamp(0) event_date PK
        varchar(20) coupon_num FK
        varchar(20) event_type
        bigint promo_id FK
        varchar(10) store_code
        int pos_num
        char(1) is_success
        varchar(4000) error_desc
    }

    PROMO {
        bigint promo_id PK
        uuid promo_guid UK
        timestamp(0) created_date
        varchar(1000) description
        timestamp(0) load_date
    }

    PROMO_STORE {
        bigint promo_id PK, FK
        varchar(10) store_code PK
        date date_from PK
        date date_to
    }

    POS_KEY {
        varchar(10) store_code PK
        int pos_num PK
        date valid_from PK
        varchar(128) secret_key_hash
        date valid_to
    }

    MANUAL_OPERATION_TASK {
        uuid task_id PK
        varchar(20) operation_type
        int total_count
        int processed_count
        varchar(30) status
        jsonb result_summary
        timestamp(0) created_at
        timestamp(0) completed_at
    }

    IDEMPOTENCY_KEY {
        varchar(64) idempotency_key PK
        varchar(20) coupon_num
        varchar(20) operation_type
        varchar(20) result_code
        timestamp(0) created_at
    }