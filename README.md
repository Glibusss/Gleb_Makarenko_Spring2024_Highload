# Проектирование высоконагруженного сервиса обмена сообщениями

# 1. Тема и целевая аудитория

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

# Глобальная балансировка нагрузки

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

# Локальная балансировка нагрузки

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

# Логическая схема базы данных
## Схема базы данных

[Ссылка на логическую схему БД](https://dbdiagram.io/d/Highload-Gleb-Makarenko-DB-6600741fae072629ced3dd73)

![Highload Gleb Makarenko DB (2)](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/fbe81611-2627-4633-bbcd-c783fc65ddef)

## Схема S3

[Ссылка на схему S3](https://dbdiagram.io/d/Highload-Gleb-Makarenko-DB-6600741fae072629ced3dd73)

![S3 Highload Makarenko Gleb](https://github.com/Glibusss/Gleb_Makarenko_Spring2024_Highload/assets/113942267/917cc2e1-e65d-4f7c-a513-84a86060b5a2)


# Физическая схема БД
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

# Алгоритмы
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

# Технологии

## Frontend

**TypeScript**. TS позволяет отлавливать ошибки на этапе компиляции и делает работу с типами более детерминированной. При масштабировании проекта в случае использования JavaScript могут возникать ошибки, которые станет сложнее и дороже найти и исправить, чего можно избежать используя TypeScript.
TypeScript поддерживает новые функции ECMAScript, что позволяет разработчикам использовать новые возможности JavaScript до того, как они станут стандартом.
TS также поддерживает модульность по аналогии с JavaScript, что позволяет эффективно осуществлять поддержку, масштабирование и тестирование кода

**React**. Наиболее гибкий и популярный фреймворк для Frontend-разработки. С помощью него можно решить практически все необходимые задачи, а также относительно легко масштабировать проект за счёт компонентного подхода.
React - наиболее популярный фреймворк, следовательно будет наиболее просто найти разработчиков, владеющих им.
Также будет использоваться SCSS для поддержки CSS-модулей, что упрощяет работу со стилями за счёт более атомарной работы с файлами стилей

## Балансировщик
В качестве баланировщика L4 используем nginx. Благодаря нему трафик будет эффективно балансироваться по разным серверам.

На уровне L7 будем также использовать nginx за счёт его высокой производительности и гибкости настройки. В целом, nginx подойдёт под микросервисную архитектуру.

## Backend
Для создания бизнес-логики будет использоваться Golang, за счёт многопоточности с большим количеством готовых библиотек. Telegram будет выполнен в формате микросервисной архитектуры. Она повышает отказоустойчивость и разделяет сервис на логические блоки.

В качестве микросервисов можно выделить:

Auth - Авторизация
Messenger - Мессенждер
Channels - Каналы
User - Профиль пользователя + Регистрация
Comments - Комментарии
Search - Поиск
Analithycs - Аналитика

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



# Список источников

1. https://tgstat.ru/research-2023

2. https://www.demandsage.com/telegram-statistics/

3. https://www.businessofapps.com/data/telegram-statistics/

