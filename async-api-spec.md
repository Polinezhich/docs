# Асинхронное взаимодействие — VetHub

## Обоснование выбора Kafka

Платформа VetHub построена на микросервисной архитектуре (сервис записи, сервис уведомлений, сервис клиник, сервис оплаты и т.д.). Выбор Kafka обоснован следующим:

1. **Стандарт для событийной шины** — Kafka является стандартом надёжной событийной шины между микросервисами.
2. **Fanout** — событие `appointment.created` публикуется один раз, но может потребоваться нескольким потребителям (уведомления, аналитика).
3. **Буферизация и масштабирование** — Kafka буферизует события при недоступности сервиса уведомлений и позволяет горизонтально масштабировать обработку. Для агрегатора с потенциально высоким трафиком это критично.

---

## Процесс: уведомление о новой записи

После создания записи клиника должна получить уведомление о поступившей заявке. Взаимодействие асинхронное: владелец питомца не ждёт подтверждения доставки уведомления — ему достаточно факта регистрации записи в системе.

### Sequence-диаграмма (шаблон)

```
Владелец питомца → Сервис записи: POST /appointments
Сервис записи → Kafka [appointment.created]: publish event
Kafka → Сервис уведомлений: consume event
Сервис уведомлений → Клиника: push / email уведомление
```

> Полная sequence-диаграмма добавляется в следующей итерации.

---

## AsyncAPI + Avro контракт

```yaml
asyncapi: 2.6.0
info:
  title: VetHub
  version: 1.0.0
  description: >
    Асинхронные события, связанные с записью на приём в ветеринарные клиники.
    Событие appointment.created публикуется сервисом записи при успешном создании новой заявки.

servers:
  production:
    url: kafka.vethub.local:9092
    protocol: kafka
    description: Kafka-кластер продакшн-окружения

channels:
  vethub.events.appointment.created:
    description: >
      Топик, в который публикуется событие о создании новой записи.
      Консьюмеры: сервис уведомлений (отправка push/email клинике), сервис аналитики.
    publish:
      summary: Публикация события о новой записи
      operationId: onAppointmentCreated
      message:
        name: AppointmentCreatedEvent
        contentType: application/avro
        schemaFormat: application/vnd.apache.avro+json;version=1.9.0
        payload:
          namespace: com.vethub.events
          type: record
          name: AppointmentCreated
          fields:
            - name: appointmentId
              type: long
              doc: Уникальный идентификатор записи
            - name: clinicId
              type: long
              doc: ID клиники
            - name: clinicName
              type: string
              doc: Название клиники
            - name: doctorId
              type: long
              doc: ID выбранного врача
            - name: doctorName
              type: string
              doc: Имя врача
            - name: petId
              type: long
              doc: ID питомца
            - name: petName
              type: string
              doc: Кличка питомца
            - name: petType
              type:
                type: enum
                name: PetType
                symbols: [DOG, CAT, BIRD, REPTILE, RODENT, OTHER]
                doc: Вид питомца
            - name: serviceId
              type: long
              doc: ID услуги
            - name: serviceName
              type: string
              doc: Название услуги
            - name: dateTime
              type:
                type: long
                logicalType: timestamp-millis
              doc: Дата и время приёма (UTC)
            - name: price
              type: double
              doc: Стоимость услуги в рублях
            - name: ownerId
              type: long
              doc: ID владельца питомца
            - name: ownerPhone
              type: string
              doc: Номер телефона владельца (для экстренной связи)
            - name: createdAt
              type:
                type: long
                logicalType: timestamp-millis
              doc: Время создания записи (UTC)
        examples:
          - name: ExampleAppointmentCreated
            payload:
              appointmentId: 100234
              clinicId: 12
              clinicName: "ВетКлиника на Пушкина"
              doctorId: 451
              doctorName: "Иванов Сергей Петрович"
              petId: 789
              petName: "Буба"
              petType: "CAT"
              serviceId: 5
              serviceName: "Первичный приём терапевта"
              dateTime: 1716408000000
              price: 1500.0
              ownerId: 3321
              ownerPhone: "+79991234567"
              createdAt: 1716310800000
```
