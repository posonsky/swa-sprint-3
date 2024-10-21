# ER-диаграмма микросервиса «Реестр»

На диаграмме отражены только самые основные атрибуты. В реальности таблицах могут быть и другие столбцы.

Данные в таблицу `person` подтягиваются из микросервиса, управляющим пользовательскими данными. Таблицы `building`, `object`, `module_type`, `device_type`, `metric_type` заполняются через административный интерфейс. Данные в `module`, `device`, `metric` «провижионятся» из микросервиса «Телеметрия».

```puml
@startuml
'hide circle
skinparam linetype ortho

title [ER Diagram] ПП «Хитрый Дом» — «Реестр»

entity "person" as person {
    * person_id: bigint
    --
}

entity "building" as building {
    * id: bigserial
    --
    * address: varchar2(256)
    person_id: bigint <<FK>>
}

entity "object" as object {
    * id: bigserial
    --
    * building_id: bigint <<FK>>
    person_id: bigint <<FK>>
    comment: varchar2(256)    
}

entity "module_type" as module_type {
    * id: serial
    --
    * type: varchar2(128)
}

entity "module" as module {
    * id: bigserial
    --
    * module_type_id: int <<FK>>
    * object_id: bigint <<FK>>
    * person_id: bigint <<FK>>
}

entity "device_type" as device_type {
    * id: serial
    --
    * type: varchar2(128)
}

entity "device" as device {
    * id: bigserial
    --
    * device_type_id: int <<FK>>
    * module_id: bigint <<FK>>
}

entity "metric_type" as metric_type {
    * id: bigserial
    --
    type: varchar2(128)
}

entity "metric" as metric {
    * id: bigserial
    --
    * device_id: bigint <<FK>>
    * metric_type_id: bigint <<FK>>
}

building ||--o{ object: building_id
person |o..o{ building
person |o..o{ object: person_id
person ||--o{ module: person_id
module ||--o{ device: module_id
module_type ||--o{ module: module_type_id
device_type ||--o{ device: device_type_id
device ||--o{ metric: device_id
metric_type ||--o{ metric: metric_type_id

@enduml
```
