# 03. UML-моделі системи «TechFix»

Моделі описують ту саму предметну область, що й вимоги (`02-requirements.md`): заявка на ремонт,
пристрій, кошторис, роботи й запчастини, гарантія.

## 1. Діаграма варіантів використання

Актори: **Клієнт**, **Майстер**, **Менеджер**, **Адміністратор**.

| Актор | Варіанти використання |
|---|---|
| Клієнт | Подати заявку · Переглянути статус · Погодити кошторис · Відхилити кошторис |
| Майстер | Взяти заявку в роботу · Провести діагностику · Сформувати кошторис · Завершити ремонт |
| Менеджер | Прийняти заявку · Призначити майстра · Оформити видачу · Сформувати звіт |
| Адміністратор | Керувати користувачами · Керувати довідниками · Переглянути журнал дій |

Ключові зв'язки: «Погодити кошторис» розширює «Сформувати кошторис» (include-навпаки: погодження
можливе лише після формування), «Оформити видачу» включає «Сформувати гарантійний талон».

## 2. Діаграма класів (основні сутності)

```mermaid
classDiagram
    class Client {
      +int id
      +string full_name
      +string phone
      +string email
    }
    class Device {
      +int id
      +string type
      +string brand
      +string serial_number
    }
    class Request {
      +int id
      +string number
      +string status
      +string access_token
      +datetime created_at
      +changeStatus(status)
    }
    class Estimate {
      +int id
      +int version
      +decimal total
      +string state
      +recalculate()
    }
    class EstimateItem {
      +int id
      +string kind
      +string title
      +decimal price
      +int quantity
    }
    class Work {
      +int id
      +string title
      +decimal hours
    }
    class SparePart {
      +int id
      +string title
      +int stock
      +decimal price
    }
    class Warranty {
      +int id
      +date starts_at
      +date ends_at
    }
    class User {
      +int id
      +string login
      +string role
    }
    Client "1" --> "0..*" Device
    Device "1" --> "0..*" Request
    Request "1" --> "1..*" Estimate
    Estimate "1" --> "1..*" EstimateItem
    EstimateItem --> Work
    EstimateItem --> SparePart
    Request "1" --> "0..1" Warranty
    User "1" --> "0..*" Request : master
```

Пояснення до зв'язків: один клієнт може мати кілька пристроїв; кожна заявка стосується рівно
одного пристрою; кошторис версіонується, тому в заявки їх може бути кілька, але активною є
остання погоджена версія; гарантія створюється лише після видачі.

## 3. Діаграма станів заявки

```mermaid
stateDiagram-v2
    state "Нова" as New
    state "Прийнята" as Accepted
    state "Діагностика" as Diagnostics
    state "Погодження кошторису" as Approving
    state "Ремонт" as Repair
    state "Готова до видачі" as Ready
    state "Видано" as Issued
    state "Відхилено" as Rejected
    state "Скасовано" as Cancelled

    [*] --> New
    New --> Accepted : менеджер прийняв
    Accepted --> Diagnostics : призначено майстра
    Diagnostics --> Approving : кошторис сформовано
    Approving --> Repair : клієнт погодив
    Approving --> Rejected : клієнт відхилив
    Repair --> Ready : роботи завершено
    Ready --> Issued : видано клієнту
    New --> Cancelled : клієнт забрав пристрій
    Accepted --> Cancelled : клієнт забрав пристрій
    Issued --> [*]
    Rejected --> [*]
    Cancelled --> [*]
```

Заборонені переходи (перевіряються на сервері): «Нова» → «Ремонт», «Діагностика» → «Видано»,
будь-який перехід із кінцевих станів.

## 4. Діаграма послідовності: подання заявки та погодження кошторису

```mermaid
sequenceDiagram
    actor C as Клієнт
    participant UI as Веб-інтерфейс
    participant S as RequestService
    participant DB as База даних
    actor M as Майстер

    C->>UI: заповнює форму заявки
    UI->>S: createRequest(dto)
    S->>DB: INSERT client, device, request
    S-->>UI: номер заявки + токен доступу
    UI-->>C: сторінка статусу

    M->>UI: формує кошторис
    UI->>S: addEstimate(requestId, items)
    S->>DB: INSERT estimate, estimate_items
    S-->>M: кошторис збережено (стан «на погодженні»)

    C->>UI: відкриває посилання з токеном
    C->>UI: погоджує суму
    UI->>S: approveEstimate(estimateId, token)
    S->>DB: UPDATE estimate SET state='approved'
    S-->>C: статус «Ремонт»
```

## 5. Діаграма діяльності (спрощена)

Подання заявки → реєстрація пристрою → призначення майстра → діагностика → формування кошторису →
погодження клієнтом (якщо відхилено — закриття заявки) → ремонт → перевірка якості → видача з
гарантійним талоном.
