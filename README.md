# Polog - ультимативный логгер, пишущий в базу данных

Используйте преимущества базы данных для логирования в ваших проектах! Легко ищите нужные вам логи, составляйте статистику и управляйте записями при помощи SQL.

Данный пакет максимально упростит миграцию ваших логов в базу данных. Вот список некоторых преимуществ логгера Polog:

- **Автоматическое логирование**. Просто повесьте декоратор на вашу функцию или класс, и каждый вызов будет автоматически логироваться в базу данных (или только ошибки - это легко настроить)!
- **Высокая производительность**. Непосредственно сама запись в базу делается из отдельных потоков и не блокирует основной поток исполнения вашей программы.
- **Поддержка асинхронных функций**. Декораторы для автоматического логирования работают как на синхронных, так и на асинхронных функциях.
- **Минималистичный синтаксис без визуального мусора**. Сделать логирование еще проще уже вряд ли возможно. Вы можете залогировать целый класс всего одним декоратором. Имена функций короткие, насколько это позволяет здравый смысл.
- **Удобное профилирование**. В базу автоматически записывается время работы ваших функций. Вы можете накопить статистику производительности вашего кода и легко ее анализировать.
- Поддержка **SQLite**, **PostgreSQL**, **MySQL**, **Oracle** и **CockroachDB** за счет использования "под капотом" [Pony ORM](https://ponyorm.org/).
- Удобная работа с **несколькими сервисами**, которые пишут в одну базу.
- Возможность писать собственные **расширения** или пользоваться встроенными. К примеру, вы можете [настроить отправку](#включаем-оповещения-по-электронной-почте) уведомлений об ошибках по электронной почте.

## Оглавление

- [**Быстрый старт**](#быстрый-старт)
- [**Общая схема работы**](#как-это-все-работает)
- [**Уровни логирования**](#уровни-логирования)
- [**Настройки логгера**](#общие-настройки)
- [**Декоратор ```@flog```**](#декоратор-flog)
- [**```@clog``` - декоратор класса**](#clog---декоратор-класса)
- [**Перекрестное использование ```@сlog``` и ```@flog```**](#перекрестное-использование-сlog-и-flog)
- [**Запрет логирования через декоратор ```@logging_is_forbidden```**](#запрет-логирования-через-декоратор-logging_is_forbidden)
- [**"Ручное" логирование через ```log()```**](#ручное-логирование-через-log)
- [**Написание своих расширений**](#пишем-свои-расширения)
- [**Как включить уведомления по электронной почте**](#включаем-оповещения-по-электронной-почте)
- [**Работа с записями**](#работа-с-записями)
- [**Общие советы про логирование**](#общие-советы-про-логирование)

## Быстрый старт

Установите Polog через [pip](https://pypi.org/project/polog/):

```
$ pip install polog
```

Теперь просто импортируйте декоратор ```@flog``` и примените его к вашей функции. Никаких настроек, ничего лишнего - все уже работает:

```python
from polog.flog import flog


@flog
def sum(a, b):
  return a + b

print(sum(2, 2))
```

На этом примере при первом вызове функции sum() в папке с вашим проектом будет автоматически появится файл с базой данных SQLite, в которой будет соответствующая запись. В данном случае сохранится информация о том, какая функция была вызвана, из какого она модуля, с какими аргументами, сколько времени заняла ее работа и какой результат она вернула.

Теперь попробуем залогировать ошибку:

```python
from polog.flog import flog


@flog
def division(a, b):
  return a / b

print(division(2, 0))
```

Делим число на 0. Что на этот раз записано в базу? Очевидно, что результат работы функции записан не будет, т.к. она не успела ничего вернуть. Зато там появится подробная информация об ошибке: название вызванного исключения, текст его сообщения, трейсбек и даже локальные переменные. Кроме того, появится отметка о неуспешности выполненной операции - они проставляются ко всем автоматическим логам, чтобы вы могли легко выбирать из базы данных только успешные или только неуспешные операции и как-то анализировать результат.

Еще небольшой пример кода:

```python
from polog.flog import flog


@flog
def division(a, b):
  return a / b

@flog
def operation(a, b):
  return division(a, b)

print(operation(2, 0))
```

Чего в нем примечательного? В данном случае ошибка происходит в функции division(), а затем, поднимаясь по стеку вызовов, она проходит через функцию operation(). Однако логгер записал в базу данных сообщение об ошибке только один раз! Встретив исключение в первый раз, он пишет его в базу и подменяет другим, специальным, которое игнорирует в дальнейшем. В результате ваша база данных не засоряется бесконечным дублированием информации об ошибках.

На случай, если ваш код специфически реагирует на конкретные типы исключений и вы не хотите, чтобы логгер исключал дублирование логов таким образом, его поведение можно изменить, об этом вы можете прочитать в более подробной части документации ниже. Однако имейте ввиду, что, возможно, существуют лучшие способы писать код, чем прокидывать исключения через много уровней стека вызовов функций, после чего ловить их там, ожидая конкретный тип.

Что, если мы хотим залогировать целый класс? Обязательно ли проходиться по всем его методам и на каждый вешать декоратор ```@flog```? Нет! Для классов существует декоратор ```@clog```:

```python
from polog.clog import clog


@clog
class OneOperation(object):
  def division(self, a, b):
    return a / b

  def operation(self, a, b):
    return self.division(a, b)

print(OneOperation().operation(2, 0))
```

Что он делает? Он за вас проходится по методам класса и вешает на каждый из них декоратор ```@flog```. Если вы не хотите логировать ВСЕ методы класса, передайте в ```@clog``` имена методов, которые вам нужно залогировать, например: ```@clog('division')```.

Если вам все же не хватило автоматического логирования, вы можете писать логи вручную, вызывая функцию ```log()``` из своего кода:

```python
from polog.log import log


log("All right!")
log("It's bad.", exception=ValueError("Example of an exception."))
```

На этом введение закончено. Если вам интересны тонкости настройки логгера и его более мощные функции, можете почитать более подробную документацию.

## Как это все работает?

Процесс выполнения любой программы состоит из событий: вызываются функции, поднимаются исключения и т. д. Эти события, в свою очередь, состоят из других событий, масштабом поменьше. Задача логгера - записать максимально подробный отчет обо всем этом, чтобы программист в случае сбоя мог быстро обнаружить место ошибки и устранить ее. Записать все события до мельчайших деталей невозможно - данных было бы слишком много. Поэтому обычно человек непосредственно указывает места в программном коде, записи из которых его интересуют. На этом принципе построен [модуль логирования](https://docs.python.org/3/library/logging.html) из стандартной библиотеки.

Обычно вызовы стандартного логгера в коде выглядят как-то так:

```python
import logging


logging.debug('Skip this message!')
logging.info("Sometimes it's interesting.")
logging.warning('This is serious.')
logging.error('PANIC')
logging.critical("I'm quitting.")
```

В разных ситуациях нам нужно получать разное количество информации из программы: от максимальной подробности на этапе разработки, до редких записей об ошибках и иных важных событиях при реальной эксплуатации. Чтобы не лазить по всему коду и не исправлять / удалять каждый вызов логгера, мы манипулируем лишь общим уровнем логгирования. Необходимый уровень важности записи является фильтром, который устанавливается глобально по всей программе.

В чем отличие Polog от описанной схемы? Больших отличий по принципу действия тут нет. Программа все так же производит события, а мы их записываем или нет, в зависимости от выбранного уровня логирования. Задача библиотеки - по возможности, максимально очистить код ваших функций от визуального мусора, создаваемого вызовами логгеров. Дело в том, что большинство событий, записываемых логгерами, вполне стандартны. Программисту обычно интересно знать, что программа "зашла" в какой-то блок кода, или что в каком-то блоке было поднято исключение. Но если программа написана хорошо и ее логические части разбиты на независимые функции, нам нет нужды залезать в них, чтобы записать все эти вещи. Достаточно обернуть их вызов в [декоратор](#декоратор-flog), который сделает все сам.

С появлением декоратора возникают новые вопросы. Если у меня большой класс с кучей методов, мне что, на каждый метод навешивать декоратор? Если навесить его на одну функцию несколько раз, он при каждом вызове будет несколько записей делать? Что, если исключение пройдет через несколько таких задекорированных функций по стеку вызовов - оно много раз запишется? Как быть, если я точно не хочу, чтобы информация из какой-то функции вылезала наружу, например если там важные клиентские данные? Это все работает только на обычных функциях, или на корутинах тоже?

Для решения каждого из этих вопросов у Polog есть свой инструмент, об этом вы можете подробнее прочитать далее.

Помимо интерфейса регистрации событий, Polog немного отличается и по внутреннему устройству. Схема его работы выглядит примерно так:

- События "ловятся" через декораторы или функции ручного логирования.
- У события определяется важность. Если важность выше или равна текущему уровню логирования, мы работаем с событием дальше. Если нет, логгер не делает с ним больше ничего.
- Из события извлекаются данные. Когда оно произошло? В каком месте кода? Было ли исключение? И еще несколько вопросов, ответы на которые записываются в некое промежуточное представление.
- Промежуточное представление кладется в очередь на запись.
- В эту очередь смотрят несколько "воркеров" из отдельных потоков. Как только один из них освобождается, он может взять следующий лог из очереди и записать его. Числом потоков с воркерами вы можете управлять, по умолчанию их 2.
- Получив объект промежуточного представления события, воркер последовательно передает его в каждый из имеющихся у него обработчиков.
- Обработчики делают с событием кто что умеет: записывают в файл, отправляют по почте, пишут в базу данных, отправляют на внешние сервисы и т. д.

Вы как пользователь библиотеки можете использовать готовые обработчики или писать свои. Кроме того, вы можете гибко настраивать критерии, по которым определяется, записывать событие или нет.

Обратите внимание, что когда событие попадает в очередь, ваша программа уже приступает к дальнейшему выполнению. То есть ввод-вывод логгера практически не влияет на ее производительность, в отличие от большиинства прочих решений.

### Уровни логирования

Как было сказано выше, по умолчанию автоматические логи имеют 2 уровня: 1 и 2. 1 - это рядовое событие, 2 - исключение. Однако это легко поменять.

В декораторах вы можете указать желаемый уровень логирования:

```python
from polog.flog import flog


@flog(level=5)
def sum(a, b):
  return a + b

print(sum(2, 2))
# В базу упадет лог с меткой 5 уровня.
```

Это доступно как для ```@flog```, так и для ```@clog```, работает одинаково.

Также вы можете присвоить уровням логирования имена и в дальнейшем использовать их вместо чисел:

```python
from polog.config import config
from polog.flog import flog


# Присваиваем уровню 5 имя 'ERROR', а уровню 1 - 'ALL'.
config.levels(ERROR=5, ALL=1)

# Используем присвоенное имя вместо номера уровня.
@flog(level='ERROR')
def sum(a, b):
  return a + b

print(sum(2, 2))
# В базу упадет лог с меткой 5 уровня.
```

При этом указание уровней числами вам по-прежнему доступно, имена и числа взаимозаменяемы.

Если вы привыкли пользоваться стандартным модулем [logging](https://docs.python.org/3.8/library/logging.html), вы можете присвоить уровням логирования [стандартные имена](https://docs.python.org/3.8/library/logging.html#logging-levels) оттуда:

```python
from polog.config import config
from polog.flog import flog


# Имена уровням логирования проставляются автоматически, в соответствии со стандартной схемой.
config.standart_levels()

@flog(level='ERROR')
def sum(a, b):
  return a + b

print(sum(2, 2))
# В базу упадет лог с меткой 40 уровня.
```

Также вы можете установить текущий уровень логирования:

```python
from polog.config import config
from polog.flog import flog


# Имена уровням логирования проставляются автоматически, в соответствии со стандартной схемой.
config.standart_levels()

# Устанавливаем текущий уровень логирования - 'CRITICAL'.
config.set(level='CRITICAL')

@flog(level='ERROR')
def sum(a, b):
  return a + b

print(sum(2, 2))
# Запись в базу произведена не будет, т. к. уровень сообщения 'ERROR' ниже текущего уровня логирования 'CRITICAL'.
```

Все события уровнем ниже в базу не пишутся. По умолчанию уровень равен 1.

Используя декораторы, для ошибок вы можете установить отдельный уровень логирования:

```python
# Работает одинаково в декораторе функций и декораторе классов.
@flog(level='DEBUG', error_level='ERROR')
@clog(level='DEBUG', error_level='ERROR')
```

Также вы можете установить уровень логирования для ошибок глобально через настройки:

```python
from polog.config import config


config.set(error_level='CRITICAL')
```

Сделав это 1 раз, вы можете больше не указывать уровни логирования локально в каждом декораторе. Но иногда вам это может быть полезным. Уровень, указанный в декораторе, обладает более высоким приоритетом, чем глобальный. Поэтому вы можете, к примеру, для какого-то особо важного класса указать более высокий уровень логирования. Или наоборот, понизить его, если не хотите в данный момент записывать логи из конкретной функции или класса.

### Общие настройки

Выше уже упоминалось, что общие настройки логирования можно делать через класс ```config```. Давайте вспомним, откуда его нужно импортировать:

```python
from polog.config import config
```

Класс ```config``` предоставляет несколько методов. Все они работают непосредственно от класса, без вызова \_\_init\_\_, например вот так:

```python
config.set(pool_size=5)
```

Методы класса ```config```:

- **```set()```**: общие настройки логгера.

  Принимает следующие именованные параметры:

    **pool_size** (int) - количество потоков-воркеров, которые пишут в базу данных. По умолчанию оно равно 2-м. Вы можете увеличить это число, если ваша программа пишет в базу достаточно интенсивно. Но помните, что ~~большое число потоков - это большая ответственность~~ дополнительные потоки повышают накладные расходы интерпретатора и могут замедлить вашу программу.

    **service_name** (str) - имя сервиса. Указывается в каждой записи в базу. По умолчанию 'base'.

    **level** (int, str) - общий уровень логирования. События уровнем ниже записываться в базу не будут. Подробнее в разделе "**Уровни логирования**" (выше).

    **errors_level** (int, str) - уровень логирования для ошибок. По умолчанию он равен 2-м. Также см. в "**Уровни логирования**".

    **original_exceptions** (bool) - режим оригинальных исключений. По умолчанию False. True означает, что все исключения остаются как были и никак не видоизменяются логгером. Это может приводить к дублированию информации об одной ошибке в базе данных, т. к. исключение, поднимаясь по стеку вызовов функций, может пройти через несколько задекорированных логгером функций. В режиме False все исключения логируются 1 раз, после чего оригинальное исключение подменяется на ```polog.errors.LoggedError```, которое не логируется никогда.

    **delay_before_exit** (int, float) - задержка перед завершением программы для записи оставшихся логов. При завершении работы программы, может произойти небольшая пауза, в течение которой будут записаны оставшиеся логи из очереди. Максимальная продолжительность такой паузы указывается в данной настройке.

- **```levels()```**: присвоение имен уровням логирования, см. подробнее в разделе "**Уровни логирования**".

- **```standart_levels()```**: присвоение стандартных имен уровням логирования, см. подробнее в разделе "**Уровни логирования**".

- **```db()```**: указание базы данных, куда будут писаться логи.

  Для управления базой данных Polog использует [Pony ORM](https://ponyorm.org/) - самую быструю и удобную ORM из доступных на python. В метод ```db()``` вы можете передать данные для подключения к базе данных в том же формате, в каком это делается в методе ```bind()``` [самой ORM](https://docs.ponyorm.org/database.html).

  Pony поддерживает: **SQLite**, **PostgreSQL**, **MySQL**, **Oracle**, **CockroachDB**.

  Если вы не укажете никакую базу данных, по умолчанию логи будут писаться в базу SQLite, которая будет автоматически создана в папке с проектом в файле с названием ```logs.db```.

### Декоратор ```@flog```

Декоратор ```@flog``` используется для автоматического логирования вызовов функций. Поддерживает как обычные функции, так и [корутины](https://docs.python.org/3/library/asyncio-task.html).

```@flog``` можно использовать как со скобками, так и без. Вызов без скобок эквивалентен вызову со скобками, но без аргументов.

Напомним, он импортируется так:

```python
from polog.flog import flog
```

Используйте параметр ```message``` для добавления произвольного текста к каждому логу.

```python
@flog(message='This function is very important!!!')
def very_important_function():
  ...
```

Про управление уровнями логирования через аргументы к данному декоратору читайте в разделе "**Уровни логирования**" (выше).

### ```@clog``` - декоратор класса

По традиции, вспомним, откуда он импортируется:

```python
from polog.clog import clog
```

Может принимать все те же аргументы, что и ```@flog()```, либо использоваться без аргументов - как со скобками, так и без. Автоматически навешивает декоратор ```@flog``` на все методы задекорированного класса.

Игнорирует дандер-методы (методы, чьи названия начинаются с "\_\_").

Если не хотите логировать все методы класса, можете перечислить нужные в качестве неименованных аргументов:

```python
@clog('important_method', message='This class is also very important!!!')
class VeryImportantClass:
  def important_method(self):
    ...
  def not_important_method(self):
    ...
  ...
```

### Перекрестное использование ```@сlog``` и ```@flog```

При наложении на одну функцию нескольких декораторов логирования, срабатывает из них по итогу только один. Это достигается за счет наличия внутреннего реестра задекорированных функций. При каждом новом декорировании декорируется оригинальная функция, а не ее уже ранее задекорированная версия.

Пример:

```python
@flog(level=6) # Сработает только этот декоратор.
@flog(level=5) #\
@flog(level=4) # |
@flog(level=3) #  > А эти нет. Они знают, что их несколько на одной функции, и уступают место последнему.
@flog(level=2) # |
@flog(level=1) #/
def some_function(): # При каждом вызове этой функции лог будет записан только 1 раз.
  ...
```

Мы наложили на одну функцию 6 декораторов ```@flog```, однако реально сработает из них только тот, который выше всех. Это удобно в ситуациях, когда вам нужно временно изменить уровень логирования для какой-то функции. Не редактируйте старый декоратор, просто навесьте новый поверх него, и уберите, когда он перестанет быть нужен.

Также вы можете совместно использовать декораторы ```@сlog``` и ```@flog```:

```python
@clog(level=3)
class SomeClass:
  @flog(level=10)
  def some_method(self):
    ...

  def also_some_method(self):
    ...
  ...
```

У ```@flog``` приоритет всегда выше, чем у ```@сlog```, поэтому в примере some_method() окажется задекорирован только через ```@flog```, а остальные методы - через ```@сlog```. Используйте это, когда вам нужно залогировать отдельные методы в классе как-то по-особенному.

### Запрет логирования через декоратор ```@logging_is_forbidden```

На любую функцию или метод вы можете навесить декоратор ```@logging_is_forbidden```, чтобы быть уверенными, что тут не будут срабатывать декораторы ```@сlog``` и ```@flog```. Это удобно, когда вы хотите, к примеру, временно приостановить логирование какой-то функции, не снимая логирующего декоратора.

Импортируется ```@logging_is_forbidden``` так:

```python
from polog.forbid import logging_is_forbidden
```

```@logging_is_forbidden``` сработает при любом расположении среди декораторов логирования:

```python
@flog(level=5) # Этот декоратор не сработает.
@flog(level=4) # И этот.
@flog(level=3) # И этот.
@logging_is_forbidden
@flog(level=2) # И вот этот.
@flog(level=1) # И даже этот.
def some_function():
  ...
```

Также ```@logging_is_forbidden``` удобно использовать совместно с ```@сlog``` для отдельных методов класса.

```python
@сlog
class VeryImportantClass:
  def important_method(self):
    ...

  @logging_is_forbidden
  def not_important_method(self):
    ...
  ...
```

Иногда это может быть удобнее, чем прописывать "разрешенные" методы в самом ```@сlog```. Например, когда в вашем классе много методов и строка с их перечислением получилась бы слишком огромной.

Имейте ввиду, что ```@logging_is_forbidden``` "узнает" функции по их id. Это значит, что, если вы задекорируете конкретную функцию после того, как она помечена в качестве нелогируемой, декораторы Polog будут относиться к ней как к незнакомой:

```python
@flog(level=2) # Этот декоратор сработает, так как не знает, что some_function() запрещено логировать, поскольку функция, вокруг которой он обернут, имеет другой id.
@other_decorator # Какой-то сторонний декоратор. Из-за него изменится первоначальный id функции some_function() и теперь для декораторов Polog это совершенно новая функция.
@logging_is_forbidden
@flog(level=1) # Этот декоратор не сработает, т.к. сообщается с @logging_is_forbidden.
def some_function():
  ...
```

Поэтому декораторы Polog лучше всего располагать поверх всех прочих декораторов, которые вы используете.

### "Ручное" логирование через ```log()```

Отдельные важные события в вашем коде вы можете регистрировать вручную.

Импортируйте функцию ```log()```:

```python
from polog.log import log
```

И используйте ее в вашем коде:

```python
log('Very important message!!!')
```

Уровень логирования указывается так же, как в декораторах ```@flog``` и ```@сlog```:

```python
# Когда псевдонимы для уровней логирования прописаны по стандартной схеме.
log('Very important message!!!', level='ERROR')
# Ну или просто в виде числа.
log('Very important message!!!', level=40)
```

Вы можете передать в ```log()``` функцию, в которой исполняется код:

```python
def foo():
  log(function=foo)
```

Колонки **function** и **module** в этом случае заполнятся автоматически.

Также вы можете передать в ```log()``` экземпляр исключения:

```python
try:
  var = 1 / 0
except ZeroDivisionError as e:
  log('I should probably stop dividing by zero.', exception=e)
```

Колонки **exception_message** и **exception_type** тогда тоже заполнятся автоматически. Флаг ```success``` будет установлен в значение False. Трейсбек и локальные переменные той функции, где произошла ошибка, заполнятся автоматически.

При желании, в качестве аргументов ```function``` и ```exception``` можно использовать и обычные строки, но тогда дополнительные поля не заполнятся сами как надо.

Также вы можете передавать в ```log()``` произвольные переменные, которые считаете нужным залогировать. Для этого нужно использовать функцию ```json_vars()```, которая принимает любые аргументы и переводит их в стандартный json-формат:

```python
from polog.utils.json_vars import json_vars
from polog.log import log


def bar(a, b, c, other=None):
  ...
  log(':D', function=bar, vars=json_vars(a, b, c, other=other))
  ...
```

Также вы можете автоматически получить все переменные в функции при помощи [```locals()```](https://docs.python.org/3/library/functions.html#locals):

```python
def bar(a, b, c, other=None):
  ...
  log(':D', function=bar, vars=json_vars(**locals()))
  ...
```

### Пишем свои расширения

Если вам не хватает функциональности Polog, ее можно расширить. К примеру, вы можете написать собственное расширение, которое будет отправлять логи в вашу любимую NoSQL базу данных, или писать их в файловую систему, или выводить в консоль, или писать вам в мессенджерах / соцсетях.

Расширением будет считаться любая функция, которая принимает в себя набор определенных именованных аргументов. Набор переменных, которые она должна принимать, аналогичен набору полей в базе данных, который вы можете увидеть в разделе [общей информации о логгере](#подробности) (кроме ```id```). Эта функция будет вызываться при каждом событии, которое должно быть залогировано в БД.

Имейте ввиду, что в некоторых случаях Polog может не передавать некоторые переменные в вашу функцию, если извлечение соответствующих данных не подразумевается произошедшим событием. Скажем, переменная ```traceback``` не передается в случае событий, не сопровождаемых исключениями. Если быть точным, следующие аргументы являются обязательными: ```level```, ```success```, ```time```, ```service```, ```auto```. Остальные могут передаваться или нет, по ситуации.

Кроме того, что вы написали расширение, его еще нужно зарегистрировать в движке Polog. Для этого достаточно передать вашу функцию в метод "add_handler":

```python
from polog.config import config
from polog.log import log


# Пример расширения.
def print_function_name(**kwargs):
  if 'function' in kwargs:
    print(kwargs['function'])
  else:
    print('is unknown!')

# Передаем ваше расширение в Polog. В метод add_handler() можно передать несколько функций подряд.
config.add_handler(print_function_name)
# В консоли появится сообщение из вашей функции-расширения.
log('hello!')
```

### Включаем оповещения по электронной почте

В составе Polog есть ряд уже готовых расширений. Одно из них позволяет вам настроить отправку электронных писем по [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol)-протоколу. Вам это может пригодиться для быстрого реагирования на какие-то особо критичные события в вашем коде.

Подключается расширение так:

```python
from polog.config import config
from polog.handlers.smtp.sender import SMTP_sender


# Адреса и пароль абсолютно случайны.
config.add_handlers(SMTP_sender('from_me42@yandex.com', 'JHjhhb87TY(*Ny08z)', 'smtp.yandex.ru', 'to_me@yandex.ru'))
```

```SMTP_sender``` - это вызываемый класс. Обязательных аргументов для его инициализации 4: адрес, с которого мы посылаем письма, пароль от ящика, адрес сервера, к которому мы подключаемся, и адрес, куда мы посылаем письма.

Письма, которые будут сыпаться вам на почту, будут выглядеть примерно так:

```
Message from the Polog:

auto = True
module = __main__
function = do
time = 2020-09-22 20:31:45.712366
exception_message = division by zero
exception_type = ZeroDivisionError
success = False
traceback = [" File \"some_path\", line 46, in wrapper\n result = func(*args, **kwargs)\n"," File \"test.py\", line 23, in do\n return x \/ y\n"]
local_variables = {"x":{"value":1,"type":"int"},"y":{"value":0,"type":"int"}}
time_of_work = 2.86102294921875e-06
level = 2
input_variables = {"args":[{"value":1,"type":"int"},{"value":0,"type":"int"}]}
```

При необходимости, вы можете настроить отправку писем более тонко. Для этого в конструктор класса нужно передать дополнительные параметры. Вот их список:

- ```port``` (int) - номер порта в почтовом сервере, через который происходит отправка почты. По умолчанию 465 (обычно используется для шифрованного соединения).
- ```text_assembler``` (function) - альтернативная функция для генерации текста сообщений. Должна принимать в себя те же аргументы, которые обычно передаются в пользовательские расширения Polog, и возвращать строковый объект.
- ```subject_assembler``` (function) - по аналогии с аргументом "text_assembler", альтернативная функция для генерации темы письма.
- ```only_errors``` (bool) - фильтр на отправку писем. В режиме False (то есть по умолчанию) через него проходят все события. В режиме True - только ошибки, т. е., если это не ошибка, письмо гарантированно отправлено не будет.
- ```filter``` (function) - дополнительный фильтр на отправку сообщений. По умолчанию он отсутствует, т. е. отправляются сообщения обо всех событиях, прошедших через фильтр "only_errors". Вы можете передать сюда свою функцию, которая должна принимать стандартный для расширений Polog набор аргументов, и возвращать bool. Возвращенное значение True из данной функции будет означать, что сообщение нужно отправлять, а False - что нет.
- ```alt``` (function) - функция, которая будет вызвана в случае, если отправка сообщения не удалась или запрещена фильтрами. Набор принимаемых аргументов, опять же, стандартный для расширений.
- ```is_html``` (bool) - флаг, является ли отправляемое содержимое HTML-документом. По умолчанию False. Влияет на заголовок письма.

Имейте ввиду, что отправка письма - процесс довольно затратный, поэтому имеет смысл это делать только в исключительных ситуациях. Кроме того, если у вас не свой SMTP-сервер, а вы пользуетесь какими-то публичными сервисами, у них часто есть свои ограничения на отправку писем, так что злоупотреблять этим тоже не стоит. В некоторых случаях письма могут просто не отправляться из-за политики используемого вами сервиса.

Кроме того, опять же, из-за затратности процесса отправки, некоторые письма могут не успеть отправиться в случае экстренного завершения программы.

### Работа с записями

Как уже упоминалось выше, Polog управляет базой данных с помощью [Pony ORM](https://ponyorm.org/). Pony была выбрана в первую очередь за свой самый лаконичный и самый питонячий язык запросов, а также за самую высокую [скорость работы](https://habr.com/ru/post/496116/).

В составе пакета Polog есть модельный класс лога. Импортировать его вы можете так:

```python
from polog.select import logs
```

Кроме того, если до сих пор с момента запуска ваша программа еще ни разу не делала записей в лог, вам нужно инициализировать Polog:

```python
from polog.connector import Connector


Connector()
```

В Polog используется каскадная инициализация внутренних механизмов, которая по умолчанию запускается вместе с созданием первой записи. В данном случае мы запускаем тот же процесс вручную.

Теперь вы можете делать запросы, используя функцию [```select()```](https://docs.ponyorm.org/queries.html). Для извлечения данных из базы вам необходимо также использовать [```@db_session```](https://docs.ponyorm.org/transactions.html) (работает как декоратор и как менеджер контекста). Пример запроса с помощью Pony:

```python
from datetime import datetime
from pony.orm import select, db_session
from polog.select import logs
from polog.connector import Connector


# Инициализация Polog.
Connector()
# @db_session еще можно использовать в качестве декоратора для функции, читайте подробнее в документации Pony.
with db_session:
  # Получаем списко-подобный объект, содержащий все записи с пометкой о неуспешности операций. Вы можете прописать здесь сколь угодно сложный набор условий, опираясь на чисто питоновский синтакисис выражений-генераторов.
  # Если вы не знакомы с выражениями-генераторами как с концепцией, прочитайте вот это: https://www.python.org/dev/peps/pep-0289/
  all_unsuccessful = select(x for x in logs if x.success == False)
  # От полученного списко-подобного объекта также можно делать запросы через select(). В данном случае мы фильтруем логи, выбирая только те их них, которые были записаны позднее 2 июня 2019 года.
  all_new_unsuccessful = select(x for x in all_unsuccessful if x.time > datetime(2019, 6, 2))
  for one in all_new_unsuccessful:
    # К атрибутам лога вы можете обращаться, используя те же имена колонок в таблице базы данных.
    print(one.message)
```

Вам будет полезно знать, что Pony делает ленивые запросы к БД, обращаясь туда только когда вы действительно используете данные. То есть каждый из каскадной последовательности селектов, пока вы никак не задействовали оттуда данные, не представляет собой "реальный" запрос к базе. Кроме того, выражение-генератор, которое вы скармливаете функции ```select()```, в реальности не перебирает элементы. Pony хитро парсит [абстрактное синтаксическое дерево](https://docs.python.org/3/library/ast.html) генератора и преобразует его в соответствующий запрос SQL. И еще кое-что. Советуем не использовать для извлеченных из базы последовательностей встроенную функцию len(), предпочитайте ей [```count()```](https://docs.ponyorm.org/aggregations.html#function-count) из пакета Pony, т.к. первая повлечет за собой полную выборку последовательности из базы, а вторая соответствует одноименной SQL-функции.

Для удобства вы можете импортировать все те же функции из пакета Polog:

```python
from polog.select import logs, select, db_session
from polog.connector import Connector


Connector()

with db_session:
  all = select(x for x in logs)
```

Функция ```select()``` в составе Polog слегка модифицирована относительно оригинала из Pony, т. к. по умолчанию сортирует результат запроса по полю **time**. ```@db_session``` оригинальная.

## Общие советы про логирование

Чтобы получить наибольшую пользу от ведения логов, следуйте нескольким небольшим правилам для организации вашего проекта.

- Заведите для логов отдельную базу данных. Она может быть одна для нескольких разных проектов или сервисов, однако желательно отделить ее от той базы, с которой работает ваше приложение. Логирование иногда может быть достаточно интенсивным, и так вы защитите функциональность вашего приложения от этой нагрузки.
- Держите каждый класс в отдельном файле. Не держите "отдельно стоящих" функций в одном файле с классом. Помимо очевидного, что это делает вашу работу с проектом удобнее, это также устраняет возможность конфликта имен. Polog записывает название функции и модуля. Но если в модуле присутствуют 2 функции с одинаковыми названиями (например, в составе разных классов), вы не сможете их отличить, когда будете читать логи, и можете принять за одну функцию, которая почему-то ведет себя по-разному.
- Следите за конфиденциальностью данных, которые вы логируете. Скажем, если функция принимает в качестве аргумента пароль пользователя, ее не стоит логировать. Polog предоставляет удобные возможности для экранирования функций от логирования, например декоратор ```@logging_is_forbidden```.
- Избегайте логирования функций, которые вызываются слишком часто. Обычно это функции с низким уровнем абстракции, лежащие в основе вашего проекта. Выберите уровень абстракции, на котором количество логов становится достаточно комфортным. Помните, что, поскольку запись логов в базу делается в отдельном потоке, то, что вы не чувствуете тормозов от записи логов, не означает, что логирование не ведется слишком интенсивно. Вы можете не замечать, пока Polog пишет по несколько гигабайт логов в минуту.

  Для удобства вы можете разделить граф вызова функций на слои, в зависимости их отдаленности от точки входа при запуске приложения. Каждому из уровней присвоить название, а каждому названию указать уровень логирования, который будет тем меньше, чем дальше соответствующий ему уровень от точки входа. Пока вы тестируете свое приложение, общий уровень логирования можно сделать равным уровню самого дальнего слоя, после чего его можно повысить, оставив логируемыми только 2-3 слоя вокруг точки входа.

  Как пример, если вы пишете веб-приложение, у вас наверняка там будут какие-то классы или функции-обработчики для отдельных URL. Из них наверняка будут вызываться некие функции с бизнес-логикой, а оттуда - функции для работы с базой данных. Запускаете вы приложение в условной функции main(). В данном случае функции main() можно присвоить уровень 4, обработчикам запросов - 3, слою бизнес-логики - 2, ну и слою работы с БД - 1.
- Избегайте [излишнего экранирования ошибок](https://en.wikipedia.org/wiki/Error_hiding). Когда вы ловите исключения блоками try-except, в логирующие декораторы они могут не попасть. Поэтому полезно взять за правило каждое использование инструкции except сопровождать [ручным логированием](#ручное-логирование-через-log) образовавшегося исключения.
