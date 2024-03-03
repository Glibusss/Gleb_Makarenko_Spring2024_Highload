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
Северная Америка | 6% 

Отметим также, что **27% всех пользователей Telegram находятся в России**.

Соответственно, география _глобальных датацентров_:

Регион | Локация датацентра(-ов)|
------ |------------------ |
Азия   | Гонконг, Сингапур |
Европа | Амстердам, Москва |
Латинская Америка | Сан-Паулу |
Страны MENA | Дубаи |
Северная Америка | Сан-Франциско|

В каждом регионе предлагается распределить датацентры относительно [плотности пользователей](https://inclient.ru/telegram-stats/):

Регион | Процент серверов|
------ |------------------ |
Азия   | 40% |
Европа | 30% |
Латинская Америка | 21% |
Страны MENA | 5% |
Северная Америка | 4%|

## Схема балансировки

1. Определяем регион через Geo-based DNS
2. В рамках региона выбираем датацентр через BGP Anycast


# Список источников

1. https://tgstat.ru/research-2023

2. https://www.demandsage.com/telegram-statistics/

3. https://www.businessofapps.com/data/telegram-statistics/

