# VetHub — Техническая документация

> Агрегатор ветеринарных услуг: поиск клиник, онлайн-запись и оплата в одном приложении.

## О проекте

**VetHub** — мобильная платформа формата B2B2C для записи владельцев питомцев к ветеринарам. Агрегирует ветеринарные клиники и фриланс-ветеринаров, обеспечивает онлайн-бронирование слотов с подтверждением в течение 15 минут и оплатой через приложение.

**Бизнес-модель:** комиссия 15% с каждой записи + B2B API-подписка для партнёров (сети клиник, зоосервисы).

**Целевая аудитория MVP:** владельцы питомцев (Москва), ветеринарные клиники, фриланс-ветеринары.

---

## Структура документации

```
docs/
├── requirements/
│   ├── functional-requirements.md     # Функциональные требования (UC-001..UC-008)
│   ├── non-functional-requirements.md # НФТ: производительность, безопасность, SLA
│   └── stakeholders.md                # RACI-матрица и вопросы стейкхолдеров
├── architecture/
│   ├── overview.md                    # Архитектурный обзор (шаблон)
│   ├── platformization.md             # Стратегия платформизации и roadmap
│   └── async-interaction.md           # Асинхронное взаимодействие (Kafka, AsyncAPI)
├── api/
│   ├── rest-api-spec.md               # REST API спецификация (OpenAPI 3.0)
│   └── async-api-spec.md              # AsyncAPI + Avro-контракт (Kafka)
├── data/
│   └── data-storage.md                # Модель данных и выбор технологий хранения
└── processes/
    └── bpmn-appointment.md            # BPMN-процесс записи на приём
```

---

## Быстрый старт для разработчика

```bash
# 1. Клонировать репозиторий
git clone https://github.com/Polinezhich/docs.git
cd docs

# 2. Ознакомиться с требованиями
cat docs/requirements/functional-requirements.md

# 3. Посмотреть REST API
cat docs/api/rest-api-spec.md
```

---

## Ключевые артефакты

| Артефакт | Файл | Статус |
|---|---|---|
| Функциональные требования (UC) | `docs/requirements/functional-requirements.md` 
| Нефункциональные требования | `docs/requirements/non-functional-requirements.md`
| RACI + вопросы стейкхолдеров | `docs/requirements/stakeholders.md`
| REST API (OpenAPI 3.0) | `docs/api/rest-api-spec.md`
| AsyncAPI / Kafka (Avro) | `docs/api/async-api-spec.md`
| Модель данных | `docs/data/data-storage.md`
| BPMN-процесс записи | `docs/processes/bpmn-appointment.md`
| Стратегия платформизации | `docs/architecture/platformization.md`
| Архитектурный обзор | `docs/architecture/overview.md`

---

## Технологический стек

| Слой | Технология |
|---|---|
| База данных | PostgreSQL + PostGIS |
| Объектное хранилище | Yandex Object Storage (S3-совместимое) |
| Кэш | Redis |
| Очередь событий | Apache Kafka |
| Платёжный шлюз | Тинькофф.Касса |
| Авторизация | JWT (7 дней) + SMS OTP |
| Соответствие | 152-ФЗ, 54-ФЗ, PCI DSS Level 2 |

---

## Roadmap

| Этап | Сроки | Цель |
|---|---|---|
| MVP | 1–6 мес | Базовый поиск, запись, оплата (Москва) |
| Пилот B2B2C | 9–12 мес | API для 2–3 партнёров, white-label виджет |
| Масштабирование | 12–18 мес | 20+ B2B-клиентов, микросервисы |
| Экосистема | 18+ мес | Страхование, зооаптеки, открытый API |
