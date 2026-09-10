@startuml as-is-context
left to right direction
skinparam shadowing false
skinparam defaultFontName DejaVu Sans
skinparam defaultFontSize 12
skinparam wrapWidth 600
skinparam maxMessageSize 400
skinparam packageStyle rectangle
skinparam componentStyle rectangle
skinparam ArrowColor #555555
skinparam rectangle {
  BorderColor #555555
  BackgroundColor #FFFFFF
}
skinparam component {
  BorderColor #555555
  BackgroundColor #F8F8F8
}
skinparam database {
  BorderColor #555555
  BackgroundColor #FFF4CC
}
skinparam queue {
  BorderColor #555555
  BackgroundColor #EAF4FF
}
skinparam cloud {
  BorderColor #777777
  BackgroundColor #F2F2F2
}
skinparam note {
  BackgroundColor #FFEBEE
  BorderColor #C62828
  FontSize 11
}

title RetailCore: контекст и текущее устройство (AS-IS)

actor "Клиент" as Customer
actor "Оператор поддержки,\nстаршие специалисты" as Support
actor "Операционный блок" as Ops
actor "Маркетинг, финансы,\nоперации" as Biz

rectangle "RetailCore (собственный ДЦ)" as RC {

  component "Интернет-магазин\nи мобильное приложение,\nличный кабинет" as Channels

  package "Java-монолит: центральный оркестратор заказа" as Mono #FFF5F5 {
    component "Оформление заказа\nOrder API, createOrder [!1]" as Checkout #FFCDD2
    component "Расчёт цены и промо\n3 механизма, ветки по каналу\nи региону в коде" as Pricing
    component "Резервирование товаров" as Reserve
    component "Маршрутизация доставки\n[!4]" as Routing
    component "Оркестрация оплаты" as Payment
    component "Управление статусами\n[!2]" as Statuses #FFCDD2
    component "Уведомления клиенту,\nвыгрузки для аналитики" as Notif
  }

  package "Сервисы вокруг ядра" as Aux {
    component "Промо-сервис" as Promo
    component "Сервис наличия" as Avail
  }

  package "Логистический интеграционный контур [!4]" as LogZone #FFF5F5 {
    component "Логистические интеграции\nREST / брокер / выгрузки,\nконфигурация ограничений\nпо партнёрам" as LogInt
    component ".NET-сервис\nпартнёрских интеграций" as DotNet
  }

  queue "Брокер, очереди, пакетные задания:\nbatch-переходы статусов,\nповтор резерва доставки,\nночные SQL-скрипты маршрутов" as Async

  database "Oracle [!3]\nORDERS, история статусов,\nPL/SQL: legacy_promo_pkg и правила,\nсправочники маршрутов,\nINT_ORDER_EXPORT (staging),\nRETRY_DELIVERY_QUEUE" as Oracle #FFE0B2

  component "Административные\nинструменты" as Admin
  component "CRM поддержки" as CRM
  component "Python-задания\nаналитики [!5]" as Py
}

cloud "Маркетплейсы" as MPActor
cloud "Платёжные\nпровайдеры" as PSP
cloud "Службы доставки,\nлогистические партнёры" as DP
cloud "Склады" as WH
cloud "BI / отчёты" as BI

' --- клиентский вход ---
Customer --> Channels : оформление,\nпросмотр статуса
Channels --> Checkout : createOrder\nсинхронно
MPActor --> Checkout : заказы, обновления,\nспецстатусы\n(старые API)
Customer <.. Notif : письма\nо статусе

' --- синхронная цепочка оформления ---
Checkout -[#C62828,bold]-> Pricing : сумма заказа
Pricing -[#C62828,bold]-> Promo : calculateDiscount\nсинхронно
Pricing -[#C62828,bold]-> Oracle : fallback:\nlegacy_promo_pkg\n.calculate_discount
Checkout -[#C62828,bold]-> Avail : наличие\nсинхронно
Checkout --> Reserve : резерв товаров
Checkout -[#C62828,bold]-> Routing : findRule(region,\ndeliveryType, channel)
Routing -[#C62828,bold]-> DP : reserveSlot\n{order_id, regionCode,\ndeliveryType, partnerCode}\nсинхронно
Checkout -[#C62828,bold]-> Payment : total, paymentType
Payment -[#C62828,bold]-> PSP : createSession\nсинхронно\n→ sessionId, redirectUrl

' --- запись в Oracle ---
Checkout --> Oracle : ORDERS: insert;\nupdate payment_session_id,\ndelivery_partner;\nINT_ORDER_EXPORT\n{order_id, export_state}
Routing --> Oracle : правила маршрутов;\nпри неуспехе резерва\nRETRY_DELIVERY_QUEUE
Statuses --> Oracle : статусы,\nистория
Payment --> Oracle : переходы\nстатусов оплаты

' --- асинхронный и пакетный контур ---
Async ..> Oracle : batch-процедуры,\nночные SQL
Async ..> Statuses : переходы статусов\nпо расписанию
Async ..> LogInt : события заказа
Oracle ..> DotNet : чтение\nINT_ORDER_EXPORT
DotNet ..> DP : партнёрская\nлогистика
LogInt ..> DP : REST / брокер /\nвыгрузки пачками
LogInt ..> WH : складские\nинтеграции
DP ..> LogInt : подтверждения\n(с задержкой)

' --- ручной контур ---
Ops ..> LogInt : запрос на\nперераспределение\nпотока между партнёрами
Support --> Admin : разбор зависших\nзаказов
Admin -[#C62828]-> Statuses : ручной перевод\nв технический статус
Admin --> Oracle : технические\nатрибуты заказа
Support --> CRM : обращения

' --- аналитика ---
Py -[#C62828]-> Oracle : прямые SQL,\nисторические выборки\nв пиковые окна
Notif ..> Py : выгрузки\nмонолита, staging
Py ..> Aux : API сервисов
BI -[#C62828]-> Oracle : отчётные запросы
Biz --> BI : отчёты по выручке,\nкампаниям

legend bottom
  |= Обозначение |= Смысл |
  | <color:#C62828>━━━</color> | синхронная цепочка оформления заказа |
  | <color:#C62828>───</color> | проблемная связь |
  | ─── | синхронный вызов или прямая запись |
  | - - - | асинхронно, пакетно, через таблицы |
  | розовая заливка | компонент или контур в проблемной зоне |
  | <b>Зона</b> | <b>Проблема</b> |
  | [!1] Оформление | цена, промо, запись в staging, маршрут, резерв доставки и платёжная сессия в одном синхронном вызове; длинные транзакции; нет защиты от повторного оформления |
  | [!2] Статусы | переходы в сервисе заказов, платёжном модуле, batch-процедурах и админке; в коде WAITING_MANUAL_REVIEW, DELIVERY_PENDING, PAYMENT_PENDING вне формального цикла |
  | [!3] Oracle | бизнес-логика в пакетах, интеграция через таблицы, аналитика и операционный контур на одной БД |
  | [!4] Логистика | правила маршрута в четырёх слоях; разная семантика доставки сообщений; дубли заявок при повторах; ручные правки приоритетов, иногда ночью |
  | [!5] Аналитика | несколько «почти канонических» моделей заказа, нет потока событий, тяжёлые выборки нагружают операционную БД |
  | [!6] Наблюдаемость | корреляционные идентификаторы есть не везде, сквозную историю заказа собирают вручную |
endlegend

@enduml
