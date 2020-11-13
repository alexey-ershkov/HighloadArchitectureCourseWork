# HighloadArchitectureCourseWork

## 1. Выбор темы
Аналог современной торговой биржи, например [Московская биржа](https://www.moex.com/) или [Nasdaq](https://www.nasdaq.com/)

##  2.  Определение возможного диапазона нагрузок

Рассмотрим особенности расчета нагрузки на биржу. Основная нагрузка идет на бирже на торгово-клиринговую систему (ТКС), через которую проходят транзакции, заявки и сделки.
 
Можно сделать вывод, что если мы рассмотрим нагрузки на ТКС валютного, фондового и срочного рынка, то тем самым рассмотрим основную нагрузку на биржу. 

В качестве аналога, относительно которого будем рассматривать нагрузку - Московская Биржа. Особенность биржи в том, что часть данных о нагрузке и отказоустойчивости есть в открытом доступе, так как это важно для привлечения клиентов. 

Для определения возможного диапазона нагрузок и дальнейших расчетов воспользуемся следующими материалами:
- [Результаты нагрузочного тестирования московской биржи в от 30 марта 2019 года](https://www.moex.com/n23621/?nt=0)

### Нагрузка на ТКС фондового рынка

![](./media/ТКС_ФР.png)

![](./media/ТКС_ФР_График.png)	

Участвовало 14 банков

### Нагрузка на ТКС валютного рынка

![](./media/ТКС_ВР.png)

![](./media/ТКС_ВР_График.png)

Участвовало 18 банков

### Нагрузка на ТКС срочного рынка

![](./media/ТКС_СР.png)

![](./media/ТКС_СР_График.png)

Участвовало 15 банков

### Следовательно суммарная нагрузка на биржу за 2.5 часа нагрузочного тестирования

В день:

|Транзакции|Заявки|Сделки|
|----------|------|------|
|522 058 763|269 974 995|9 368 033|

В секунду:

|Транзакции|Заявки|Сделки|
|----------|------|------|
|299 720|66 355|4 474|

##### Примечание: 

 В отчете о нагрузочном тестировании оговаривается то, что реальная нагрузка в боевых условиях ниже, чем та, что была представлена в нагрузочном тестировании

## 3. Выбор планируемой нагрузки
Будем считать, что нагрузка на биржу от каждого клиента в среднем одинакова. 

В среднем при оценке нагрузки участвовало 16 банков. Следовательно с одного клиента приходилась нагрузка:

В секунду:

|Транзакции|Заявки|Сделки|
|----------|------|------|
|18 732,5|4 147,19|279,625|

Исходя из тех же результатов нагрузочного тестирования, по итогам рекомендуется использовать для обычного протокола канал в 10 Мбит/с 

Зная нагрузку в секунду посчитаем приблизительный размер пакета:

```
(10 x 1024 x 1024) / (18 732,5 + 4 147,19 + 279,625) ~ 450 бит
```

Будем расчитывать на нагрузку для [100 крупнейших банков Европы](https://www.invest-rating.ru/financial-forecasts/?id=7903)
Тогда нагрузка в секунду будет:

|Транзакции|Заявки|Сделки|
|----------|------|------|
|1 873 250|414 719|27 962,5| 


```
(1 873 250 + 414 719 + 27 962,5) x 450 = 1 042 169 175 бит/с = 1,04 Гбит/с
```

Исходя из [данных](https://www.moex.com/s223) максимальное время торгов 14 часов 20 минут (860 минут), тогда нагрузка в день со всех клиентов считается по формуле: 

```
(Нагрузка в секунду) * 860 * 60

 1,04 Гбит/с * 860 * 60 = 53 664 Гбит
 ```
 
## 4. Логическая схема Базы Данных
![](./media/ExhangeDB.png)

## 5. Физические системы хранения
Выделим основные особенности нагрузки на СУБД Биржи: 
- Часто пишем
- Часто читаем
- Данные нельзя потерять
- Задержка должна быть минимальной

Так как мы часто пишем и читаем - нам подойдет обычная SQL СУБД. Возьмем Postgres 12, так как она хорошо поддерживается сообществом, имеет хороший встроенный механизм репликации, а также хорошие инструменты шардинга (например Postgres XL). 

Также во время торгов нам нужно обеспечить минимальную задержку. Для этого мы будем использовать in-memory хранилище tarantool, в котором будем хранить данные **только текущих торгов**, так как это одни из самых "горячих" данных данного сервиса.

Из плюсов мы получим довольно быстрый ответ от бд, с другой стороны нам нужно сделать две операции записи - в tarantool и в Postgres. 

Так как объем данных довольно большой - нужно подумать о том, как мы будем шардить наши данные между серверами СУБД. Шардинг будем производить с помощью хеш-функции от имени банка, совершающего операцию и количества серверов, что позволит нам распределить наши данные равномерно по серверам и не переполнить шарды. 

Ото всех клиентов нагрузка будет (из таблицы транзакции, сделки и заказы в секунду): 

```
1 873 250 + 414 719 + 27 962,5 = 2 315 931,5 RPS
```

Пусть в среднем на один сервер будет приходиться 5000 RPS, тогда нам понадобиться: 

```
2 315 931,5 RPS / 5000 RPS = 464
```

464 сервера с Postgers 12 + 464 сервера с Tarantool. Для каждого сервера Postgres добавим по одной реплике, чтобы не потерять данные в случае отказа. С реплик будем читать, а на матера писать. Что дает нам еще плюс 464 сервера. Итого нам понадобиться 1392 сервера для хранения и работы с данными. 

## 6. Выбор прочих технологий: языки программирования, фреймворки, протоколы взаимодействия, веб-сервера и т.д.

### Frontend
В нашем случаем клиентом будет пользовательский браузер. Так как браузер исполняет только JavaScript код, то наш клиент будет использовать нативную для браузера технологию в связке с HTML и CSS для отображения данных пользователю. 

**Технологии**
- Javascript
- HTML
- CSS

### Backend
Наш бэкенд будет иметь микросервисную архитектуру. Это позволит нам без особых усилий масштабировать конкретный сервис, а не всю систему. Также если упадет один из сервисов - не упадает вся система. Главной причиной почему нам необходимо использовать микросервисную архитектуру - это возможность разработки сервисов на разных языках. 

Сервис торгов должен давать максимальную производительность, следовательно нам нужен однозначный контроль ресурсов для данного сервиса. Значит мы не можем выбирать языки с Garbage Collector. Нашим выбором для него будет однозначно язык C++. Разработка на С++ дорога и сложна. 

Поэтому для остальных сервисов мы будем использовать язык Golang, так он специально разрабатывался как язык для написания серверов. Имеет встроенный механизм горутин, позволяющий нам просто и эффективно писать код. Также встроенный кодстайл "из коробки", который нам будет полезен в командной разработке. 

**Технологии**
- С++
- Golang
- Микросервисная архитектура

### Протоколы общения
Для общения между клиентом и сервером выберем протокол HTTP2, который обеспечит нам быстрое общение с сервером благодаря двунаправленности установленного соединения, а также благодаря тому, что протокол бинарен. 

Также с целью ускорения общения между клиентом и сервером будем использовать protobuf, так его размер меньше аналогичных форматов (json, xml), а скорость парсинга выше из-за заранее скомпилированный структур данных. Protobuf поддерживает много языков, в том числе C++, Golang и Javascript. Также несомненным плюсом protobuf является генерация документации на основе структур, что позволит проще обеспечивать качество в разработке нашего продута. 

Для общения между микросервисами будем использовать gRPC - протокол над HTTP2, который позволяет удаленно вызывать функции на других серверах. Так же он для общения использует по умолчанию Protobuf, что является несомненным плюсом к скорости общения между сервисами. 

**Технологии**
- HTTP2
- Protobuf
- gRPC


### Балансировка нагрузки и отдача статики

Для балансировки нагрузки и отдачи статики будем использовать Nginx, так как он зарекомендовал себя на больших нагрузках в современных высоконагруженных сервисах. Он поддерживается сообществом и имеет открытый исходный код, что позволяет нам собрать его под наши нужды, под него можно писать свои модули, а также использовать модули других разработчиков. Все это добавляет гибкости к производительному инструменту, что является несомненным плюсом для нас, как для разработчиков. 

**Технологии**
- Nginx