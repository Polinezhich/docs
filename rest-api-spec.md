# REST API Спецификация — VetHub

**Версия:** 1.0.0  
**Base URL:** `https://api.vetclinic.com/v1`  
**Авторизация:** Bearer JWT (`Authorization: Bearer <token>`)

---

## Обзор эндпоинтов

| Метод | Путь | Описание |
|---|---|---|
| GET | `/clinics` | Поиск ближайших клиник |
| GET | `/clinics/{clinicId}` | Детали клиники |
| GET | `/clinics/{clinicId}/slots` | Доступные слоты для записи |
| GET | `/services` | Список ветеринарных услуг |
| POST | `/appointments` | Создание записи |
| GET | `/appointments` | Список записей пользователя |
| GET | `/appointments/{appointmentId}` | Детали записи |
| PATCH | `/appointments/{appointmentId}` | Обновление записи |
| POST | `/payments` | Инициировать платёж |
| POST | `/payments/{paymentId}/confirm` | Подтверждение платежа (3-D Secure) |
| GET | `/payments/{paymentId}/status` | Статус платежа |
| POST | `/cards` | Сохранить платёжную карту |
| GET | `/cards` | Список сохранённых карт |

---

## Эндпоинты

### GET /clinics — Поиск клиник

**Параметры запроса:**

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `lat` | number (double) | Нет* | Широта |
| `lng` | number (double) | Нет* | Долгота |
| `radius` | integer (1–50) | Нет | Радиус поиска в км (по умолчанию: 5) |
| `text` | string | Нет* | Адрес для ручного ввода (заменяет lat/lng) |
| `specialization` | enum | Нет | Специализация |
| `minRating` | float (0–5) | Нет | Минимальный рейтинг |
| `limit` | integer (1–50) | Нет | Кол-во результатов (по умолчанию: 10) |
| `offset` | integer | Нет | Смещение (по умолчанию: 0) |

*Передать либо `lat`+`lng`, либо `text`.

**Ответы:** `200 OK`, `400`, `401`, `500`

---

### GET /clinics/{clinicId} — Детали клиники

**Параметры пути:** `clinicId` (int64, обязательный)

**Ответы:** `200 OK`, `401`, `404`, `500`

---

### GET /clinics/{clinicId}/slots — Свободные слоты

**Параметры:**

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `clinicId` | integer | Да | ID клиники (path) |
| `date` | string (date) | Да | Дата (только будущие >= сегодня) |
| `doctorId` | integer | Нет | Фильтр по врачу |
| `serviceId` | integer | Нет | Фильтр по услуге |

**Ответы:** `200 OK`, `400`, `401`, `404`, `500`

---

### GET /services — Список услуг

**Параметры:**

| Параметр | Тип | Описание |
|---|---|---|
| `category` | enum | Категория услуги |
| `limit` | integer (1–100) | По умолчанию: 20 |
| `offset` | integer | По умолчанию: 0 |

**Категории:** `consultation`, `vaccination`, `surgery`, `diagnostics`, `dentistry`, `grooming`, `laboratory`, `other`

**Ответы:** `200 OK`, `401`, `500`

---

### POST /appointments — Создание записи

**Тело запроса (`CreateAppointmentRequest`):**

```json
{
  "clinicId": 12,
  "doctorId": 451,
  "petId": 789,
  "serviceId": 5,
  "dateTime": "2026-05-23T09:00:00Z",
  "saveAsDraft": false
}
```

**Ответы:** `201 Created`, `400`, `401`, `409` (слот занят), `500`

---

### GET /appointments — Список записей пользователя

**Параметры:** `status` (enum), `limit` (по умолчанию: 20), `offset`

**Статусы записи:** `draft`, `pending_payment`, `paid_confirmed`, `cancelled`, `rejected`

**Ответы:** `200 OK`, `401`, `500`

---

### GET /appointments/{appointmentId} — Детали записи

**Ответы:** `200 OK`, `401`, `403` (чужая запись), `404`, `500`

---

### PATCH /appointments/{appointmentId} — Обновление записи

> Доступно только для статусов `draft` или `pending_payment`.

**Тело запроса (`UpdateAppointmentRequest`):**

```json
{
  "dateTime": "2026-05-24T10:00:00Z",
  "doctorId": 452,
  "serviceId": 6
}
```

**Ответы:** `200 OK`, `400`, `401`, `403`, `404`, `409` (новый слот занят), `500`

---

### POST /payments — Инициировать платёж

**Тело запроса (`CreatePaymentRequest`):**

```json
{
  "appointmentId": 100234,
  "paymentMethod": "card",
  "paymentToken": "tok_xxx",
  "saveCard": false
}
```

**Ответы:** `201 Created` (с заголовком `Location`), `400`, `401`, `404`, `409` (платёж уже существует), `500`

---

### POST /payments/{paymentId}/confirm — Подтверждение 3-D Secure

```json
{
  "confirmationCode": "123456"
}
```

**Ответы:** `200 OK`, `400`, `401`, `404`, `408` (таймаут), `500`

---

### GET /payments/{paymentId}/status — Статус платежа

**Статусы платежа:** `pending`, `succeeded`, `failed`, `timeout`

**Ответы:** `200 OK`, `401`, `404`, `500`

---

### POST /cards — Сохранить карту

```json
{
  "paymentToken": "tok_xxx",
  "saveAsDefault": false
}
```

**Ответы:** `201 Created`, `400`, `401`, `409` (уже сохранена), `500`

---

### GET /cards — Список карт

**Ответы:** `200 OK`, `401`, `500`

---

## Схемы данных (компоненты)

### Специализации (`Specialization`)
`general`, `surgery`, `dermatology`, `cardiology`, `dentistry`, `ophthalmology`

### Типы питомцев (`PetType`)
`dog`, `cat`, `bird`, `reptile`, `rodent`, `other`

### Типы карт (`CardType`)
`Visa`, `Mastercard`, `Mir`

### ErrorResponse
```json
{
  "code": "SLOT_ALREADY_TAKEN",
  "message": "Выбранный слот уже занят",
  "details": {},
  "timestamp": "2026-05-04T12:00:00Z"
}
```

> Полная OpenAPI 3.0 спецификация в формате YAML доступна в файле `openapi.yaml`
