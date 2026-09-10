@startuml to-be-context
top to bottom direction
skinparam shadowing false
skinparam defaultFontName DejaVu Sans
skinparam defaultFontSize 12
skinparam wrapWidth 600
skinparam maxMessageSize 400
skinparam packageStyle rectangle
skinparam componentStyle rectangle
skinparam ArrowColor #777777
skinparam rectangle {
  BorderColor #555555
  BackgroundColor #FFFFFF
}
skinparam cloud {
  BorderColor #777777
  BackgroundColor #F2F2F2
}
skinparam component {
  BorderColor #9E9E9E
  BackgroundColor #F5F5F5
}
skinparam component<<new>> {
  BackgroundColor #C8E6C9
  BorderColor #2E7D32
}
skinparam component<<changed>> {
  BackgroundColor #FFE0B2
  BorderColor #EF6C00
}
skinparam component<<keep>> {
  BackgroundColor #F5F5F5
  BorderColor #9E9E9E
}
skinparam component<<frozen>> {
  BackgroundColor #F5F5F5
  BorderColor #C62828
}
skinparam component<<future>> {
  BackgroundColor #FFFFFF
  BorderColor #1565C0
}
skinparam package<<frozenzone>> {
  BorderColor #C62828
  BackgroundColor #FFF8F8
}
skinparam queue<<new>> {
  BackgroundColor #C8E6C9
  BorderColor #2E7D32
}
skinparam database<<new>> {
  BackgroundColor #C8E6C9
  BorderColor #2E7D32
}
skinparam database<<keep>> {
  BackgroundColor #FFF4CC
  BorderColor #9E9E9E
}
hide stereotype

title RetailCore: целевое состояние первого этапа (TO-BE)

actor "Клиент" as Customer
actor "Оператор поддержки" as Support
actor "Операционный блок" as Ops
actor "Маркетинг, финансы,\nоперации" as Biz

cloud "Маркетплейсы" as MP
cloud "Платёжные\nпровайдеры" as PSP
cloud "Службы доставки,\nлогистические партнёры" as DP
cloud "Склады" as WH
cloud "BI / отчёты" as BI

rectangle "RetailCore (собственный ДЦ)" as RC {

  component "Интернет-магазин, мобильное\nприложение, личный кабинет\nстатус читается из сервиса состояния" <<changed>> as Channels

  package "Java-монолит: путь записи заказа, на первом этапе не выносится" <<frozenzone>> as Mono {
    component "Оформление заказа [ТТ-6]\n+ ключ идемпотентности,\nкомпенсация резерва доставки,\ncorrelation_id" <<changed>> as Checkout
    component "Адаптер промо [ТТ-5]\nскидка есть / скидки нет /\nсервис недоступен" <<changed>> as PromoAdapter
    component "Расчёт цены:\n3 механизма, надбавки" <<frozen>> as Pricing
    component "Резервирование\nтоваров" <<frozen>> as Reserve
    component "Маршрутизация\nдоставки" <<frozen>> as Routing
    component "Оркестрация\nоплаты" <<frozen>> as Payment
    component "Запись статусов: сервис заказов,\nплатёжный модуль, batch, админка" <<frozen>> as Statuses
  }

  package "Сервисы вокруг ядра" as Aux {
    component "Промо-сервис [ТТ-5]\nвсе новые механики акций" <<changed>> as Promo
    component "Сервис наличия" <<keep>> as Avail
  }

  database "Oracle: путь записи без изменений\nORDERS, история статусов, PL/SQL\n(legacy_promo_pkg заморожен),\nбез новых потребителей" <<keep>> as Oracle

  component "Пакетный контур: batch-переходы,\nповтор резерва доставки,\nночные SQL-скрипты" <<keep>> as Async

  component "Захват изменений\nзаказа [ТТ-1]" <<new>> as Capture
  queue "Брокер: канонические события\nзаказа и доставки [ТТ-1]\nevent_id, order_id, correlation_id" <<new>> as Broker

  component "Сервис состояния заказа [ТТ-2]\nслой перевода статусов;\nклиентский статус; внутреннее\nсостояние с причиной; история" <<new>> as State

  database "Аналитическое хранилище [ТТ-3]\nсогласованные определения\nотмен, возвратов, промо, выручки" <<new>> as Store

  package "Логистический интеграционный контур" as LogZone {
    component "Слой логистических интеграций [ТТ-4]\nперевод форматов партнёров,\nединый контракт статуса доставки,\nидемпотентный приём подтверждений,\nжурнал изменений настроек" <<changed>> as LogInt
    component ".NET-сервис [ТТ-4]\nвход из события\n«заказ подтверждён»" <<changed>> as DotNet
  }

  component "Административные инструменты\nчтение из сервиса состояния,\nручные действия через монолит" <<changed>> as Admin
  component "CRM поддержки" <<keep>> as CRM
  component "Python-задания аналитики [ТТ-3]\nтяжёлые выгрузки из хранилища" <<changed>> as Py

  component "Новые сервисы (горизонт года):\nперсонализация, партнёрские сценарии" <<future>> as NewSvc
}

' --- клиентский вход ---
Customer --> Channels : оформление,\nпросмотр статуса
Channels --> Checkout : createOrder\n+ ключ идемпотентности
Channels -[#2E7D32,bold]-> State : клиентский статус,\nистория
MP --> Checkout : заказы, обновления,\nспецстатусы (без изменений)

' --- путь записи: без изменений, кроме ТТ-5 и ТТ-6 ---
Checkout --> Pricing : сумма заказа
Pricing -[#EF6C00]-> PromoAdapter : расчёт скидки
PromoAdapter -[#EF6C00]-> Promo : calculateDiscount\nсинхронно
PromoAdapter -[#EF6C00,dashed]-> Oracle : резервный расчёт\nтолько при недоступности,\nпуть расчёта в журнал
Checkout --> Avail : наличие
Checkout --> Reserve : резерв товаров
Checkout --> Routing : правило маршрута
Routing --> DP : reserveSlot\nсинхронно
Checkout -[#EF6C00,dashed]-> DP : отмена резерва слота\nпри сбое оплаты\n(если партнёр поддерживает)
Checkout --> Payment : total, paymentType
Payment --> PSP : createSession\nсинхронно
Checkout --> Oracle : запись заказа
Statuses --> Oracle : статусы, история
Async ..> Oracle : batch, ночные SQL

' --- новый путь чтения и событий ---
Oracle -[#2E7D32,bold]-> Capture : изменения заказа\nи статусов
Capture -[#2E7D32,bold]-> Broker : бизнес-события\nзаказа
LogInt -[#2E7D32,bold]-> Broker : события\nдоставки
Broker -[#2E7D32,bold]-> State : события
Broker -[#2E7D32,bold]-> Store : события
Broker -[#2E7D32,bold]-> DotNet : «заказ подтверждён»
Broker -[#1565C0,dashed]-> NewSvc : события
NewSvc -[#1565C0,dashed]-> State : состояние заказа

' --- логистика ---
DotNet ..> DP : партнёрская\nлогистика
LogInt ..> DP : заявки
DP ..> LogInt : подтверждения,\nстатусы партнёров
LogInt ..> WH : складские\nинтеграции
Ops --> LogInt : перераспределение\nпотока; изменения\nнастроек в журнале

' --- поддержка ---
Support --> Admin : разбор заказов
Admin -[#2E7D32,bold]-> State : состояние, причина,\nистория
Admin --> Statuses : ручной перевод\nв технический статус
Support --> CRM : обращения

' --- аналитика ---
Py -[#2E7D32]-> Store : тяжёлые выгрузки
Py ..> Oracle : оставшиеся лёгкие\nвыборки (сокращаются)
BI -[#2E7D32]-> Store : отчёты
BI ..> Oracle : оставшиеся\nзапросы
Biz --> BI : отчёты по выручке,\nкампаниям

legend bottom
  | <b>Обозначение</b> | <b>Смысл</b> |
  | зелёная заливка | новое, выносится первым |
  | оранжевая заливка | изменяется на первом этапе |
  | серая заливка | без изменений |
  | серая заливка, красная рамка | зона высокого риска, на первом этапе не трогаем |
  | синяя рамка | новые сервисы горизонта года, подключаются к событиям и сервису состояния |
  | <color:#2E7D32>━━━</color> | новый поток событий и чтения |
  | <color:#EF6C00>───</color> | изменённая связь |
  | - - - | асинхронно, пакетно, через таблицы |
  | <b>Убрано</b> | .NET-сервис больше не читает INT_ORDER_EXPORT; тяжёлые аналитические выгрузки не читают операционную Oracle; каналы и поддержка не читают статус из Oracle напрямую |
  | <b>ТТ-1</b> | журнал бизнес-событий заказа и доставки |
  | <b>ТТ-2</b> | сервис состояния заказа и двухуровневая модель статусов |
  | <b>ТТ-3</b> | аналитический контур на событиях |
  | <b>ТТ-4</b> | единый контракт статуса доставки, .NET на событиях |
  | <b>ТТ-5</b> | изоляция расчёта скидки за контрактом промо-сервиса |
  | <b>ТТ-6</b> | защитные изменения оформления без переноса логики |
endlegend

@enduml
