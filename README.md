# Разговорный чат-бот для пассажиров РЖД

В последнее время, с появлением больших языковых разговорных моделей (Large Language Models, LLM) всё более актуальным и интересным вызовом становится создание предметно-ориентированных чат-ботов, то есть таких, которые способны поддерживать беседу в рамках определённой области, в частности, отвечать на интересующие пользователя вопросы. 

## Классический чат-бот
Основным методом для работы с LLM был выбран [RAG (Retrieval-Augmented Generation)](https://habr.com/ru/articles/772130/). Он предполагет дополнение запросов пользователей информацией из внешних источников данных, таких как текстовые документы или базы знаний, путем поиска и извлечения релевантных фрагментов текста (чанков), и их последующей подачи на вход языковой модели вместе с запросом пользователя. Таким образом, модель получает **дополнительный контекст**, что повышает качество и точность ее ответов.

Прежде всего необходима база знаний РЖД. Она была создана в несколько этапов:
1. Сбор данных с веб-сайта РЖД о расписаниях поездов, статусах отправлений, билетных ценах и т.д.
2. Обработка данных (очистка, разбиение на смысловые отрезки) с использованием LLM, таких как **Gemini** или **Claude**
3. Векторизация данных с помощью **YandexEmbedings** 
4. Сохранение векторов в специальном хранилище [LanceDB](https://lancedb.com/)

Затем при комбинации библиотек [LangChain](https://www.langchain.com/) и [YandexChain](https://github.com/yandex-datasphere/yandex-chain) был составлен классический конвеер RAG:
* **Retriever**: поиска релевантных чанков по косинусному сходству
* **PromptTemplate**: структурирование запросов
* **StuffDocumentChain**: “упаковка” документов в запрос
* **YandexGPT**: основной LLM

В итоге архитектура технологии схематично выглядит так:

```mermaid
flowchart LR
    User--query---> Retriver 
    Retriver --search()--> LancDB[(LanceDB)]
    LancDB[(LanceDB)] --chunk--> Retriver
    Retriver --chunk--> Chain
    Chain --query+chunk--> YandexGPT
    YandexGPT --answer---> User
```

## Тестирование
Для тестирования были взяты 10 самых часто задаваемых вопросов с сайта РЖД, а также ответы к ним. Вопросы были пропущены через описанную выше технологию RAG, а полученные ответы дополнили тройку `(вопрос; ответ ржд; ответ RAG)`. 

Далее ответ с сайта РЖД был взят как эталонный, ответ RAG как тестовый, оба были векторизированы. Затем между ними было рассчитано косинусное сходство (как между векторами); данный показатель и стал мерой сравнения двух ответов. 

Для наглядности была построена матрица сравнения, где ответы RAG были сравнены по косинусу не только с соответствующим идеальным ответом, но и с другими, в качестве доказательства, что мера сравнения объективна. По итогу по диагонали видно насколько соответствуют ответы RAG идеальным ответам. 

`Средний показатель схожести ответа ≈0.78.`

![faq_cosinus_similarity_matrix](https://github.com/YurDuiachenko/readme-yandex/assets/72216941/27dbf822-1f42-451e-9f94-85f79ca4c5c3)

Тем не менее были выявлены недостатки. Созданная технология оказалась не способна выдавать релевантные ответы на запросы к динамическим данным, **к расписанию поездов**.

Например, на следующие вопросы были получены такие ответы:

`Когда ближайший поезд из Москвы в Санкт-Петербург?`
> По вашему запросу я не могу предоставить точную информацию о расписании поездов из Москвы в Санкт-Петербург, так как в предоставленных данных нет соответствующей информации.


`Мне нужно расписание поездов из Новороссийска в Воронеж на 15 июня.`
> К сожалению, я не нашел в предоставленных данных конкретного расписания поездов из Новороссийска в Воронеж на 15 июня. Однако я могу предоставить общую информацию о маршрутах и услугах РЖД.

## API Яндекс.Расписаний

Было принято решение об использовании [API Яндекс.Расписаний](https://yandex.ru/dev/rasp/raspapi/) в контексте создания разговорного чат-бота для возможности получения актуальных расписаний поездов, поиска поездов по заданным параметрам, предоставления информации о маршрутах и обновленной информации в реальном времени.

Данный модуль - набор функций для взаимодействия с API сервиса Яндекс.Расписания, который предоставляет информацию о транспортных маршрутах.
* `make_request(endpoint, request)`: отправляет запрос к API Яндекс.Расписания с заданными параметрами и возвращает результат в виде словаря, полученного из JSON-ответа.
* `get_city_code(city_name)`: выполняет поиск кода населенного пункта по его названию с помощью API Яндекс.Расписания.
* `get_routes(from_location, to_location, date=None, limit=None)`: выполняет поиск транспортных маршрутов от одного населенного пункта к другому на заданную дату. Возвращает информацию о маршрутах в виде текстовой строки.

Каждая функция в коде написана так, чтобы обрабатывать возможные ошибки при взаимодействии с API и возвращать понятные сообщения об ошибках в случае неудачи. Благодаря этому модулю динамическая информация о раcписании поездов становится доступна, но как определить, спрашивает ли пользователь о расписании?

## Классификатор

Данный модуль представляет собой реализацию системы, которая использует языковую модель (YandexGPT) для классификации запросов пользователей и предоставления соответствующих ответов. Основная функциональность сосредоточена вокруг обработки запросов. Запрос к языковой модели YandexGPT формируется на основе заданного промпта и сообщения пользователя.

### Промпт-инжиниринг
Промпт, передаваемый модель, содержит инструкции о том, как интерпретировать сообщение пользователя. Модель на основе инструкции должна определить, запрашивает ли пользователь информацию о расписании поездов или нет. Разработка правильной инструкции прошла через несколько версий.

Изначальная версия: 
> Ты умный ассистент, который отвечает на сообщения пользователя и вопросы про РЖД.
Ты анализируешь запросы пользователя и предоставленный текст, но не упоминаешь о них в своем ответе!
Если пользователь СПРАШИВАЕТ ЧТО-ТО СВЯЗАННОЕ с РЖД и железными дорогами в целом, то ответь пользователю максимально информативно.
Если пользователь СПРАШИВАЕТ ЧТО-ТО НЕ СВЯЗАННОЕ с РЖД и железными дорогами в целом или в данных НЕТ ответа на его вопрос, тогда ПРОИГНОРИРУЙ ДАННЫЕ и ответь пользователю на интересующий его запрос.

Промежуточная версия:
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

Финальная версия:
> Ты умный ассистент, который отвечает на сообщения пользователя в формате чат-бота.\n
> У тебя есть информация о сегодняшней дате и времени: {str(today_date)} и {str(today_time)}.\n
> Далее тебе нужно понять, НУЖНО ЛИ пользователю расписание поездов или нет.\n
> Если пользователю НУЖНО расписание поездов, то верни ответ в формате, который будет в примерах запросов и ответов ниже.
> Если пользователь не указал, откуда и куда ему нужно, то уточни эти данные.
> Если пользователь явно не указал дату, то попробуй высчитать ее на основе информации о сегодняшнем дне.\n
> Если пользователю НЕ НУЖНО расписание поездов, то продолжи отвечать на запросы пользователя в режиме чат-бота.\n
> Примеры запросов и ответов, когда пользователю нужно расписание:\n
> 1) Пользователь: Когда поезд из Липецка в Москву? YaGPT: РАСПИСАНИЕ, Липецк, Москва, дата не указана, поезд.\n
> 2) Пользователь: Ближайший поезд от Ростова до Воронежа на завтра? YaGPT: РАСПИСАНИЕ, Ростов-на-Дону, Воронеж, {today_date + datetime.timedelta(days=1)}, поезд.\n
> 3) Пользователь: Какие поезда есть из Новороссийска в Россошь 17 мая? YaGPT: РАСПИСАНИЕ, Новороссийск, Россошь, 2024-05-17, поезда.\n
> 4) Пользователь: Мне нужно расписание поездов из Владивостока в Уссурийск через месяц? YaGPT: РАСПИСАНИЕ, Владивосток, Уссурийск, {today_date + datetime.timedelta(days=30)}, поезда.\n
> 5) Пользователь: Будут ли поезда из Новосибирска в Хабаровск через год? YaGPT: РАСПИСАНИЕ, Новосибирск, Хабаровск, {today_date + datetime.timedelta(days=365)}, поезда.\n
> 6) Пользователь: Поезда из мск в уссур. YaGPT: РАСПИСАНИЕ, Москва, Уссурийск, дата не указана, поезда.\n
> 7) Пользователь: Хочу добраться из астрахани в волгоград. YaGPT: РАСПИСАНИЕ, Астрахань, Волгоград, дата не указана, поезда.

В процессе совершенствования промпта главными проблемами были:
- узкий набор соответсвий
- сложность обработки дат
- непонимание слов "завтра", "послезавтра" и т.д.
- излишняя информация
- непостоянность
- галюцинации

Принятые значительные изменения (советы):
- пониженение температуры модели
- обработка запросов не относящихся к РЖД 
- большое количество разнообразных примеров запросов
- информация о сегодняшней дате и времени
- **единая форма ответа модели**

### Ответ модели
Если модель определяет, что запрос связан с расписанием поездов, она должна сформировать ответ в специальном формате, который включает в себя:
- пункт отправления
- пункт назначения
- дату
- тип запрашиваемой информации (поезд или поезда)

Изначальные версии формата ответа претерпели упрощения и уточнения параметров, что позволило повысить качество.

| Запрос | Изначальная версия классификатора | Финальная версия классификатора |
|--------------|-----------|------------|
| Когда ближайший поезд из Москвы в Санкт-Петербург? | БЛИЖАЙШИЙ, Москва, Санкт-Петербург | РАСПИСАНИЕ,       Москва, Санкт-Петербург, дата не указана, поезд. |
| Мне нужно расписание поездов из Новороссийска в Воронеж на 15 июня. | ДАТА, Новороссийск, Воронеж, 2024-06-15 | РАСПИСАНИЕ, Новороссийск, Воронеж, 2024-06-15, поезда. |

### Логика обработки ответа
- Eсли ответ соответствует формату запроса о расписании поездов, ответ языковой модели анализуется, и из него извлекается необходимая информация о маршруте и дате.
    - Если извлеченная информация о маршруте и дате является действительной, функция get_routes вызывается для получения фактического расписания поездов, и результат объединяется с ответом языковой модели.

- Если запрос не связан с расписанием поездов, модель должна продолжить обрабатывать сообщение в режиме чат-бота.
    - Если извлеченная информация о маршруте и дате не является действительной или в запросе пользователь не хочет расписание (например узнать что входит в купе "Люкс"), то функция rag_answer вызывается для получения ответа из альтернативного источника (LanceDB) без использования API расписания.

### Результат
Классификатор повысил качество ответов. Технология лучше "понимает" чего хочет пользователь. Например: 

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

## Имплементация

Чтобы пользователь мог удобно взаимодействовать с чат-ботом, было принято решение о создании Telegram-бота, в основе которого лежит реализованная реализация технология.
Telegram был выбран для реализации чата по следующим причинам: 
- **Богатый API:** Telegram предоставляет мощный и гибкий API, который поддерживает широкий спектр возможностей, включая обработку сообщений, файлов, мультимедийных данных и т.д. 
- **Популярность и доступность:** Telegram — популярный мессенджер с большим количеством пользователей, что упрощает доступность и использование бота для демонстрации.  
- **Интеграция:** Легкость интеграции с другими сервисами и API, что важно для подключения и использования модели RAG.

Бот работает в режиме `bot.polling()`, который обеспечивает постоянный опрос сервера Telegram и обработку входящих сообщений

## Общая архитектура

В итоге архитекура чат-бота выгдлядит так:

```mermaid
flowchart LR
    User--query--> TGbot[Telegram Bot]
    TGbot[Telegram Bot]--query--> Classifier
    Ya1[YandexGPT] --yes/noo--> Classifier
    Classifier --query+prompt--> Ya1[YandexGPT]
    Classifier--query--> Retriver 
    Classifier--get_roots()--> YandexAPI
    YandexAPI --roots--> Chain
    Retriver -- search()--> LanceDB[(LanceDB)]
    LanceDB[(LanceDB)]--chunk--> Retriver
    Retriver --chunk--> Chain
    Chain --chunk/roots--> YandexGPT
    YandexGPT --answer--> TGbot[Telegram Bot]
    TGbot[Telegram Bot] --answer--> User
```

Пользователь задаёт вопрос **телеграм-боту**, его вопрос отправляется в классификатор. Там, он оборачивается в заранее подготовленный промт и отдается **языковой модели**. 

Модель принимает решение о том, справшивает ли пользователь о расписании поездов. На основе этого решения, классификатор либо отправляет запрос на поиск **в базе знаний РЖД** чанка информации подходящего под вопрос пользователя, либо составляет `GET` запрос к **API Яндекс.Расписаний**. 

Далее, из полученных данных составляется **цепочка LangChain**, которая состоит из дополнительного промта, запроса пользователя и данных, полученных из базы знаний или Яндекс.Расписание API. 
При запуске цепочки, вся информация снова отдается **языковой модели**, ответ которой и получит пользователь. 

## Команда проекта

* [Брежнева Алена](https://github.com/alenka192003)
* [Васильев Владимир](https://github.com/SilentMiver)
* [Дьяченко Юрий](https://github.com/YurDuiachenko)
* [Замуруев Роман](https://github.com/Zamuruev)
* [Карпушин Андрей](https://github.com/recwayer)
* [Левшенко Денис](https://github.com/kottzi)

## Источники

* Хабр-статья про [Retrieval-Augmented Generation](https://habr.com/ru/articles/772130/)
* Документация [Yandex API Расписаний](https://yandex.ru/dev/rasp/raspapi/)
* Мастер-класс по [Промпт-инжинирингу](https://github.com/yandex-datasphere/PromptEngineering4Devs)
* Мастер-класс [Создаём вопрос-ответного чат-бота (potter-bot)](https://github.com/yandex-datasphere/yatalks-potter-bot)
* Документация [Yandex-chain](https://github.com/yandex-datasphere/yandex-chain)
* Хабр-статья [Оцениваем RAG-пайплайны](https://habr.com/ru/articles/778166/)
