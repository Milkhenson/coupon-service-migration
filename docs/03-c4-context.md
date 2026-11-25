# C4 Level 1 — System Context

```mermaid
flowchart TB
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef external fill:#999999,stroke:#6b6b6b,color:#fff

    Buyer["Покупатель"]:::person
    Cashier["Кассир"]:::person
    Manager["Менеджер"]:::person
    Support["Поддержка / антифрод"]:::person

    POS["Касса (POS)"]:::external
    Promo["Система акций"]:::external
    DWH["DWH компании"]:::external

    Coupon["Сервис купонов"]:::system

    Buyer -->|"Предъявляет / получает купон"| POS
    Cashier -->|"Работает"| POS
    POS -->|"API, внутренний канал"| Coupon
    Manager -->|"Массовые операции (до 200 000)"| Coupon
    Support -->|"Разбор E, массовое гашение"| Coupon
    Promo -->|"Акции, ТТ, даты (ETL 15 мин)"| Coupon
    Coupon -->|"Купоны и события (Ежедневно + Kafka)"| DWH