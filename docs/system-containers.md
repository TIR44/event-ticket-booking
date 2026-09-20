# C4 Containers: первая поставка

Представление проектируемых исполняемых частей, не список уже развёрнутых систем. Внешние участники — в [C4 Context](system-context.md); доменные границы — на [Context Map](domain-model.md).

```mermaid
flowchart TB
    U["Покупатель / организатор"]
    subgraph S["Event Ticket Booking"]
      WEB["Web-приложение в браузере\nUI, восстановление состояния"]
      EDGE["Статический origin / reverse proxy\nДоставка web и маршрутизация API"]
      API["Backend API\nCatalog, Booking, Payments, Notifications"]
      WORK["Backend worker\nOutbox, истечение, сверка"]
      CD[("catalog_db")]
      BD[("booking_db")]
      PD[("payments_db")]
      ND[("notifications_db")]
    end
    PAY["Платёжный провайдер"]
    MSG["Сервис уведомлений"]
    U -->|"Работа с UI"| WEB
    WEB -->|"HTTPS: HTML, JS и API"| EDGE
    EDGE -->|"HTTP: API-запрос"| API
    API -->|"Catalog adapter: SQL"| CD
    API -->|"Booking adapter: SQL"| BD
    API -->|"Payments adapter: SQL"| PD
    API -->|"Notifications adapter: SQL"| ND
    WORK -->|"Через адаптер Catalog"| CD
    WORK -->|"Через адаптер Booking"| BD
    WORK -->|"Через адаптер Payments"| PD
    WORK -->|"Через адаптер Notifications"| ND
    WORK -->|"HTTPS: создание, сверка, возврат"| PAY
    PAY -->|"HTTPS: callback"| EDGE
    WORK -->|"HTTPS: доставка"| MSG
    WEB -->|"Переход на платёжную страницу"| PAY
```

API и worker — два режима запуска одного артефакта с общей версией доменных контрактов. Worker обращается к данным через код владельцев; схема не разрешает произвольные межмодульные SQL. Четыре базы могут жить на одном PostgreSQL instance с четырьмя ролями; их отказ будет коррелированным. Независимых релизов логических сервисов пока нет.

API обрабатывает короткие команды и callback; долгие внешние операции выполняются worker после долговечной постановки работы. Booking принимает намерение начать оплату и outbox в своей транзакции; запрос UI означает приём этого намерения, а не создание внешнего платежа. Доставщик outbox вызывает порт Payments, который фиксирует inbox и попытку в своей БД. Так синхронный внутренний вызов при доставке реализует асинхронный долговечный процесс без ожидания банка в HTTP-запросе покупателя.

Только Booking соединяет резерв, заказ и билет своей транзакцией. Другие БД фиксируют свою работу и очередь, доставка между ними повторяема. При недоступности Booking резерв не принимается; UI сообщает об ошибке/неизвестности и перечитывает прежнюю операцию.

Рендеринг, splitting и доставка раскрываются frontend-работой. Эта схема не требует SSR, Kubernetes, отдельного BFF или брокера.
