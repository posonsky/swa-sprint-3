# Диаграмма контейнеров (С4)

Более подробно технические аспекты решения можно рассмотреть на диаграмме контейнеров. На схеме обозначены только имеющие непосредственное отношение управлению системами УД и сбором данных телеметрии. Микросервисы, предназначенные для управления пользователями, обслуживания бизнес-аудита и т.п., не изображены.

Зелёным цветом выделен контейнер брокера сообщений, т.к. реализация асинхронного взаимодействия между контейнерами внутри системы запланирована на следующую итерацию. В первое время подключений будет не так много, и ReST API микросервисов, полагаю, успешно справится с нагрузкой.

По правде говоря, не хватило мне опыта, чтобы разработать схему асинхронного API, а времени не хватило, чтобы реализовать какие-то прототипы микросервисов, «пощупать» их, а потом уж взяться за разработку схемы. Мне доводилось работать с Celery на Python, но там несколько иная парадигма.

```puml
@startuml
!include <C4/C4_Container>
!include <tupadr3/common>
!include <tupadr3/devicons2/opentelemetry>
!include <tupadr3/devicons2/postgresql>
!include <tupadr3/devicons2/react_original>
!include <tupadr3/devicons2/redis>
!include <tupadr3/font-awesome-6/book_open>
!include <tupadr3/font-awesome-6/database>
!include <tupadr3/font-awesome-6/microchip>
!include <tupadr3/font-awesome-6/mobile_screen>
!include <tupadr3/font-awesome-6/person_military_pointing>
!include <tupadr3/font-awesome-6/right_left>
!include <tupadr3/font-awesome-6/route>
!include <tupadr3/font-awesome-6/temperature_three_quarters>

title [Container] ПП «Хитрый Дом»

System_Ext(frontend, "HTTP-фронтенд", \
    "Предоставляет интерфейс для взаимодействия с пользователями системы", \
    $sprite=react_original, $type="Software System: React")
System_Ext(mobile_app, "Мобильное приложение", \
    "Предоставляет интерфейс для управления УД", \
    $sprite=mobile_screen, $type="Software System")

System(module, "Модуль", "Модуль управления устройствами умного дома", \
    $sprite=microchip, $type="Software System")

System_Boundary(c1, "ПП Хитрый дом") {
    ContainerQueue(broker, "Брокер сообщений", "Container: NATS", \
        "Маршрутизирует сообщения между сервисами", $sprite=right_left, \
        $link="https://nats.io/") #00bb00
    Container(apigw, "API шлюз", "Container: Kusk", \
        "Маршрутизирует ReST запросы", $sprite=route)
    ContainerQueue(collector, "Брокер MQTT", \
        "Container: EMQX", \
        "Направляет в БД поток данных о состоянии от устройств", \
        $link="https://www.emqx.com/en/")

    Container(inventory, "Реестр", "Container: Python", \
        "Управляет взаимосвязями и свойствами объектов", $sprite=book_open, \
        $link="../03_C4_component_inventory/")
    Container(commander, "Командир",  "Container: Python", \
        "Управляет устройствами", $sprite=person_military_pointing, \
        $link="../04_C4_component_commander/")
    Container(telemetry, "Телеметрия", "Container: Python", \
        "Предоставляет данные телеметрии", \
        $sprite=temperature_three_quarters, \
        $link="../05_C4_component_telemetry/")
    
    ContainerDb(db_inventory, "БД Реестра", \
        "Container: PostgreSQL", "Хранит данные Реестра", $sprite=postgresql)    
    ContainerDb(cache_commander, "Кэш Командира", \
        "Container: Valkey", "Обслуживает кэш Командира", $sprite=redis, \
        $link="https://valkey.io/")
    ContainerDb(db_telemetry, "БД Телеметрии", "Container: TDEngine", \
        "Хранит данные Телеметрии", $sprite=database, \
        $link="https://tdengine.com/")
}

Lay_R(broker, apigw)

Rel(inventory, broker, "Отправляет и получает сообщения", $techn="MessagePack") 
Rel(commander, broker, "Отправляет и получает сообщения", $techn="MessagePack")
Rel(telemetry, broker, "Отправляет и получает сообщения", $techn="MessagePack")

Rel(inventory, db_inventory, "Сохраняет, обновляет, выбирает данные", \
    $techn="SQL/TCP")
Rel(commander, cache_commander, "Сохраняет, выбирает данные")
Rel(telemetry, db_telemetry, "Выбирает данные", $techn="SQL/TCP")

Rel(apigw, inventory, "Транслирует запросы к ReST API", $techn="ReST/JSON")
Rel(apigw, commander, "Транслирует запросы к ReST API", $techn="ReST/JSON")
Rel(apigw, telemetry, "Транслирует запросы к ReST API", $techn="ReST/JSON")

Rel(frontend, apigw, "Направляет запросы к ReST API системы", \
    $techn="ReST / JSON")

Rel(commander, module, "Посылает команды", $techn="gRPC || MQTT")
Rel(module, collector, "Отправляет телеметрию", $techn="MQTT")
Rel_L(collector, db_telemetry, "Вставляет данные", $techn="SQL/TCP")

Rel(mobile_app, module, "Управляет системой УД", $techn="gRPS")
Rel(mobile_app, apigw, \
    "Просматривает историю/Управляет системой (резервный канал)", \
    $techn="gRPS")

@enduml 
```

## О выборе программ

Несколько слов о выборе того или иного программного продукта для конкретной задачи.

* В качестве API-шлюза (прямо в задании был предложен) [Kusk](https://kusk.io/). Ничего не имею против (но для реального проекта поискал что-нибудь поживее). Попробовать не успел, но то, что он open source — это уже хорошо. Не очень хорошо, то что он 2 года никак не развивается, если верить GitHub'у.

* Кэш микросервиса «Командир». С тех пор, как Redis сменил лицензию, [Valkey](https://valkey.io/) — весьма достойная замена.

* Брокер сообщений. Не смотря на то, что в задании ненавязчиво предлагалась Kafka, всё же я предпочёл [NATS](https://nats.io/). Он написан на Go — теоретически, более быстрый, менее ресурсоёмкий, проще развёртываемый, динамично развивающийся.

* Брокер MQTT. Просто остановился на лидере [EMQX](https://www.emqx.com/en). Со временем (и средствами на эксперименты) можно мигрировать на OSS-решение. 

* БД Телеметрии. Пока остановился на [TDengine](https://tdengine.com/). На сайте *красивые* графики, open source, развивается.

Безусловно, любой подобный выбор необходимо делать только проведения нагрузочных испытаний на стенде, который максимально полно воспроизводил бы особенности работы проектируемого комплекса приложений.

## О развёртывании

На первом этапе инфраструктуру желательно развернуть в Kubernates в одном ДЦ, так чтобы те приложения, которые могут быть развёрнуты в кластерном варианте, были запущены в кластерном. Бэкап следует настроить и вынести в другой ДЦ. При таком развёртывании непредвиденный даунтайм возможен, но 1) инцидент должен быть весьма серьёзным, что происходит крайне редко, 2) восстановить работу системы можно достаточно быстро и при наличии свежих бэкапов данные почти не утратятся.

В дальнейшем, когда доходы предприятия позволят подобную экспансию проводить не в убыток, будет иметь смысл развернуть аналогичную группу контейнеров в другом ДЦ. Между площадками потребуется настроить репликацию и автоматическую балансировку.

Затем целесообразно довести количество площадок до 3-х. При этом следует учитывать географическое распределение подключенных к системе домохозяйств с тем расчётом, чтобы данные телеметрии можно было шардировать по географическому признаку.
