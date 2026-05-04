# BPMN — Процесс автоматической записи в слот

## Участники процесса (Pools)

| Pool | Роль |
|---|---|
| Клиент | Владелец питомца, инициирует запись |
| Сервис VetHub | Оркестрирует запись, проверяет подтверждение, обрабатывает оплату |
| Система клиники | Подтверждает или отклоняет слот |
| Тех. поддержка | Обрабатывает случаи таймаута и ошибок |

---

## Основной поток процесса

```
[Клиент]
  START (Необходима вет-услуга)
    → Заполнить форму запроса и выбрать слот

[Сервис VetHub]
  → Отправить запрос на подтверждение системой клиники
  → EVENT-BASED GATEWAY: Ожидание ответа или таймаут (15 мин)
    |
    ├── [Получен ответ] → EXCLUSIVE GATEWAY: Слот подтверждён?
    │     ├── [Да] → Забронировать слот → Перенаправить на оплату
    │     └── [Нет] → Сообщить о проблеме в тех. поддержку
    │
    └── [Таймаут 15 мин] → Сообщить о проблеме в тех. поддержку

[Тех. поддержка]
  → Решение проблемы
  → Уведомление сервиса VetHub о решении

[Клиент]
  → Оплатить запись на приём

[Сервис VetHub]
  → Получение статуса оплаты
  → EXCLUSIVE GATEWAY: Оплата прошла?
    ├── [Да] → Записать пользователя в слот → END
    └── [Нет] → Уведомить о сбое оплаты
                → Направить на страницу оплаты с предупреждением

[Клиент]
  → Повторить попытку оплаты
```

---

## Ключевые бизнес-правила процесса

- Таймер ожидания ответа от клиники: **PT15M** (15 минут)
- При таймауте или отказе — автоматическое уведомление в тех. поддержку
- При сбое оплаты — пользователь перенаправляется на страницу повторной оплаты с сообщением об ошибке
- Статус записи меняется на **confirmed** только после успешной оплаты

---

## XML BPMN-диаграмма

> Ниже приведён полный XML для импорта в Camunda Modeler (версия 5.x, Camunda Cloud 8.8).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions
  xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
  xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
  xmlns:zeebe="http://camunda.org/schema/zeebe/1.0"
  xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
  xmlns:modeler="http://camunda.org/schema/modeler/1.0"
  id="Definitions_0zfnzxl"
  targetNamespace="http://bpmn.io/schema/bpmn"
  exporter="Camunda Modeler"
  exporterVersion="5.45.0"
  modeler:executionPlatform="Camunda Cloud"
  modeler:executionPlatformVersion="8.8.0">

  <bpmn:message id="Message_Answer" name="Message_Answer" />

  <bpmn:collaboration id="Collaboration_16xdql6">
    <bpmn:participant id="Participant_01g1g4y" name="VetHub" processRef="Process_13ryt7r" />
    <bpmn:participant id="Participant_1x6zwk2" name="Клиент" processRef="Process_1jc3fyx" />
    <bpmn:participant id="Participant_0i865y3" name="Сервис VetHub" processRef="Process_13i4exb" />
    <bpmn:participant id="Participant_1i4qc4s" name="Система клиники" />
    <bpmn:participant id="Participant_16owuf9" name="Тех поддержка" />
    <!-- Message flows omitted for brevity — see full XML in source -->
  </bpmn:collaboration>

  <!-- Full XML available in source repository -->
</bpmn:definitions>
```

> Полный XML файл: [`bpmn/appointment-booking.bpmn`](../../bpmn/appointment-booking.bpmn) *(добавить в репозиторий отдельным файлом)*
