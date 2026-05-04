# Модель данных и технологии хранения — VetHub

> Предположения о масштабе основаны на бизнес-цели: 150k MAU через 6 месяцев.

## Сущности и технологии хранения

| Сущность | Тип хранилища | Обоснование |
|---|---|---|
| Пользователь | PostgreSQL | OLTP, строгая схема, ACID |
| Питомец | PostgreSQL | OLTP, связь с пользователем по FK |
| Клиника | PostgreSQL + PostGIS | Геопоиск: `ST_DWithin` по координатам |
| Ветеринар | PostgreSQL | OLTP, связь с клиникой |
| Услуга | PostgreSQL | OLTP, связь с ветеринаром и клиникой |
| Запись | PostgreSQL (строгий ACID) | Двойное бронирование недопустимо |
| Отзыв | PostgreSQL | OLTP, связь с записью для верификации |
| Платёж | PostgreSQL (строгий ACID) | Финансовые данные, 152-ФЗ |
| Файлы (фото, сканы) | Yandex Object Storage (S3) | В БД хранятся только ссылки |
| Кэш поиска | Redis (TTL 10 мин) | Список клиник в радиусе, профили клиник |

---

## Атрибуты сущностей

### Пользователь

```
user_id         UUID (PK)
phone_number    VARCHAR UNIQUE (хэшируется для 152-ФЗ)
full_name       VARCHAR
email           VARCHAR (опционально)
hashed_password VARCHAR (NULL при входе через соцсети)
provider        ENUM (phone, google, apple)
created_at      TIMESTAMPTZ
updated_at      TIMESTAMPTZ
last_login_at   TIMESTAMPTZ
```

### Питомец

```
pet_id          UUID (PK)
owner_id        UUID (FK → users)
name            VARCHAR
species         VARCHAR (кошка, собака и т.д.)
breed           VARCHAR
birth_date      DATE
weight_kg       DECIMAL(5,2)
allergies       TEXT
vaccinations    JSONB
medical_notes   TEXT
photo_urls      TEXT[]
```

### Клиника

```
clinic_id       UUID (PK)
name            VARCHAR
address         TEXT
latitude        DECIMAL(9,6)
longitude       DECIMAL(9,6)
phone           VARCHAR
working_hours   JSONB  -- {"mon": "09:00-21:00", ...}
is_24_7         BOOLEAN
photos          TEXT[]
license_info    VARCHAR
rating          DECIMAL(3,2)
status          ENUM (active, pending_moderation, suspended)
created_at      TIMESTAMPTZ
```

### Ветеринар

```
vet_id              UUID (PK)
clinic_id           UUID (FK → clinics, NULL для фрилансеров)
full_name           VARCHAR
specializations     TEXT[]
experience_years    INTEGER
photo_url           TEXT
certificate_urls    TEXT[]
schedule            JSONB
rating              DECIMAL(3,2)
status              ENUM (active, pending_moderation, blocked)
```

### Услуга

```
service_id          UUID (PK)
vet_id              UUID (FK → vets)
clinic_id           UUID (FK → clinics, NULL для фрилансера)
name                VARCHAR
description         TEXT
price               DECIMAL(10,2)
duration_minutes    INTEGER
is_active           BOOLEAN
```

### Запись

```
appointment_id      UUID (PK)
pet_id              UUID (FK → pets)
owner_id            UUID (FK → users)
vet_id              UUID (FK → vets)
clinic_id           UUID (FK → clinics)
service_id          UUID (FK → services)
start_time          TIMESTAMPTZ
end_time            TIMESTAMPTZ
status              ENUM (pending_confirmation, confirmed, completed,
                          cancelled_by_owner, cancelled_by_clinic)
price_final         DECIMAL(10,2)
created_at          TIMESTAMPTZ
updated_at          TIMESTAMPTZ
```

### Отзыв

```
review_id           UUID (PK)
author_id           UUID (FK → users)
target_type         ENUM (clinic, vet)
target_id           UUID
appointment_id      UUID (FK → appointments)
rating              SMALLINT (1–5)
message_text        TEXT
media_urls          TEXT[]
status              ENUM (pending_moderation, published, rejected)
created_at          TIMESTAMPTZ
```

### Платёж

```
payment_id              UUID (PK)
appointment_id          UUID (FK → appointments)
user_id                 UUID (FK → users)
amount                  DECIMAL(10,2)
status                  ENUM (pending, succeeded, failed, refunded)
payment_method          ENUM (card, sbp)
external_payment_id     VARCHAR
receipt_url             TEXT
created_at              TIMESTAMPTZ
updated_at              TIMESTAMPTZ
```

---

## Обоснование реляционной БД как основного хранилища

Все сущности имеют чёткую, предсказуемую структуру со связями по внешним ключам. Требования ACID критически важны для сущностей **Запись** и **Платёж**. Сложные JOIN необходимы для истории записей (Пользователь → Питомец → Запись → Клиника → Ветеринар → Отзыв). Хранение персональных данных в структурированном виде с разграничением доступа проще реализовать в РСУБД.

### Дополнительные надстройки

- **PostGIS** — геопространственные запросы («найти все клиники в радиусе 5 км от точки»)
- **PostgreSQL tsvector** (или Elasticsearch в v2.0) — полнотекстовый поиск по клиникам, специализациям, услугам
- **Redis** — кэш часто запрашиваемых данных (список услуг клиники, профиль клиники), TTL 10 минут, инвалидация по триггеру из админки
- **Yandex Object Storage** — хранение всех файлов (фото, сканы документов). В БД — только ссылки

---

## Масштаб (оценка для MVP / 18 мес)

| Сущность | MVP (6 мес) | 18 мес |
|---|---|---|
| Пользователи | ~150k MAU | ~500k зарегистрированных |
| Клиники | ~800 (Москва) | ~2 000+ |
| Записи | ~30k/мес | ~360k/год |
| Отзывы | ~6–9k/мес | — |
| Платежи | ~30k/мес | — |
