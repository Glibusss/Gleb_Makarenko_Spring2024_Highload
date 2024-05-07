# Проектирование высоконагруженного сервиса обмена сообщениями

# 1 Тема и целевая аудитория

## Тема

Мессенджер Telegram - кроссплатформенная система мгновенного обмена сообщениями, позволяющая обмениваться текстовыми, голосовыми и видеосообщениями, стикерами и фотографиями, файлами многих форматов.

## MVP
1. Авторизация, регистрация по номеру телефона
2. Отправка текстовых сообщений
3. Отправка голосовых сообщений
4. Отправка файлов
5. Реакции на сообщения
6. Список чатов
7. История сообщений
8. Каналы
9. Метрики для каналов(активность, просмотры, рост подписчиков и т.д.)
10. Поиск людей по номеру телефона

## Продуктовые особенности
* Платная подписка, дающая доступ к распознаванию голосовых сообщений, анимированным стикерам, кастомным эмодзи и эмодзи-статусам

## ЦА
Ниже представлена статистика исследования аудитории Telegram. В опросе приняли участие жители из более чем 70 стран.
Возраст целевой аудитории варьируется в диапазоне от 18 до 64 лет [[1]](https://tgstat.ru/research-2023)
Возрастная группа | Процентное соотношение
------------ | -------------
до 17 лет | 6,4%
18-24 года| 18,8% 
25-34 года| 29,4% 
35-44 года| 23,8% 
45-64 года| 21,6% 

Размер целевой аудитории: 800 млн. пользователей в месяц, 55.2 млн. пользователей в день на момент конца 2023г. [[2]](https://www.demandsage.com/telegram-statistics/)

Общемировая география пользователей на момент 2024г [[3]](https://www.businessofapps.com/data/telegram-statistics/):  
Регион           | Процентное соотношение
---------------- | -------------
Азия             | 38%
Европа           | 27% 
Латинская Америка| 21% 
Страны MENA      | 8% 

География русскоязычных пользователей:
Страна       | Процентное соотношение
-------------| -------------
Россия       | 60%
Беларусь     | 17% 
Украина      | 13% 

Гендерное распределение аудитории Telegram:
Пол          | Процентное соотношение
-------------| -------------
Мужской      | 58%
Женский      | 42% 

# 2 Расчёт нагрузки

Ежемесячная аудитория Telegram: 800 млн. пользователей.

Ежедневная аудитория Telegram: 55.2 млн. пользователей.

Согласно исследованию [[4]](https://www.demandsage.com/telegram-statistics/):

Количество отправленных сообщений в день: 15 миллиардов

Количество регистраций в день: 8.33 млн.

Средняя длительность сессии: 6 минут.

Исходя из исследования [[5]](https://www.tadviser.ru/index.php/Статья:Мессенджеры_(Instant_Messenger,_IM)#:~:text=Как%20выяснилось%2C%20в%20среднем%20пользователь,идут%20Viber%2C%20Telegram%20и%20Skype.):

Количество заходов в день: 24 

### Хранилище данных для пользователя
Посчитаем количество сообщений в день на одного пользователя : 15 млрд. / 55.2 млн. ~ 272 [сообщ/чел]

Параметр          | Число
-------------| -------------
Среднее количество сообщений      | 272(из них 40 - файлы; 10 - голосовые)
Средняя длина сообщения      | 60 символов UTF-8[[6]](https://cyberleninka.ru/article/n/poliformatnyy-messendzher-kak-zhanr-2-0-na-primere-messendzhera-mgnovennyh-soobscheniy-telegram/viewer)
Средняя длина записи одного голосового сообщения| 30 секунд
Средний размер голосвого сообщения при битрейте 128 kbps | 480 кБ
Средний размер одного файла|2 МБ
Срок хранения сообщения | 3 года
Среднее количество чатов у пользователя | 15
Среднее количество единовременно подружаемых сообщений | 6
Среднее количество людей, с которыми пользователь общается в день | 8
Среднее количество реакций на пользователя | 15

Отметим, что размер одной реакции равен 2 байта

Тогда общий объем _сообщений_, отправляемых в день, на одного пользователя равен: 222 * 0.06 (кБ) + 10 * 480 (кБ) + 40 * 2048 (кБ) + 15*0.002 (кБ) = 86733 кБ

Итого: суммарный объем данных на одного пользователя за 3 года: 86733(кБ) * 365 * 3 + 256 = 90.57 ГБ

### Динамический рост

Так как в день отправляется около 15 млрд. сообщений, то можно вычислить, какой объем данных будет занят пользователями за, например, 1 год.

Расчет для фотографий и голосовых сообщений: ((10 * 480 (кБ) +  40 * 2048 (кБ)) * 55 200 000 * 365)/(1024^4) =  1689 ПБ/год
Расчет для текстовых сообщений и реакций: ((222 * 0.06 (кБ) + 15 * 0.002 (кБ)) * 55 200 000 * 365)/(1024^4) = 0.24 ПБ/год

### Сетевой трафик
При расчете сетевого трафика не будем учитывать запросы, связанные с регистрацией пользователей, так как они не создают ощутимую нагрузку на наш сервис.
Основная нагрузка приходится на отправку сообщений, загрузку списка чатов и историю сообщений.

### Предварительные расчеты:
Посчитаем объем трафика для списка чатов на одного человека с учетом количества заходов в сутки: 24 * 15 * 101.3 кБ = 36 468 кБ
Посчитаем объем трафика для истории сообщений на одного человека с учетом  количества единовременно подгружаемых сообщений в диалоге(канале), среднего количества людей, с которыми общается пользователь в день(каналов, на которые пользователь подписан): 6 * 15 * 0.06 (кБ) = 5.4 кБ

Согласно графику дневной активности в телеграм[[7]](https://stock-bt.ru/wp-content/uploads/7/8/d/78d07edab4abb2eb268bc027bf59142d.png)
пиковое значение активности пользователей(соответственно RPS тоже) приблизительно в 1.66 раза выше среднего значения. Возьмем коэффициент запаса равный 2

Рассмотрим трафик по видам активностей:

Тип          | Отправка (дневная аудитория 55.2 млн)   | Отправка Гб/сек | Пиковое значение | Значение с коэффициентом запаса 2 |
-------------| ------------------------------------ |---------------- |----------------- |---------------------------------- |
Текстовое     | 222 * 55.2 млн. * 0. 06 кБ             | 0.008            | 0.013|0.026|
   Голосовое     | 10 * 55.2 млн. * 480 кБ                 | 2.92              | 4.85 | 9.69|
   Файл    | 40 * 55.2 млн. * 2048 кБ                | 50              | 83 | 166
   Список чатов(подгрузка чатов при заходе в приложение)  | 36 468 кБ * 55.2 млн.                  | 22.22 | 36.52 |73.04|
   История сообщений  | 4.8 кБ * 55.2 млн.                | 0.004             | 0.007|0.13|
   Реакции | 15 * 0.002 * 55.2 млн. | Незначительно | Незначительно | Незначительно |
   Итого  |                                             | 75.152           | 124.752 | 249.5|

### RPS
 
* Отправка сообщения: 55.2 млн * 272 / 86400 = 141 833 RPS
* Регистрация: 8.33 млн / 86400 = 96 RPS
* Авторизация: 55.2 млн. / 86400 = 638.8 ~ 639 RPS
* Получение списка чатов: 55.2 млн. * 24 / 86400 = 15 333 RPS
* Получение истории сообщений: 55.2 млн. * 6 / 86400 = 3833 RPS

Действие                            | RPS | Пиковое значение | Пиковое значение с коэффициентом запаса 2  |
------------------------------------| --- |----------------- |------------------------------------------- |
Отправка сообщения                  | 173 777 | 288 469 | 576 939
Регистрация                         | 96 | 159 | 318|
Авторизация                         | 639 | 1061 | 2 122|
Получение списка чатов              | 15 333 | 25453 | 50 906|
История сообщений                   | 3 833 | 8484 | 16 968|
**Итого**                           | 193 678 | 321 492 | 642 984 | 

# 3 Глобальная балансировка нагрузки

Общемировая география пользователей на момент 2024г:  
Регион           | Процентное соотношение
---------------- | -------------
Азия             | 38%
Европа           | 27% 
Латинская Америка| 21% 
Страны MENA      | 8%
Северная Америка | 3%
Южная и Центральная Африка     | 3% 

Отметим также, что **27% всех пользователей Telegram русскоязычные. То есть находятся в основном в постсоветских странах**.

Соответственно, география _глобальных датацентров_:

Регион | Локация датацентра(-ов)|
------ |------------------ |
Азия   | Токио, Сингапур |
Европа | Амстердам, Москва |
Латинская Америка | Сан-Паулу |
Страны MENA | Дубаи |
Северная Америка | Сан-Франциско |
Южная Африка | Йоханесбург |
Центральная Африка | Додома |

Примечание: В Китае **запрещен** Telegram и другие иностранные мессенджеры. Разрешен только WeChat

В каждом регионе предлагается распределить датацентры относительно [плотности пользователей](https://inclient.ru/telegram-stats/):

Регион | Процент серверов|
------ |------------------ |
Азия   | 40% |
Европа | 30% |
Латинская Америка | 21% |
Страны MENA | 4% |
Северная Америка | 1%|
Южная Африка | 1% |
Центральная Африка | 1% |

## Схема балансировки

1. В первую очередь определяем регион с помощью Geo-Based DNS
2. После чего рамках региона выбираем датацентр через BGP Anycast

# 4 Локальная балансировка нагрузки

L3:
L3 будет использоваться в качестве первичной балансировки, чтобы распределить трафик по серверам внутри ЦОД. Балансировка будет по схеме Virtual Server via IP Tunneling

![image](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/de3f99a0-3714-429c-b93a-7ac53e610c17)


Для обеспечаения отказоустойчивости системы стоит использовать framework *keepalived* благодаря следующим преимуществам:
* Производительность
* Открытость
* Простота настройки

Будем использовать **Virtual Router Redundancy Protocol(VRRP)**. Соответственно, мы получаем возможность отслеживать состояние узлов системы, а при отказе одного из них перенаправить трафик на другой узел.

L7:
На отдельно взятом сервере будет проходить балансировка с помощью L7. На этом уровне для балансировщика нагрузки важна поддержка gRPC. Будем использовать балансировщик NGINX благодаря следующим преимуществам:
* Лучшая производительность
* Поддержка различных протоколов(в т.ч. gRPC и HTTP)
* Открытость
* Гибкость настройки

Также для решения проблемы отказоустойчивости на этом уровне будет использоваться k8s с использованием:
* Liveness probe
* Readiness probe

Для шифрования будем пользоваться услугами центра [Let's encrypt](https://letsencrypt.org)
Для оптимизации установки соединения будет пользоваться Session cache

# 5 Логическая схема базы данных
## Схема базы данных

[Ссылка на логическую схему БД](https://dbdiagram.io/d/Highload-Gleb-Makarenko-DB-6600741fae072629ced3dd73)

![Highload Gleb Makarenko DB (3)](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/585f3771-e387-463a-b9c6-8f7b222e4d66)

## Схема S3

[Ссылка на схему S3](https://dbdiagram.io/d/Highload-Gleb-Makarenko-DB-6600741fae072629ced3dd73)

![S3 Highload Makarenko Gleb](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/917cc2e1-e65d-4f7c-a513-84a86060b5a2)


# 6 Физическая схема БД
Для хранения информации о сессиях используется база данных Redis по причине того, что она осуществляет хранение данных in-memory, имеет поддержку неблокирующей репликации master-slave и возможность организации кластера Redis cluster.

Для хранения очереди непрочитанных сообщений используется Apache Kafka.
Для хранения файлов будет использоваться сервер с большим объемом памяти.

Технологии для хранения данных:
Таблица(-ы) | Технология|
------ |------------------ |
sessions   | redis |
users, chats, channels, users_meta, channels_meta, users_channels, users_chats, subsribes_metric | PostgreSQL |
reactions, posts, comments, messages, attachments, subscribes_metric, post_views_per_post | MongoDB |
*_search | ElasticSearch |

Так как один сервер PostgreSQL и MongoDB не выдержит планируемую нагрузку, выполним шардинг таблицы сообщений. Для более быстрого доступа к данным будет необходимо использовать индексы. 

* Для таблиц users, chats, channels инекс по полю id
* Для таблиц channels_meta, user_meta индекс по полю channel_id и user_id соответственно.
* Остальное храним в MongoDB в виде ключ-значение.
* post_reactions, comments, views_per_post получаем по полю post_id
* attachments - id
* subscribes_metric, posts по полю channel_id
* messages - chat_id
* messages_reactions - message_id
* comments_reactions - comment_id

Также необходимо выполнить шардинг

* Для шардинга сообщений необходимо сделать шардинг таблиц messages по полю chat_id
* Для шардинга постов необходимо сделать шардинг posts, subscribes_metric по полю channel_id
* Для шардинга комментариев и просмотра постов необходимо сделать шардинг comments и post_views_per_user по полю post_id
* Для шардинга реакций необходимо сделать шардинг по полю *_id
* Также имеет смысл сделать шардинг таблиц users_channels и users_chats по полю channel и chat_id соответственно

В совокупности это даст следующий результат:

На одном сервере хранятся пользователи с метаданными, прикрепелнные файлы

На шардах для чатов хранятся чаты с id в диапазоне от х до y, соответствующие им сообщения, которым соответствуют реакции, а также таблицы для связи с пользователей и чатов

На шардах для каналов хранятся каналы с id в диапазоне от х до y, соответствующие им посты, которым соответствуют комментарии, которым соответствуют реакции, а также таблицы для связи пользователей и каналов

Прирост в скорости получения данных будет сильным за счёт относительно маленьких таблиц, индексов, а также хранения в виде пар ключ-значение

# 7 Алгоритмы
## Поиск
Для эффективного поиска людей, поиска по постам и сообщениям будем использовать движок ElasticSearch. Это поисковый и аналитический движок, который основан на технологии поиска по тексту.

Копии таблиц users, messages, channels, posts можно положить в хранилище ElasticSearch (в логической схеме user_search, message_search, channel_search, post_search). Оно предоставляет механизм для поиска, анализа и хранения данных в реальном времени.

* ```user```
   * ```phone_number``` - полное совпадение по нормализированному формату.
     
     Индекс: ```phone_number```
     
     Обращение к ХД: По ```phone_number```(в нормализованном формате)

* ```message```
   * ```content``` - частичное совпадение, допускаются различные формы слов
     
     Индексы(в порядке убывания важности): ```timestamp```, ```chat_id```
     
     Обращение к ХД: По ```content```, возвращаются все частичные совпадения

* ```post```
   * ```content``` - частичное совпадение, допускаются различные формы слов
   * ```channel_name``` - частичное совпадение
     
     Индексы(в порядке убывания важности):  ```timestamp```, ```channel_name```
     
     Обращение к ХД: по полю ```content```, возвращаются все частичные совпадения, либо по полю ```channel_name```, возвращаются все частичные совпадения

* ```channel```
   * ```name``` - частичное совпадение, допускаются различные формы слов
     
     Индексы(в порядке убывания важности): ```subscribers_count```, ```name```
     
     Обращение к ХД: по полю ```name```, возвращаются все частичные совпадения


Частичные совпадения для ```content``` в постах и сообщениях ищутся по следующей схеме:  
   Создается **wildcard** запрос, который ищет вхождения переданной строки в текст сообщения или поста

Частичные совпадения для ```channel_name``` и ```name``` ищутся по следующей схеме:
   Создается **query_string** запрос, который ищет совпадения префикса искомого имени с переданным значением

Поиск по сообщениям и постам будет отдавать приоритет наиболее новым

Поиск по номеру без приоритезации т.к. мы используем полное совпадение по номеру

Поиск канала будет отдавать приоритет каналам с наибольшим числом подписчиков. Но для случая, если искомое название канала полностью совпало с каким-либо, то этому результату дается вес 2. В случае частичного совпадения - дается вес 1. Итого: Первые в приоритете каналы с полным совпадением, отсортированные по убыванию количества подписчиков, после них -  с частичным совпадением по убыванию количества подписчиков.

Модерация будет осуществляться "вручную". При подаче жалобы пользователем и рассмотрением её модератором.

# 8 Технологии

## Frontend

**TypeScript**. TS позволяет отлавливать ошибки на этапе компиляции и делает работу с типами более детерминированной. При масштабировании проекта в случае использования JavaScript могут возникать ошибки, которые станет сложнее и дороже найти и исправить, чего можно избежать используя TypeScript.
TypeScript поддерживает новые функции ECMAScript, что позволяет разработчикам использовать новые возможности JavaScript до того, как они станут стандартом.
TS также поддерживает модульность по аналогии с JavaScript, что позволяет эффективно осуществлять поддержку, масштабирование и тестирование кода

**React**. Наиболее гибкий и популярный фреймворк для Frontend-разработки. С помощью него можно решить практически все необходимые задачи, а также относительно легко масштабировать проект за счёт компонентного подхода.
React - наиболее популярный фреймворк, следовательно будет наиболее просто найти разработчиков, владеющих им.
Также будет использоваться SCSS для более быстрой и удобной работы со стилями.
Необходимо также хранить уведомления, для чего будет достаточно SQLite - персистентное хранилище на клиенте.

## Мобильное приложение

Для реализации Frontend на мобильных устройствах будем использовать React Native.

Необходимо также хранить уведомления, для чего будет достаточно SQLite - персистентное хранилище на клиенте.

## Балансировщик

На уровне L7 будем также использовать nginx за счёт его высокой производительности и гибкости настройки. В целом, nginx подойдёт под микросервисную архитектуру.

## Backend
Для создания бизнес-логики будет использоваться Golang, за счёт многопоточности с большим количеством готовых библиотек. Telegram будет выполнен на микросервисной архитектуре

Для отслеживания проблем в микросервисной архитектуре будет использоваться Jaeger

Как на клиенте, так и на сервере для реализации Realtime-сообщений будет использоваться технология WebSocket. Данная технология позволит отправлять сообщения и уведомления без дополнительных запросов.

## Кеш
В качестве кеш хранилища взяли Redis. Имеет больше функциональных возможностей по сравнению с аналогами (например, memcached) - поддержку транзакций, очереди, а также поддерживает репликацию master-slave и кластеризацию с версии 3.0.

## БД

В проекте присутствуют 2 основные СУБД - MongoDB и PSQL
Хранение осуществляется следующим образом

Таблицы | Технология|
------ |------------------ |
users, chats, channels, users_meta, channels_meta, users_channels, users_chats, subsribes_metric | PostgreSQL |
reactions, posts, comments, messages, attachments, subscribes_metric, post_views_per_post | MongoDB |

**PostgreSQL** был выбран за счёт следующих преимуществ:
* Открытость - это полностью бесплатная СУБД с открытым исходным кодом
* Масштабируемость - отлично подходит для большого объема данных, поддерживает репликацию
* Популярность - имеет большое сообщество разработчиков
* Надежность и стабильность - обладает механизмами сохранения целостности данных, транзакционной безопасности и отказоустойчивости

**MongoDB** был выбран за счёт следующих преимуществ:
* Скорость – специфическая структура и поддержка индексации хорошо сказываются на производительности системы, так что её имеет смысл использовать там, где скорость имеет критическое значение;
* Удобство масштабирования – можно без особых сложностей масштабировать и менять в зависимости от потребностей
* Функция работы на нескольких серверах – MongoDB использует механизмы репликации и сегментирования, позволяющие формировать функциональные копии БД, которым можно в любой момент делегировать управление, и равномерно распределять между ними нагрузку.
* Популярность - имеет большое сообщество разработчиков

## Брокер сообщений

Брокер сообщений представляет собой тип построения архитектуры, при котором элементы системы «общаются» друг с другом с помощью посредника. Благодаря его работе происходит снятие нагрузки с веб-сервисов, так как им не приходится заниматься пересылкой сообщений: всю сопутствующую этому процессу работу он берёт на себя.

Для этого подойдет Kafka

## Метрики

В качестве СУБД для сбора и анализа метрик возьмем ClickHouse. Поддерживает распределенное выполнение запросов. Использует движок MergeTree, что позволяет с высокой производительностью выполнять аналитические запросы, поддерживает индексирование, репликацию и партиционирование данных, горизонтальное масштабирование.

## Поиск

Для поиска используем ElasticSearch. Это - хранилище данных для поиска, анализа и хранения в реальном времени. Очень хорошо горизонтально масштабируется добавлением новых узлов в кластер.

## Хранилище файлов

Для сервиса-мессенджера я бы выбрал Amazon Simple Storage Service (S3) Glacier Instant Retrieval хранилище.

Amazon S3 Glacier Instant Retrieval предлагает:

* Мгновенное получение данных: Доступ к данным в течение нескольких миллисекунд, как и в случае с S3 Standard.
* Низкая стоимость: Уровень хранения Glacier Instant Retrieval дешевле, чем S3 Standard, что делает его экономичным вариантом для больших объемов неактивных данных.
* Высокая доступность: Данные хранятся в нескольких центрах обработки данных по всему миру, что обеспечивает высокую доступность и защиту от сбоев.
* Масштабируемость: S3 Glacier Instant Retrieval может масштабироваться для хранения петабайтов данных.

Для сервиса-мессенджера, который обычно содержит большие объемы сообщений, изображений и других вложений, важны мгновенное получение данных, низкая стоимость и высокая доступность. S3 Glacier Instant Retrieval соответствует всем этим требованиям, что делает его идеальным выбором для хранения данных такого сервиса.

## Медиа

Фото:
Для сжатия фото будем использовать **WebP**. Сжатие без потерь

Видео:
Будем использовать **VP9**. Сжатие с потерями

Аудио:
Будем использовать **Opus**. Сжатие с потерями

Также будет использоваться адаптивное сжатие, которое регулирует уровень сжатия в зависимости от скорости соединения и типа данных. Это позволяет оптимизировать размер файла и качество передачи для различных условий сети.

# 9 Схема

Легенда:

![image](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/718b91f4-5c33-4f52-a99f-d3a975e669bc)

![image](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/f67163a2-e536-48a6-98e7-70aabce68d59)

![image](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/515be32d-4faf-4279-8e82-7a017c5a32f1)

Будет использоваться API Gateway для обеспечения изолированности логики различных клиентов и централизованной точки маршрутизации. На Gateway также будет происходить балансировка между сервисами, т.к. архитектура будет разделена на микросервисы для изоляции бизнес-логики.

* Сервис auth будет отвечать за авторизацию и аутентификацию
* Сервис user будет отвечать за бизнес-сущность пользователя: его статус, профиль, язык, данные профиля и т.д.
* Сервис messenger будет отвечать за чаты, сообщения и взаимодействия между ними. Этот сервис может отправлять данные на клиент для реализации Realtime.
* Сервис channels будет отвечать за каналы, их аналитику, управление правами и данными(аватарка, описание и т.д.) у каналов
* Сервис posts будет отвечать за посты, их аналитику, файлы и т.д. Этот сервис может отправлять данные на клиент для реализации Realtime.
* Сервис comments будет отвечать за работу с бизнес-сущностью комментария к посту. Этот сервис может отправлять данные на клиент для реализации Realtime.
* Сервис Files Manager отвечает за обработку и запись файлов. Будет 1 инстанс
* Сервис DB Manager отвечает за предобработку и запись в БД(Mongo или Postgres). Будет 1 инстанс
* Сервис Stats Manager отвечает за обработку и запись статистики в ClickHouse и PSQL/Mongo. Будет 1 инстанс
* Сервис Search Manager отвечает за обработку и запись в ElasticSearch. Будет 1 инстанс

Для чтения и записи в БД будет реализован паттерн CQRS, где мы будет разделять запись и чтение на 2 пайплайна. Запись будет производиться асинхронно через kafka, а чтение - синхронно напрямую с сервера БД.
Также синхронно будут производиться пайплайны чтения и записи файлов в S3.
# 10 Отказоустойчивость

Будут резервироваться датацентры для того, чтобы в случае отказа одного была возможность либо перенаправить трафик, либо восстановить данные из другого

Компонент | Метод| Обоснование|
------ |------------------ |---------|
nginx | резервирование ресурсов, резервирование серверов | т.к. балансировщик - это узкое место системы, то в случае высоких нагрузок понадобятся дополнительные ресурсы для обеспечения отказоустойчивости. Также отказ сервера балансировщика означает отсутствие доступа к сервису у пользователей, поэтому необходимо зарезервировать дополнительные сервера| 
MongoDB | репликация | Репликация БД необходима для сохранения данных в случае отказа работы одного из серверов БД, а также для обеспечения более высокой доступности и равномерного распределения нагрузки|
PSQL | репликация | --"--|
S3 | репликация | --"--|
Buiseness-logic(ex: User)| Резервирование компонентов, резервирование ресурсов| В случае высоких нагрузок на сервера, необходимо для обеспечения отказоустойчивости зарезервировать сервера, а также компоненты, чтобы в случае отказа чего-либо(что происходит часто) оперативно восстановить работу сервера. При высоких нагрузках серверу потребуются дополнительные ресурсы для своевременной обработки данных.|
API Gateway| Резервирование логики| Т.к. на шлюз поступает много запросов, требующих тривиальной логики, то имеет смысл реплицировать эту логику для повышения доступности системы|
Kafka| Репликация| Для Kafka очень важно использовать репликацию, чтобы обеспечить отказоустойчивость и сохранность данных в случае отказа брокера. Балансировка + репликация позволят гаранитровать, что минимум 1 брокер будет работоспособен|
ElasticSearch| Репликация(индексов)| ElasticSearch предоставляет возможности для репликации индексов, что обеспечивает отказоустойчивость и возможность восстановления данных при сбоях.|
ClickHouse| Репликация| Для обеспечения сохранности данных, а также доступности и равномерного распределения нагрузки|
Redis| Репликация| Будет использоваться репликация для обеспечения работы в случае отказа главной ноды Redis. Репликация - master-slave с 2 репликами|

Для обеспечения безопасного релиза новый версий приложения будет использоваться методика Graceful Shutdown

Для обработки отказов определенных компонентов будет использоваться Graceful Degradation. Для обеспечения Graceful Degradation на клиенте будет использоваться локальный кеш и SQLite, на сервере - ограничение функционала.

EX:
* При остановке работы сервиса сообщений, сообщения будут подгружаться из SQLite и локального кеша, что позволит просматривать сообщения и вложения без возможности отправки новых. Аналогично для постов и комментариев
* При остановке работы сервиса, работающего с данными пользователя(профиль), данные подгружаются из SQLite и кеша
* При слишком большой нагрузке на сервер, количество перезапросов уменьшается, а после - происходит отключение от потока запросов
* При слишком большой нагрузке на мессенджер/посты/комменты происходит пересылка сообщений сначала с файлами худшего качества, после - без файлов.

Faliover Policy: 
* Frontend - будут использоваться перезапросы, а также использование оффлайн логики
* Backend - будует использоваться уменьшение посылаемых запросов на проблемный хост и circuit breaker. В случае слишком большого числа запросов мы либо игнорирем запросы(в случае если с одного IP идёт слишком много), либо намерено замедляем их обработку(если слишком большое количество запросов с разных IP адресов). Circuit Breaker будет реализован с помощью nginx.

Observability:
* Мониторинг
* Алертинг
* Логирование

Асинхронные паттерны:
CQRS - т.к. запросы будут частые, будем использовать данный паттерн. Он поможет ограничить нагрузку на запись

# 11 Расчёт ресурсов

## Сервера
Сервис имеет в пике 642 984 RPS и 249.5 ГБ/сек сетевого трафика

На официально сайте nginx видно, что сервер с 16 CPU видерживает 77 427 HTTP RPS
![image](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/1d2f2110-c118-4d53-9d6d-f765da218053)

642 984/77 427 = 8.3. С округлением до целого вверх получаем 9 серверов.

Учитывая объем сетевого трафика, посчитаем количество серверов с учетом сетевого трафика: 249.5 / 40 = 6.2. С учетом округления вверх - 7

Тогда получаем следующую конфигурацию для nginx

Параметр | Значение|
------ |---------- |
CPU    | 16        |
Network| 40 ГБит/с |
Кол-во | 9         |

Микросервисы:

Сервис | RPS| CPU| RAM| Net|
------ |---------- |--- |--- |--- |
Messenger    | 429 875 | 4300 | 430Гб | 166 ГБ/с |
Auth | 2 122 | 22 | 3 ГБ | 3 МБит/с |
Channels | 22 | 1 | 1МБ | 3 МБит/с |
Posts |2 132 | 22 | 3 ГБ | 3 ГБит/с |
Comments | 208 515 | 2100 | 220 ГБ | 81 ГБит/с |
Users | 318 | 3 | 512 МБ | 512 МБит/с |

Расчёты сделаны из предположения 1 ядро CPU на 100 и 1 ГБ RAM на 1000 запросов

Соотношения 
* посты+комменты+каналы : сообщения+чаты = 1:2
* посты:комменты = 1:100
* каналы:посты = 1:100

Сервис | Хостинг| Конфиг| Count|
------ |---------- |---------- |--- |
nginx  | У себя    | 16 core CPU; 4x8 GB RAM 2400 MHz DDR4 | 9 |
Auth   | У себя    | 12 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 2 |
Users  | У себя    | 4 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 1 |
Channels  | У себя | 2 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 1 |
Posts  | У себя    | 12 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 2 |
Comments  | У себя | 16 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 132 |
Messenger | У себя | 16 core CPU; 1х4 GB RAM 2400 MHz DDR4 | 264 |

Возьмем минимум два сервера на случай отказа одного из них.

Контейнеры:

1. nginx:
- CPU: 2 core
- RAM: 4 GB

2. Auth:
- CPU: 1 core
- RAM: 2 GB

3. Users:
- CPU: 1 core
- RAM: 1 GB

4. Channels:
- CPU: 1 core
- RAM: 1 GB

5. Posts:
- CPU: 1 core
- RAM: 2 GB

6. Comments:
- CPU: 4 core
- RAM: 8 GB

7. Messenger:
- CPU: 4 core
- RAM: 8 GB

## БД
**Amazon S3**

Один бакет может хранить до 5 Тб, с учетом динамического роста данных за счет фотографий и голосовых сообщений из пункта 2, получим: 1689 ПБ / 5 ТБ = 337 848 бакетов в год для хранения новых данных.

**PSQL** 

С учетом динамического роста данных за счет текстовых сообщений из пункта 2, придется каждый год докупать новые диски для хранения 0.24 ПБ.

PostgreSQL способен выдавать около 350 000 TPS на 4 серверах на простых запросах.

Т.к. для PSQL имеются только простые запросы на 4 сервисах, то нам хватит кластера из 20 серверов(4 master + 16 slave). Необходимо сделать реплики. Тогда количество серверов будет равно 40.

Конфигурация:
CPU | RAM| ROM | Количество|
------ |---------- |---- |---------- |
24    | 512        | 5 ТБ | 40 |

**Redis**
Redis Согласно официальному сайту Redis, один сервер способен выдержать 50 000 TPS.

Является in-memory хранилищем, следовательно требуется большой объём оперативной памяти.

Ежедневный онлайн составляет около 55 200 000 пользователей. Если учесть, что все они будут одновременно пользоваться нашим сервисом, то потребуется token(64 байта) + id (4 байта) + user_id (4 байта) = 72 байта на одного пользователя. С учетом коэфициента запаса 2 на дополнительные поля, которые могут возкникнуть в нашей структуре, возьме объем в 216 байт Тогда для 55 200 000: 11.10 ГБ. Для более эффективной обработки и повышения надежности можно сделать 1 master и 2 slave сервера с репликами. Всего 6 серверов.

Конфигурация:
CPU | RAM| ROM | Количество|
------ |---------- |---- |---------- |
2    | 32       | 20 ГБ | 6 |

**Mongo**
Согласно [исследованию](https://info.enterprisedb.com/rs/069-ALB-339/images/PostgreSQL_MongoDB_Benchmark-WhitepaperFinal.pdf), MongoDB ~ в 10 раз медленнее, чем PSQL, тогда предположим, что Mongo выдерживает 35 000 на простой логике и 10 000 на сложной на кластере из 4 серверов.

Сложная логика: Posts, Comments. Тогда необходимо покрыть 210 647 RPS

Простая логика: Messenger. Тогда необходимо покрыть 429 875 RPS

Для Posts+Comments необходим кластер из 24 серверов(master) + 48(slave) + 72(реплика)
Для Messenger необходим кластер из 12 серверов(master) + 24(slave) + 36(реплика)

CPU | RAM| ROM | Количество|
------ |---------- |---- |---------- |
24    | 512        | 5 ТБ | 196 |

# Список источников

1. https://tgstat.ru/research-2023

2. https://www.demandsage.com/telegram-statistics/

3. https://www.businessofapps.com/data/telegram-statistics/

