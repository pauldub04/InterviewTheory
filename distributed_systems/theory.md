**Theory of fault-tolerant distributed systems**
# Модель распределенной системы
## Зачем нужна распределенность
Современный мир без распределенных систем сложно себе представить

Виды:
* KV Store - гигантская хеш таблица (упорядоченная)
    * Set(k, v)
    * Get(k)
* File system
* Coordination Service - распределенные системы, которые нужны, чтобы строить другие системы
    * Например, ~Atomics
* Message Queues
* Databases

Зачем нам делать что-то распределенным?
1) В одну машину все не помещается
2) Распределенные системы - это не только про большие данные. Одна машина может отказать, система должна быть отказоустойчива
3) Мы не хотим доверять каким-то машинам, одной или несколько, они могут быть злоумышленниками

Для того, чтобы строить алгоритмы, мы должны иметь модель мира, в котором мы собираемся работать

## Общая модель: узлы, каналы, отправка сообщений, клиенты, истории, модель согласованности
Что такое распределенная система?
В отличие от алгоритмов, которые что-то сортируют или ищут, мы от модели ждем каких-то гарантий
В коде клиента система выглядит как переменная, и мы выполняем над ней какие-то операции. Мы не единственные, кто так делаем, нас таких много. Внутри системы удобно говорить про **модель MessagePassing**, а снаружи разумно использовать **модель SharedMemory**

**History**
![alt text](images/1.png)

## Моделирование сети
### Гарантии доставки
Что может с сообщением случиться? У нас есть абстракция надежного канала, мы в него отправляем сообщение из узла *a* в узел *b*, и узел *b* даже начинает что-то получать и сообщение обрабатывать, и вдруг соединение рвется (например, кто-то задел провод), и все гарантии в связи с этим разрывом пропадают
Соединения либо гарантируют нам доставку, либо сообщат, что оно порвалось
![alt text](images/2.png)
### Асинхронность и частичная синхронность
**Synchronous model**:
Есть константа $\delta$
$ delay(m) \le $ $\delta$ - задержка при доставки сообщения ограничена дельтой
Хорошая сеть может работать очень быстро, доставляя сообщение между двумя узлами за миллисекунду, почему бы не взять за дельту пару миллисекунд?
Такое бывает, но не всегда. Мы можем ожидать, что сеть работает быстро в каких-то частях нашего алгоритма, но мы хотим, чтобы наш алгоритм был устойчив к тому, что эта гарантия возьмет и нарушится, потому что мы физический мир не можем заставить так работать. Может быть куча случаев ошибки извне.

**Asynchronous model**:
Если наш алгоритм будет работать в таких предположениях, не ожидая верхней границы, то он сможет работать в реальном мире
* **Safety** свойство (минимальная задача алгоритма): система не должна уходить в плохие состояния, т.е. система не должна делать что-то плохое
* **Liveness** свойство: система должна делать что-то хорошее

**Partial synchrony**
У нас жизнь устроена так: есть ось времени, и некоторое время мы живем в асинхронной модели, тогда все плохо. А потом в какой-то волшебный момент __t*__ у нас эта гарантия начинает соблюдаться
![alt text](images/3.png)
## Партишены и split brain
**Partition** - это явление, когда сеть раскалывает кластер на две части, так, что связность в пределах одной части сохраняется, но линки, которые пересекают эти партишены, перестают работать. В итоге наша система раскалывается на две части, и хуже того, она еще и клиентов раскалывает
**Split brain** - ситуация, когда система в случае партишена начинает работать независимо в двух частях
![alt text](images/4.png)
## Моделирование узлов, сбои узлов
Каждый узел - это автомат или актор, который работает однопоточно и принимает сообщение из сети, и вызывает какой-то обработчик, и когда этот обработчик вызван, узел меняет свое внутреннее состояние, принимает какое-то новое, и, возможно, отвечает какими-то новыми сообщениями

Можно представить себе, что реальная программа исполняется на пуле потоков
Можно представить себе, что вот эти все файберы и фьючеры бегут на экзекуторе, которой по пинку выполняет очередную задачу

* Crash
* Restart
* Византийские отказы

## Время, часы и их применения
Есть какая-то ось времени, но доступа ко времени мы не имеем. У нас есть часы. В теории, часы - это такая функция, которая по времени возвращает нам некоторое значение (в идеале возвращает t, тождественная функция)
Часы - это объект физического мира. И, как и все в физическом мире, они несовершенны
А как мы собираемся использовать время в алгоритме?

У нас есть now(), есть t1 и t2, мы можем сделать t1 < t2 (сравнивать показания времени) и t2 - t1 (измерять интервал времени).
Зачем нам нужно измерять интервал времени? Для timeout, для failure detection

Зачем нам сравнивать мгновенные показания времени? Для того, чтобы упорядочивать собычтия
У нас есть Петя, и он сделал запись X, и получил подтверждение. Затем Петя сказал Васе про изменение. А Вася, вместо того, чтобы читать, сделал новую запись Y. Запись X случилась до записи Y, и мы ожидаем, что если после всего этого пойти в систему и спросить "Что лежит по ключу K?", то она вернет Y. Для этого система должна понять, что в нее записали X и записали Y, и при этом запись Y - это более свежая запись. Как бы мы могли это достичь? Пусть узел, который получил запись X, присвоил ей временную метку - показание локальных часов: 12:00. А после записи Y другая машина присвоила записи временную метку 11:59. Потому что часы разное время показывали. Получилось, что для системы X произошел раньше, чем Y

**Scew** - рассинхронизированные часы

**Drift** - часы могут тикать иногда быстрее, иногда медленнее, чем идеальные часы

![alt text](images/5.png)

## Синхронизация часов, нижняя оценка
Представим очень простой мир, где есть два узла. И узел n1 хочет свести свои часы с узлом n2
![alt text](images/6.png)
Пусть у нас есть ограничение на мир: любое сообщение доставляется в таком промежутке времени: $delay(m) \in [$$\delta$$-u,$$\delta$$]$, где u - uncertainty , в то время как drift никакого нет
Синхронизировать часы (в такой модели) - это значит узлы должны выбрать своим часам какую-то поправку: $sc_i(t) = c_i(t) + o_i$
Кто-то в таком мире принес нам алгоритм и говорит: "Его можно запустить на двух узлах и через какое-то время можно подобрать такие поправки, что: $|sc_i(t) - sc_j(t)| \le $ $\varepsilon $ "
Насколько маленьким может быть этот эпсилон? Рассуждения демонстрируют общую технику, которая будет использовать дальше:
Мы построим два исполнения, и они будут разными, которые, во-первых, будут различными, т.е. будут давать разные поправки в двух исполнениях, а во-вторых, алгоритм на двух узлах не смогут эти два исполнения отличить друг от друга, поэтому они выберут одинаковые поправки. И на этой коллизии у нас построится нижняя оценка

Мы управляем сетью и у нас есть два узла: $n_1$ и $n_2$. Пусть у нас от узла $n_1$ к узлу $n_2$ сообщения идут максимально долго насколько могут, а в обратную сторону максимально быстро. Мы строим такую сеть и алгоритм работает и синхронизирует часы. А теперь мы сделаем операцию, которая называется сдвигом - **shift**. Мы все события, которые происходили на узле $n_1$, оставляем на тех же местах, но переворачиваем сеть: т.е. теперь сообщение от узла $n_1$ к узлу $n_2$ летят очень быстро, а обратно очень медленно. Что произошло? Для узла $n_1$ ничего не произошло вообще, он не чувствует разницы. А для узла $n_2$ событие съехало немного назад, но он же не тупой и может это понять: в первом случае он получил в 12:00, а во втором случае в 11:59. Во втором исполнении переведем часы второго узла на $+u$. Утверждается, что хоть и во времени все съехало, но у $n_2$ нет возможность это почувствовать, потому что времени он не знает, он знает только то, что показывают ему часы, а часы показывают ему то же самое. Что мы получили? Мы запустили алгоритм в первом и втором исполнении, он синхронизировал часы. И в обоих случаях часы, синхронизированные для $n_1$ вообще не отличаются. Посмотрим на $sc_1(t*)$. Они принадлежат от -eps до +eps. А теперь подумаем, как работают синхронизированные часы для $n_2$. Алгоритм не чувствует разницы на узле $n_2$ между двумя исполнениями, а значит он выберет одну и ту же аддитивную поправку, но как бы сами часы отличались на $u$, т.е. синхронизированные часы узла $n_2$ в этих двух исполнениях должны отличаться ровно на $u$, и при этом они оба должны отличаться не более, чем на eps, чем $sc_1(t*)$. Т.е. $\varepsilon$ $\ge u/2$
![alt text](images/7.png)
В случае n узлов оценка немного меняется: $\varepsilon$ $\ge (1 - 1/n)u$
## GPS и синхронизация часов
У нас беда с часами - мы не можем оценить асимметрию в сети, мы не можем оценить время round trip'а, а может быть иногда у нас нет этого round trip'a, у нас коммуникация односторонняя, и время доставки сообщения мы знаем довольно таки неплохо

Как найти нас на плоскости
![alt text](images/8.png)

Добавим реализма: мы вышли в лодке в туман рыбачить. У нас есть карта и часы. На карте отмечено 3 маяка. Мы знаем, где они находятся, но не знаем, где мы находимся. Но мы знаем, что маяки раз в 5 минут издают какую-то пронзительную и уникальную сирену. И тогда наша система становится вот такой уже:
![alt text](images/9.png) - навигационное уравнение GPS
4 координаты, 4 уравнения, 4 спутника
Система GPS нас позиционирует не только в пространстве, она нас позиционирует по времени, и чтобы позиционировать в пространстве, ей нужно синхронизировать часы на ресивере с часами в космическом сегменте спутника

## Google Cloud Spanner, Google TrueTime, ожидание вместо коммуникации
Непонятно, откуда тут возник GPS
Почти все те системы, про которые было написано выше, впервые написали в гугле.
Spanner - это одна из систем, которая предоставляет гугл. Это геораспределенная база данных, и чтобы ее написать, инженерам гугл понадобился новый подход к часам. Они используют часы в своих алгоритмах, но не now(), потому что доверия к ней нет. Они построили сервис - [TrueTime](https://cloud.google.com/spanner/docs/true-time-external-consistency). Как все это устроено? Мы можем на каждой машине спросить:

    TT.Now() -> [e, l]
    // e - earliest
    // l - latest
И система TrueTime гарантирует, что настоящее время внутри этого интервала
Чуть формальнее: если мы задаем запрос TrueTime.Now() в момент времени $t_0$, и получаем ответ в момент времени $t_1$, то этот интервал $[t_0, t_1]$ пересекается с интервалом $[e, l]$. Беда в том, что мы не знаем $t_0$ и $t_1$
Нам гарантируют это, и стремятся сделать так, чтобы ширина этого интервала была как можно меньше: $|l - e| \le 6ms$

Как это связано с GPS? TrueTime работает на каждой машине. И в датацентре помимо обычных машин, которые пользуются этим сервисом, стоят т.н. time-master'а. Бывает два типа тайм мастеров:
1) Машины, на которых установлены GPS антенны, которые синхронизируются с GPS для того, чтобы синхронизировать часы
2) Armageddon master - машины с атомными часами. У нас атомные часы в космосе, атомные часы у нас (для отказоустойчивости)
Исходя из всей этой информации, каждая машина выводит себе оценку этого интервала $[e, l]$. И выводит забавно: раз в 30 секунд каждая машина общается с этими мастерами и получает себе оценку интервала. А теперь через 25 секунд приходит Google Spanner и спрашивает у TrueTime текущее время. Что вернет нам TrueTime? TrueTime закладывает между точками синхронизации дрейф в 200 миллионных долей и вернется промежуток больше

Читать как иврит справа налево
![alt text](images/10.png)

## Зачем распределенным системам TrueTime?
Время - это такая хитрость, общее разделяемое состояние между машинами в сети. Чтобы мы могли что-то сделать, надо чтобы машины поговорили друг с другом, чтобы они друг о друге что-то узнали. У них есть что-то общее - время. Они могут говорить явно друг с другом, отправляя сообщения. Google используют TrueTime для того, чтобы не говорить. Они строят не просто базу данных в кластере, они строят геораспределенную систему такую, в которой узлы стоят физически далеко друг от друга. И когда мы общаемся по сети внутри ДЦ, там действительно могут быть тайминги очень маленькие (1ms). Когда мы пересылаем байтики через Атлантику или с Восточного на Западное побережье США, там уже тайминги сотни миллисекунд. И мы с помощью некоторой хитрости можем понять, как можно эту коммуникацию, которая может быть долгой, можно заменить на локальное ожидание, т.е. мы просто сидим и ничего не делаем, и засчет этого что-то становится лучше. Но сидим мало: 6ms, вместо того, чтобы общаться долго и ждать 100ms. Это большая хитрость за счет того, что время у узлов общее.

## Итоги
Можно сказать следующее: (в данном курсе) мы живем чаще всего в асинхронной модели, т.е. мы не делаем предположений о скорости доставки сообщений, мы не делаем предполоожение о скорости работы часов, и мы не делаем предположений о дрейфе часов (в тех случаях, когда мы доказываем, что системы не нарушают каких-то гарантий, т.е. safety. Когда мы доказываем, что система не делает ничего плохого, мы используем асинхронную модель, где нет параметров времени никаких. Когда мы доказываем, что система вообще говоря делает что-то полезно и отвечает пользователю, то мы тогда привлекаем таймауты, какие-то оценки, время доставки сообщений и прочее)

# Линеаризуемость. Репликация регистра, алгоритм ABD
## KV Storage
### История
Мы построили модель, которая будет решать наши задачи. Пока что мы живем в моделе, где узлы отказывают, просто выключаясь навсегда. Т.е. узлы ведут себя в планах протокола, и злоумышленников у нас нет

KV Storage
* Set(key, value)
* Get(key)

В начале 2000-х годов Google и Amazon написали статьи про Google Bigtable и Amazon Dynamo, это были KV системы
### Слои архитектуры распределенной БД
Почему мы говорим о том, что мы все еще ничего не умеем? Если мы посмотрим на современные БД, под капотом они реализованы поверх KV Storage

Мы хотим сделать KV Storage. Что это значит? Оно должно масштабироваться и быть отказоустойчивым. Но задачу мы начинаем решать на уровне одной машины. В чем сложность? В том, что машина должна переживать рестарты, т.е. хранить не в памяти, а на жестком диске. У нас API с произвольным доступом - у нас есть произвольные ключи и мы что-то по ним спрашивам: пишем или читаем. А диск так не умеет - он умеет читать и писать только последовательно
Поэтому первая задача, которая возникает в пределах одной машины - это реализация хранилища. Мы должны на самом низком уровне в пределах одного узла построить эффективно систему с произвольном доступом поверх диска, который умеет последовательный доступ эффективно (это системы типа LevelDB и RoseDB)
Теперь мы хотим добавить отказоустойчивость: т.е. мы начинаем данные реплецировать, чтобы переживать смерть отдельных дисков
Далее у нас данные перестают вмещаться на одной машине, и мы начинаем заниматься распределением этих данных по кластеру: у нас есть теперь много машин и очень большой диапазон ключей, каждый из них мы назовем range - ключи $[b, d]$. Мы скажем, что каждый range будет реплецироваться независимо. Т.е. для каждого диапазона будет несколько машин, которые реплецируют этот диапазон. Скажем, что на этом уровне мы наши данные шардировали
![alt text](image.png)
Наконец, последний уровень - уровень транзакций, потому что без транзакций в KV Storage мы не получим транзакции на уровне SQL
// TODO, остановился на 10.30