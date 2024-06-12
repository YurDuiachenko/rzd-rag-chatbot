# Разговорный чат-бот для пассажиров РЖД

В последнее время, с появлением больших языковых разговорных моделей (Large Language Models, LLM) всё более актуальным и интересным вызовом становится создание предметно-ориентированных чат-ботов, то есть таких, которые способны поддерживать беседу в рамках определённой области, в частности, отвечать на интересующие пользователя вопросы. 

## Классический чат-бот


![faq_cosinus_similarity_matrix](https://github.com/YurDuiachenko/readme-yandex/assets/72216941/27dbf822-1f42-451e-9f94-85f79ca4c5c3)


`Когда ближайший поезд из Москвы в Санкт-Петербург?`
> По вашему запросу я не могу предоставить точную информацию о расписании поездов из Москвы в Санкт-Петербург, так как в предоставленных данных нет соответствующей информации.


`Мне нужно расписание поездов из Новороссийска в Воронеж на 15 июня.`
> К сожалению, я не нашел в предоставленных данных конкретного расписания поездов из Новороссийска в Воронеж на 15 июня. Однако я могу предоставить общую информацию о маршрутах и услугах РЖД.


## API Яндекс.Расписаний


## Классификатор

Изначальный промпт: 
> Ты умный ассистент, который отвечает на сообщения пользователя и вопросы про РЖД.
Ты анализируешь запросы пользователя и предоставленный текст, но не упоминаешь о них в своем ответе!
Если пользователь СПРАШИВАЕТ ЧТО-ТО СВЯЗАННОЕ с РЖД и железными дорогами в целом, то ответь пользователю максимально информативно.
Если пользователь СПРАШИВАЕТ ЧТО-ТО НЕ СВЯЗАННОЕ с РЖД и железными дорогами в целом или в данных НЕТ ответа на его вопрос, тогда ПРОИГНОРИРУЙ ДАННЫЕ и ответь пользователю на интересующий его запрос.

Улучшенный промпт:
> Проанализируй запрос пользователя и дай ответ в формате обычного текста без форматирования markdown. Ты анализируешь запросы и предоставленные данные, но не упоминаешь о них в своем ответе Сделай вид, что ты отвечаешь конкретно мне. В твоем ответе НИЧЕГО КРОМЕ ответа на вопрос пользователя. Если вопрос пользователя ВООБЩЕ НЕ СООТНОСИТСЯ с данными, то проигнорируй их. Если пришли данные про Рейс, Отправление, Прибытие, Дата отправления, Время отправления, Местное время прибытия, то просто скажи: "Конечно! Вот, смотрите:" и не обращай внимание на дату.
> Вопрос пользователя:
>
> {query}
>
> Данные:
>
> {context}
>
> Дополнительная информация:

| Запрос | Изначальная версия классификатора | Улучшенная версия классификатора |
|--------------|-----------|------------|
| Когда ближайший поезд из Москвы в Санкт-Петербург? | БЛИЖАЙШИЙ, Москва, Санкт-Петербург | РАСПИСАНИЕ,       Москва, Санкт-Петербург, дата не указана, поезд. |
| Мне нужно расписание поездов из Новороссийска в Воронеж на 15 июня. | ДАТА, Новороссийск, Воронеж, 2024-06-15 | РАСПИСАНИЕ, Новороссийск, Воронеж, 2024-06-15, поезда. |


`Когда ближайший поезд из Москвы в Санкт-Петербург?`
> Конечно, я могу помочь вам с информацией о ближайших поездах из Москвы в Санкт‑Петербург.
Ближайшим поездом из Москвы в Петербург является поезд № 754A, отправление из Москвы (Ленинградский Вокзал) в 05:55, местное время прибытия — 09:42.
Поезд прибудет на Московский вокзал Санкт-Петербурга.
Этот поезд отправляется каждый день.


`Мне нужно расписание поездов из Новороссийска в Воронеж на 15 июня.`
> Конечно, я могу помочь вам с поиском расписания поездов из Новороссийска до Воронежа на 15 июня 2024 года.
> Расписание поездов по вашему запросу выглядит так:
> 1. Рейс 234C из Новороссийска отправляется в 09.50 и прибывает в Придачу (Воронеж-Южный), местное время прибытия — 04.13.
> 2. Рейс 126C отправляется в 13.40 и прибывает туда же, местное время прибытия 08.15.
> 3. Рейс 506C отправляется из Новороссийска и прибывает на станцию Воронеж-1, местное время — 12.27.

## Имплментация

## Архитектура
![readme-arch](https://github.com/YurDuiachenko/readme-yandex/assets/72216941/71d1680b-97ea-4613-9067-e168eb3f7cec)

## Команда проекта

* [Брежнева Алена](https://github.com/alenka192003)
* [Васильев Владимир](https://github.com/SilentMiver)
* [Дьяченко Юрий](https://github.com/YurDuiachenko)
* [Замуруев Роман](https://github.com/Zamuruev)
* [Карпушин Андрей](https://github.com/recwayer)
* [Левшенко Денис](https://github.com/kottzi)

## Источники

* [Retrieval-Augmented Generation](https://habr.com/ru/articles/772130/)
* [Yandex API Расписаний](https://yandex.ru/dev/rasp/raspapi/)
* [PromptEngineering4Devs](https://github.com/yandex-datasphere/PromptEngineering4Devs)
* [Создаём вопрос-ответного чат-бота (potter-bot)](https://github.com/yandex-datasphere/yatalks-potter-bot)
* [Yandex-chain](https://github.com/yandex-datasphere/yandex-chain)
* [Оцениваем RAG-пайплайны](https://habr.com/ru/articles/778166/)
* [Добавление собственных данных в LLM с помощью RAG](https://habr.com/ru/companies/wunderfund/articles/779748/)
