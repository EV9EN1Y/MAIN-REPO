## ⭕️ Oсновы по курсу https://code-basics.com/ru/languages/javascript/lessons/hello-world

# + библия https://developer.mozilla.org/ru/

-------


#### чтобы сразу "прикурить" после swift и python:
#### 
```JavaScript

"5" - 1 // 4 (строка стала числом)
"5" + 1 // "51" (число стало строкой)


Python - дормальная динамическая типизация
JavaScript - динамическая + принудительное приведение типов (ад для безопасности)
SWIFT - строгая типизация (как определил свойство - таким типом и будет)



то есть можно складывать/сравнивать строки с числами ёпта...
но и ваще почти все типы между собой можно изменять складывать/вычислять итд
это дичь!!!!!
это цирк!!!!
мой родной Swift в этом плане — космический корабль по сравнению с обоссанной табуреткой JS!
теперь ясно - почему нужны спецы по иб в этом цирке js

-----------
Нельзя только =

- Вызвать не функцию (`123()` — ошибка)
    
- Доступ к свойству у `null`/`undefined`
  -------------

'5' * 3           // 15
true + false      // 1
[] + {}           // "[object Object]"
null + 1          // 1
undefined + 1     // NaN
'XSS' - 1         // NaN
'5' + [1,2]       // "51,2"


true + true      // 2  (true стал 1)
true - false     // 1  (true=1, false=0)
false * 10       // 0  (false=0)
"5" * "3"        // 15 (обе строки стали числами)
"5" * "три"      // NaN (Not a Number, но typeof NaN === "number" - это вообще отдельный цирк)

нечисловые обьекты
[] + []          // "" (пустая строка)
[] + {}          // "[object Object]" (пустой массив стал пустой строкой, объект стал строкой)
{} + []          // 0 (в некоторых контекстах это интерпретируется как блок кода + пустой массив)
[] + [] + "0"    // "0" (работает как конкатенация строк)


[] + []  // "" (пустая строка)
[] + {}  // "[object Object]"
{} + []  // 0 (WAT?)
5 + "5"  // "55"
"5" - 1  // 4
true + true  // 2

операторы сравнения
0 == false       // true (потому что false преобразуется в 0)
"" == false      // true (пустая строка тоже 0)
null == undefined // true (специальное правило)
null === undefined // false (строгое сравнение — разные типы)

Самый питдец:
"0" == false     // true
"0" == 0         // true
0 == ""          // true
"0" == ""        // false (уже нет! потому что оба стали числами: 0 == 0, но строка "0" становится числом 0, а пустая строка становится 0 — а, бля, стоп, тут надо разбираться)
// Лучше просто запомнить: никогда не используй ==, только ===

инкремент
let a = "5";
a++              // 5 (пост-инкремент вернул 5, но a теперь 6)
typeof a         // "number" (строка "5" стала числом)
let b = "5";
b = b + 1        // "51" (конкатенация)
typeof b         // "string"

---------------

JavaScript
	 =     это присваивание (а = 10)
	 ==    это нестрогое равенство с приведением типов немного 
	 ===   это строгое равенство без приведения типов
	 
	 
	5 == "5"          // true (строку "5" привел к числу 5)
	0 == false        // true (false привел к 0)
	null == null      // true
	null === null     // true 
	
	// true легаси треш, null-нет значения - установил програмист, undefined -это значение не определено , установленно системой.. (типо null, когда значение есть и оно сейчас null, null это типо значения еще даже не было воще в памяти, типо ссылки нет на адресс в памяти ? )
	
	null == undefined   // true
	null === undefined  // false (разные типы)
	typeof null         // "object" (отдельный баг, признанный, но оставленный)
	typeof undefined    // "undefined"
	
	
	[] == false       // true (пустой массив стал пустой строкой "", которая стала 0, который равен false)
	"  " == 0         // true (пробелы стали 0, потому что строка пустая после trim?)
	
	5 === "5"         // false (number vs string)
	0 === false       // false (number vs boolean)
	[] === false      // false

```

чтобы понимать этот трындец нужно учить таблицы приведения:
--------------------------------------------

# https://george75.gitlab.io/articles/js-type-conversions-cheat-sheet/?ysclid=mlsejhtct297984890


### **Строковый контекст** (`+` если один операнд строка)

Всё в строку: `5 + '5' = '55'`

### **Числовой контекст** (`-`, `*`, `/`, `%`, `>`)

Всё в число: `'5' - 3 = 2`, `true * 5 = 5`

### **Логический контекст** (`if`, `&&`, `||`)

**Falsy (становятся false):**

- `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`
    

**Всё остальное — truthy:**

- `"0"`, `"false"`, `[]`, `{}`, `function(){}`
    

### **Сравнение `==`** (никогда не используй!)

Сложные правила приведения. Просто **всегда ставь `===`**.

### **С объектами**

JS спрашивает у объекта:

- Есть `[Symbol.toPrimitive]`? Используй.
    
- Нет? Есть `valueOf`? Используй.
    
- Нет? Есть `toString`? Используй.

--------------

------
- Интерпретируемые (обычно)
    
- Динамическая типизация (переменные меняют тип на лету)
    
- C-подобный синтаксис (скобки `{}`, точки с запятой)
    
- Высокоуровневый
-------

```js
console.log('Hello!');

console.log("Hello!");

и так "
и так ' 
прокатывает почему=то..
```


**Потому что JavaScript позволяет использовать и одинарные (`'`), и двойные (`"`) кавычки для строк. Это сделано для удобства разработчиков.**

## Подробное объяснение:

### 1. Гибкость для разработчика

```js
// Оба варианта работают одинаково
let message1 = 'Привет, мир!';
let message2 = "Привет, мир!";
// Удобно, когда нужно использовать кавычки внутри строки
let quote1 = 'Он сказал: "JavaScript крут!"';    // Внешние одинарные, внутренние двойные
let quote2 = "Он сказал: \"JavaScript крут!\"";  // Экранирование (менее удобно)
let quote3 = "Он сказал: 'JavaScript крут!'";    // Внешние двойные, внутренние одинарные
```

### 2. HTML-дружелюбность


```js
// В HTML часто используются двойные кавычки для атрибутов
let html1 = '<div class="container">';   // Одинарные снаружи — удобно
let html2 = "<div class=\"container\">"; // Меньше удобно, нужно экранирование
```

### 3. JSON-совместимость

```js
// JSON требует двойных кавычек
let json = '{"name": "John"}';  // Одинарные снаружи, двойные внутри — идеально
// Если бы JS поддерживал только одинарные, это было бы неудобно:
let json2 = "{\"name\": \"John\"}"; // Много экранирования
```

### 4. Историческая причина

JavaScript унаследовал этот синтаксис от других языков (C, Java, Python), где тоже есть оба типа кавычек.

### 5. Шаблонные строки (новый уровень)

```js
// В современном JS есть ещё и обратные кавычки ` (template literals)
let name = 'Вася';
let message = `Привет, ${name}!`;  // Можно вставлять переменные
let multiLine = `
  Это
  многострочная
  строка
`;  // И переносы строк сохраняются
```

-----

# комменты
одностр и многостр
```

// For Winterfell!

/* The night is dark and full of terrors. */

```

# инструкции
Код на JavaScript — это набор инструкций, которые, как правило, отделяются друг от друга символом `;`

```js
console.log('Mother of Dragons.'); console.log('Dracarys!');
```

---
# Арифметические операции
```js

3 + 4;
console.log(3 + 4);

- `*`: умножение
- `/`: деление
- `-`: вычитание
- `%`: остаток от деления
- `**`: возведение в степень

console.log(8 / 2); 
```

# Операторы
Операции, которые требуют наличия двух операндов, называются ==бинарными==
унарные- когда  (с одним операндом)
тернарные  (с тремя операндами)
```js

console.log(8 + 2);  `+` — это **оператор**, а числа `8` и `2` — это **операнды**.

console.log(-3); // => -3  унарная операции к числу `3` это одновременно и число само по себе, и оператор с операндом

console.log(6 - (-81));
```

# Коммутативная операция  **коммутативным закон
Бинарная операция считается коммутативной,
==если поменяв местами операнды, вы получаете тот же самый результат. ==
пример коммутативная операция: 3 + 2 = 2 + 3


```js
console.log(3**5);

console.log(-8 / -4);
```

# Композиция операций
```js
console.log(3 * 5 - 2); // => 13
console.log(2 * 4 * 5 * 10);
console.log(8 / 2 + 5 - -3 /2)
```

# Приоритет операций
тут фигня про 2 + 2 * 2  =6
```javaScript
console.log(70 * (3 + 4) / (8 + 2));
```


# Числа с плавающей точкой
JavaScript не делает различий между рациональными (0.5) и натуральными числами (10), для него и то, и другое - числа
```javaScript
речь про 0.2 * 0.2 // 0.04000000000000001..


// Максимальное возможное целое число console.log(Number.MAX_SAFE_INTEGER); 9007199254740991
console.log(Number.MAX_SAFE_INTEGER); == 9007199254740991



```


# Бесконечность (Infinity)
```javaScript
1 / 0  В низкоуровневых языках она приводит к краху 
Бесконечность в JavaScript —  настоящее число

Infinity + 4; // Infinity 
Infinity - 4; // Infinity 
Infinity * Infinity; // Infinity

console.log((Infinity + Infinity) / 10) = бред но так можно , че поделать
```

# NaN 
`NaN` - специальное значение "не число", которое обычно говорит о том, что была выполнена бессмысленная операция. Результатом практически любой операции, в которой участвует `NaN`, будет `NaN`.
```javaScript
Infinity / Infinity; // NaN
```

# Линтер ()про то как оформлять код
В JavaScript это [eslint](https://eslint.org/).
```tex
- [space-infix-ops] – Отсутствие пробелов между оператором и операндами.
  
- [no-mixed-operators] – По стандарту нельзя писать код, в котором разные операции используются в одном выражении без явного разделения скобками.
  
  
`*` и `/`   не должны быть в одном выражении без разделения скобками (но если скобок дох то лучше разделить это все на части)


console.log((5 ** 2) - (3 * 7)); так - норм 


-----------------

В JavaScript в именах констант и переменных каждое слово пишется с заглавной буквы, кроме первого


- CamelCase — каждое слово в переменной пишется с заглавной буквы. Например: MySuperVar
- lowerCamelCase — каждое слово в переменной пишется с заглавной буквы, кроме первого. Например: mySuperVar (для let переменных)
```

# Кавычки и экранирование
```javaScript
строки 

'Hello'
 'Goodbye' 
 'G' 
 ' ' 
 ''
 
 вот это '$ тоже самое что и " или '
 
 console.log('Dragon's mother');   будет ошибка 
 console.log("Dragon's mother");  но можно экранировать двойными кавычкамии 
 
 --------------
 
 Для экранирования используется обратный слеш \
 
 // Экранируется только ", так как в этой ситуации 
 // двойные кавычки имеют специальное значение 
 console.log("Dragon's mother said \"No\""); // тут \" внутри то что экранировалось \"
 // => Dragon's mother said "No"
 
 
 --------
 
 // обр слеш \ не выводится, если после него идет обычный
  // а не специальный символ 
  console.log("Death is \so terribly final");  // тупо сам слеш в выводе не отобразится!!!!!
  // => Death is so terribly final
  
  0
  ----
  чтобы вывести сам слеш то его нужно заэкранировать самим собой!
  console.log("\\"); вывод = \
  
  -------
  console.log("\\ \\ \\\\ \\\ \'\"");     вывод \ \ \\ \ '"
  
  пример
  console.log('"Khal Drogo\'s favorite word is \"athjahakar\""') выведет  "Khal Drogo's favorite word is "athjahakar""
 
```

# Экранирующие последовательности
```javaScript
\n  перенос строки (перевод)

получить число символов строки
'a'.length;   равно 1

-----

console.log('Joffrey loves using \\n');  а так  ответ будет Joffrey loves using \n в одну строку

----

В Windows для перевода строк 
 \r\n


пример
console.log("- Did Joffrey agree?");

console.log("- He did. He also said \"I love using \\n\".");

вывод
- Did Joffrey agree? - He did. He also said "I love using \n".

```

# Конкатенация
```javaScript
console.log('Dragon' + 'stone');
console.log("King's" + 'Landing'); разные кавы подходят
console.log("King's " + 'Landing');    // King's Landing

console.log('Winter' + ' ' + 'came ' + 'for' + ' ' + 'the' + ' House of Frey.')
```

# Кодировка  двоичный код  - бинарщина
(хз - но ладно..) просто в курсе есть
```javaScript

- 0 ← 0
- 1 ← 1
- 2 ← 10
- 3 ← 11
- 4 ← 100
- 5 ← 101
  
  - a ← 1
- b ← 2
- c ← 3
- d ← 4
- ...
- z ← 26
  
  - hello → 8 5 12 12 15
  - 7 15 15 4 → good
    
    
// вывод символа из  кодировки ASCII   
   console.log(String.fromCharCode(63)); 63 это ?

```

## Таблица соответствия (ASCII)

| Символ    | Десятичный | Двоичный | Шестнадцатеричный |
| --------- | ---------- | -------- | ----------------- |
| 0         | 48         | 110000   | 30                |
| 1         | 49         | 110001   | 31                |
| 2         | 50         | 110010   | 32                |
| 3         | 51         | 110011   | 33                |
| 4         | 52         | 110100   | 34                |
| 5         | 53         | 110101   | 35                |
| A         | 65         | 1000001  | 41                |
| B         | 66         | 1000010  | 42                |
| C         | 67         | 1000011  | 43                |
| a         | 97         | 1100001  | 61                |
| b         | 98         | 1100010  | 62                |
| c         | 99         | 1100011  | 63                |
| Пробел    | 32         | 100000   | 20                |
| . (точка) | 46         | 101110   | 2E                |
| / (слэш)  | 47         | 101111   | 2F                |


# переменные это   ==let==     ==const== - константы
```javaScript
let - это переменные
const - константы 

let greeting = 'Father!';    стринг
let msg = `Hello, ${name}`;  стринг обратные выражения


let age = 25;            Number     
let price = 9.99;        Number
let infinity = 1/0;      Number
let nan = "abc" / 2;     Nan  Not a Number — это число монстр, которое не равно самому себе
// Используется для обхода фильтров

let isAdmin = false;     // бул

let empty = null;   Null означает "ничего" или "пусто"

исторический баг =  оператор typeof null возвращает "object", хотя null — это не объект

let bigNumber = 9007199254740991n;     Тип BigInt  (n в конце ) для чисел которые больше 2 в 53-й степени минус 1

 let id = Symbol('id');  Тип Symbol
 
 -----
 
 let user = { name: "John", age: 30 }; (объект),       тип Object
 let arr = [1, 2, 3]; (массив),                         тип Object
 let func = function() {};                               тип Object
 
 
 const house = 'Stark'; 
```

# обратный порядок в строке
```javaScript
let name = 'Brienna';
name = name.split('').reverse().join('');
```

# интерполяция не работает с "  " и ' ' только с   `${    }`   (` бектик обязаловка)
[Интерполяция работает только со строками в бэктиках. Это символ `. 
```javaScript
const firstName = 'Joffrey'; 
const greeting = 'Hello'; 

console.log(`${greeting}, ${firstName}!`);      вывод   Hello, Joffrey!  // ТОЛЬКО БЕКТИКИ!!!!  ` внутри уже ${ } `

```

# Извлечение символов из строки
```javaScript
const firstName = 'Tirion'; console.log(firstName[0]); // => T   счет с 0
КОРОЧЕ ТАКЖЕ КАК В СВИФТЕ ПОЛУЧАТЬ ЭЛЕМЕНТЫ МАССИВА ЧЕРЕЗ []

console.log(firstName[10]); // => undefined т к за пределами
```


# определить тип данных (+ операции с типами данных)   - typeof -
```javaScript
typeof 3; // number typeof 'Game'; // string
```


## ОПЕРАТОРЫ (без воды)

```js
// Арифметика
+ - * / % ++ -- ** // ** это степень, бля, 2**3 = 8
// Сравнение
== // не используй, сравнивает с приведением типов (типа 2 == "2" true) дичь
=== // вот это юзай всегда, строгое сравнение
!= // тоже хня с приведением
!== // норм
> < >= <=
// Логические
&& // и
|| // или
! // не
?? // нул coalescing: a ?? b — если а не null/undefined, то а, иначе b
// Условный оператор
let result = (age >= 18) ? "взрослый" : "мелкий";
```

## УСЛОВИЯ И ЦИКЛЫ

```js
// if/else (тут всё понятно)

if (x > 10) {
		    console.log("че-нить");
} else if (x === 5) {
    console.log("еще че-нить");
} else {
    console.log("че-нить");
}

// switch (кейсы с break, иначе провалится)
switch (day) {
    case 1:
        console.log("понедельник");
        break;
    case 2:
        console.log("вторник");
        break;
    default:
        console.log("дефолтное че-то");
}

//  for циклы
for (let i = 0; i < 10; i++) { console.log(i); } // for ([начало]; [условие]; [шаг]) { действие/тело }
// Инициализация let i = 0 — выполняется один раз в самом начале 


for (let item of arr) { console.log(item); } // для массивов

for (let key in obj) { console.log(key, obj[key]); } // для объектов

// while 

while (условие) { /* тело */ }

// do 

do { /* хотя бы раз выполнится */ } while (условие);

```

## ФУНКЦИИ (ТУТ ЕБUЧИЙ ЗООПАРК)

```js
// Function declaration (можно вызывать до объявления - то есть вызываем вверху кода / а функция находится где - то внизу кода)
function sum(a, b) {
    return a + b;
}

// Function expression (нельзя вызвать до)
const sum2 = function(a, b) {
    return a + b;
};

`- **Function Declaration** (`function sum() {}`) — полностью поднимается. Можно вызывать где угодно.
    
- **Function Expression** (`const sum = function() {}`) — поднимается только объявление переменной (`const sum`), но значение (`function`) присваивается только когда доходит строчка.`

// Стрелочные функции (современный движ)
const sum3 = (a, b) => a + b; // если одна строка, return неявный

const double = x => x * 2; // если один параметр, скобки можно опустить

const getObj = () => ({ name: "братан" }); // чтобы вернуть объект, нужны скобки

// Параметры по умолчанию

function greet(name = "анонимус") {
    console.log(`привет, ${name}`);
}

// Rest параметры (собирает оставшиеся аргументы в массив)

function logAll(...args) {
    args.forEach(arg => console.log(arg));
}
```
##     контейнерs      ОБЪЕКТЫ

```js
// Создание
/Объект {} — это просто контейнер с подписями (как словарь в Swift)

const user = {    // user это такой словарь в который можно положить все че захочу! ключ-значение
    name: "Петрович",
    age: 30,
    "likes vodka": true, // ключ с пробелом
    sayHello() { // метод
        console.log(`Привет, я ${this.name}`);
    }
}; /это контейнер в том числе содержит функ котор можн вызвать так user.sayHello();

// Доступ
user.name // Петрович
user["likes vodka"] // true (через точку не получится)

// Добавление/изменение
user.job = "хакер";
delete user.age;

// Деструктуризация
/ Деструктуризация **НЕ МЕНЯЕТ** оригинальный объект!
const { name, age = 18, job = "безработный" } = user; / name подставился из оригинал user
const { name: userName, ...rest } = user; // rest соберет остальные поля

// Spread (копирование)
const userCopy = { ...user, age: 25 }; // копия с изменением возраста
const merged = { ...obj1, ...obj2 }; // слияние
```
## МАССИВЫ 

```js
const arr = [1, 2, 3, 4, 5];

// Добавление/удаление
arr.push(6); // добавить в конец
arr.pop(); // удалить с конца
arr.unshift(0); // добавить в начало
arr.shift(); // удалить с начала
arr.splice(2, 1, "хуй"); // удалить 1 элемент с индекса 2 и вставить "хуй"

// Поиск
arr.indexOf(3); // индекс элемента или -1
arr.includes(5); // true/false
arr.find(x => x > 3); // первый элемент который > 3
arr.findIndex(x => x > 3); // его индекс - элемент который > 3
arr.filter(x => x > 2); // все элементы > 2

// Трансформация
arr.map(x => x * 2); // [2,4,6,8,10] — каждый элемент преобразуется
arr.reduce((acc, x) => acc + x, 0); // сумма (15)
arr.sort((a, b) => a - b); // сортировка чисел
arr.reverse(); // развернуть
arr.slice(1, 3); // подмассив с 1 по 3 (не включая 3)

// Проверки
arr.every(x => x > 0); // true (все положительные)
arr.some(x => x > 4); // true (есть хотя бы один > 4)

// forEach (просто прогнать)
arr.forEach(x => console.log(x));
```

## КОЛЛЕКЦИИ     Map     и       Set
```js

/ Map - словарь с любыми ключами (не только строками) любые типы
const map = new Map();
map.set('ключ', 'значение');
map.set(obj, { data: 'чё угодно' }); // даже объект как ключ

-------------------------------------------------------------------
пример -------- Map с данными (массив массивов [ключ, значение])
const userMap = new Map([
    ['name', 'Петрович'],
    ['age', 30],
    [true, 'булево значение как ключ'],
    [{id: 1}, 'объект как ключ'],
    [function() {}, 'функция как ключ'] // да, и такое можно!
]);
--------------------------------------------------------------------
работа с мапом-------------
  
// 1. get() - получить по ключу
console.log(map.get('name'));        // "Петрович"
console.log(map.get('age'));         // 30
console.log(map.get(42));            // "ответ на главный вопрос"
// Если ключа нет - вернет undefined
console.log(map.get('job'));         // undefined

// 2. has() - проверить наличие ключа
console.log(map.has('name'));        // true
console.log(map.has('job'));         // false

// 3. Важно! Объекты как ключи работают по ссылке
const objKey = {id: 1};
map.set(objKey, 'нашел по ссылке');
console.log(map.get(objKey));        // "нашел по ссылке"
// А вот так не сработает, хотя объекты одинаковые:
console.log(map.get({id: 1}));       // undefined (это другой объект!)

// 1. delete() - удалить один элемент по ключу
map.delete('city');
console.log(map.has('city'));        // false

// 2. clear() - удалить ВСЁ!
map.clear();
console.log(map.size);               // 0

// 3. Можно удалять и проверять
if (map.has('job')) {
    map.delete('job');
    console.log('job удален');
}

  
// 1. Поиск по ключу (самый быстрый)
if (map.has('name')) {
    console.log(map.get('name'));     // "Петрович"
}

// 2. Поиск по значению (перебор)
// Ищем, где значение 'программист'
for (let [key, value] of map) {
    if (value === 'программист') {
        console.log(`Ключ со значением "программист": ${key}`); // job
    }
}

// 3. Через forEach
map.forEach((value, key) => {
    if (value === 'Москва') {
        console.log(`Нашли Москву по ключу: ${key}`); // city
    }
});

// 4. Найти все ключи с определенным условием
const keysWithLongValues = [];
for (let [key, value] of map) {
    if (typeof value === 'string' && value.length > 6) {
        keysWithLongValues.push(key);  //push добавить в конец массива
    }
}
console.log(keysWithLongValues);      // ['name', 'job']


  
// size - размер (не метод, а свойство!)
console.log(map.size);                // 3
// keys() - все ключи
console.log([...map.keys()]);         // ['name', 'age', 'city']
// values() - все значения
console.log([...map.values()]);       // ['Петрович', 30, 'Москва']
// entries() - все пары [ключ, значение]
console.log([...map.entries()]);      // [['name','Петрович'], ['age',30], ['city','Москва']]
// forEach - перебор
map.forEach((value, key) => {
    console.log(`${key}: ${value}`);
});
// Итерация через for...of
for (let [key, value] of map) {
    console.log(key, value);
}



------------SET ---------------

/ Set - массив УНИКАЛЬНЫХ значений где Можно мешать разные типы

const set = new Set();   Пустой Set

const set = new Set([1, 2, 3, 3, 3]); // [1, 2, 3]
set.has(2); // true
set.add(4);

/ Set с данными (дубликаты автоматически удаляются)
const numbers = new Set([1, 2, 3, 3, 3, 4, 4, 5]);


const mixed = new Set([1, "два", true, {name: "объект"}, [1, 2]]); / Можно мешать разные типы

// 1. add() - добавить элемент
set.add(1);
set.add(2);
set.add(3);
set.add(3); // игнорится, потому что уже есть
console.log(set); // Set(3) {1, 2, 3}

// Можно чейнить (цепочкой)
set.add(4).add(5).add(6);
console.log(set); // Set(6) {1, 2, 3, 4, 5, 6}

// 2. has() - проверить наличие
console.log(set.has(2)); // true
console.log(set.has(10)); // false

// 3. delete() - удалить элемент
set.delete(3);
console.log(set.has(3)); // false
console.log(set); // Set(5) {1, 2, 4, 5, 6}

// 4. clear() - удалить всё
set.clear();
console.log(set.size); // 0

// 5. size - размер (не метод, а свойство!)
const fruits = new Set(['яблоко', 'банан', 'апельсин']);
console.log(fruits.size); // 3


const set = new Set(['яблоко', 'банан', 'апельсин']);

// 1. for...of
for (let item of set) {
    console.log(item); // яблоко, банан, апельсин
}

// 2. forEach
set.forEach(item => {
    console.log(item); // яблоко, банан, апельсин
});

// 3. Превращаем в массив
const arr = [...set];
console.log(arr); // ['яблоко', 'банан', 'апельсин']

// 4. keys(), values(), entries() - есть, но особо не нужны
console.log(set.keys());    // итератор ключей (в Set ключ = значение)
console.log(set.values());  // итератор значений
console.log(set.entries()); // итератор [значение, значение] (для совместимости с Map)

/операции

const setA = new Set([1, 2, 3, 4, 5]);
const setB = new Set([4, 5, 6, 7, 8]);

// Пересечение (что есть и там, и там)
const intersection = new Set([...setA].filter(x => setB.has(x)));
console.log(intersection); // Set(2) {4, 5}

// Разность (что есть в A, но нет в B)
const difference = new Set([...setA].filter(x => !setB.has(x)));
console.log(difference); // Set(3) {1, 2, 3}

// Объединение
const union = new Set([...setA, ...setB]);
console.log(union); // Set(8) {1, 2, 3, 4, 5, 6, 7, 8}

-----------------------------------------------------------
ПРИМЕР СЕТ
-----------------------------------------------------------

// 1. Убрать дубли из строки
const str = "hello world";
const uniqueChars = [...new Set(str)].join('');
console.log(uniqueChars); // "helo wrd"
// 2. Проверить, все ли элементы уникальны в массиве
const allUnique = (arr) => arr.length === new Set(arr).size;
// 3. Случайный выбор без повторов
function randomPick(arr, count) {
    const shuffled = [...arr].sort(() => Math.random() - 0.5);
    return new Set(shuffled.slice(0, count));
}
// 4. Найти общие элементы в массивах
const common = (arr1, arr2) => {
    const set1 = new Set(arr1);
    return arr2.filter(item => set1.has(item));
};
// 5. Отслеживание уникальных посетителей
const visitors = new Set();
visitors.add('IP 192.168.1.1');
visitors.add('IP 192.168.1.2');
visitors.add('IP 192.168.1.1'); // не добавится повторно
console.log(`Уникальных посетителей: ${visitors.size}`);

```

## ПРОМИСЫ И АСИНХРОННОСТЬ

```js

/Промис* (в JavaScript) или [( Promise (или типо async - await) в Swift  ]) - это специальный объект-контейнер, который представляет результат асинхронной операции, которой сейчас нет, но она появится в будущем

// Создание промиса
const promise = new Promise((resolve, reject) => { /создал примис который принимает два аргумента resolve, reject 
    setTimeout(() => {                / setTimeout встроенная функция 
        const random = Math.random(); / создал константу в скопе setTimeout
        if (random > 0.5) {      / обычный if проверка который проверит значение random и потом либо вернет resolve или reject
            resolve("успех, блэт!!"); /вызов функции resolve
        } else {
            reject("навалил кирпича");  /вызов функции reject
        }
    }, 1000); / временная задержка 1000 миллисекунд = 1 секунда
});

/setTimeout - это встроенная функция (как print() в Swift), которая позволяет выполнить код НЕ СРАЗУ, а через указанное время, через 1000 миллисек например тут

ИСПОЛЬЗОВАТЬ ПРОМИС СОЗДАННЫЙ ВЫШЕ

promise
    .then((result) => {  // .then сработает, если вызвали resolve
        console.log("Ура!", result); // "Ура! успех, бля"
    })
    .catch((error) => {  // .catch сработает, если вызвали reject
        console.log("Блин!", error); // "Блин! обосрались"
    });
    
// Или через async/await (современный способ): (как в свифте епта)
async function testPromise() {
    try {
        const result = await promise;
        console.log("Ура!", result);
    } catch (error) {
        console.log("Блин!", error);
    }
}

-------------

// Потребление
/ ПОДПИСКА НА СОБЫТИЯ ПРОМИСА
promise
    .then(result => console.log(result))    // Если resolve("успех")
    .catch(error => console.error(error))    // Если reject("обосрались")
    .finally(() => console.log("завершили")); // В ЛЮБОМ случае в конце
    
    ------------------------------
    
// async/await (синтаксический сахар)
async function fetchData() {
    try {
        const result = await promise;
        console.log(result);
    } catch (error) {
        console.error("поймали ошибку:", error);
    }
}

// Параллельное выполнение
const [a, b] = await Promise.all([promise1, promise2]); // Запускает промисы **параллельно** и ждет, пока ВСЕ завершатся

const winner = await Promise.race([promise1, promise2]); // Запускает промисы параллельно и возвращает результат ПЕРВОГО выполнившегося
```
## КЛАССЫ (ООП типо...) + наследование 

```js
class Animal {
    constructor(name) { / специальный метод, который вызывается автоматически при создании нового объекта через new -- эти типо как init в swift
        this.name = name;
    }
    
    speak() {
        console.log(`${this.name} издаёт звук`);
    }
    
    static staticMethod() { /Статический метод - это метод, который принадлежит самому классу, а не его экземплярам (объектам) = то есть вызывать можно через сам класс Animal
        console.log("метод класса, не доступен экземплярам"); 
    }
}
/использование
const animal = new Animal("Бобик");
animal.speak();     //  Бобик издаёт звук
animal.staticMethod();    // ❌ TypeError: animal.staticMethod is not a function
Animal.staticMethod();    // ✅ метод класса, не доступен экземплярам

--------

class Dog extends Animal { // это дочерний класс который наследуется от материнского
//Dog получит ВСЕ свойства и методы Animal (имя, speak и т.д.)

    constructor(name, breed) { Принимает два параметра name и  breed
        super(name); // обязательно вызывать родительский конструктор (родительский инит!) - должен быть ПЕРВЫМ в конструкторе
        
        this.breed = breed; / новое свойство
    }
    
    speak() {
        console.log(`${this.name} гавкает`); / переоперделил метод
    }
    
    get fullInfo() {   // Возвращает строку с именем и породой - геттер
        return `${this.name} (${this.breed})`;
    }
    
    set fullInfo(value) { // сеттер -  Разбивает строку типа "Шарик Овчарка" на имя и породу
        [this.name, this.breed] = value.split(" ");
    }
}

использую 

// Создаем собаку
const dog = new Dog("Шарик", "Овчарка");
// 1. Вызывается constructor("Шарик", "Овчарка")
// 2. super("Шарик") вызывает Animal.constructor("Шарик")
// 3. Animal создает свойство this.name = "Шарик"
// 4. Добавляется this.breed = "Овчарка"

console.log(dog.name);        // "Шарик" (от Animal)
console.log(dog.breed);       // "Овчарка" (свое свойство)
dog.speak();                  // "Шарик гавкает" (переопределенный метод)

// Используем геттер (как свойство!)
console.log(dog.fullInfo);    // "Шарик (Овчарка)"

// Используем сеттер (тоже как свойство!)
dog.fullInfo = "Бобик Такса";

console.log(dog.name);        // "Бобик"
console.log(dog.breed);       // "Такса"
console.log(dog.fullInfo);    // "Бобик (Такса)"
```
## ОШИБКИ (try/catch) обработчики ошибок


```js

try { // пробуем выполнить - если падаем - то выдаем блок catch

    throw new Error("чё-то пошло по пиде"); // бросаю ошибку =после throw код дальше НЕ ВЫПОЛНЯЕТСЯ, прыгаем в catch
    
} catch (error) {

    console.error(error.message);
    
} finally { // БЛОК, КОТОРЫЙ ВЫПОЛНИТСЯ ВСЕГДА

    console.log("выполнится всегда");
}


----------

try {
    console.log("1. Начинаем опасную операцию");
    throw new Error("чё-то пошло по пиде");
    console.log("2. Этот код НЕ ВЫПОЛНИТСЯ"); // ❌
    
} catch (error) {
    console.log("3. Ловим ошибку:", error.message);
    
} finally {
    console.log("4. Это выполнится ВСЕГДА");
}




------------------------------------------
#МОДУЛИ (ES6)

 Модули — это способ разделить код на разные файлы, чтоб не было свалки из 100500 строк в одном файле. 
 //Как в Swift разные файлы с классами и функциями

// export.js
export const PI = 3.14;       // Ящик с числом
export function sum(a, b) { return a + b; }     // Ящик с функцией
export default class MyClass {} // дефолтный экспорт (один на модуль)  // главный ящик (дефолтный)

// import.js
import MyClass, { PI, sum as add } from './export.js';
/sum as add - переименовали sum в add при импорте
/ MyClass - это default export (можно назвать как хочешь)
/ { PI, sum as add } - конкретные вещи, которые нужны


import * as all from './export.js'; // всё импортировать

--------как использовать?---------------

// После первого импорта:
const obj = new MyClass();        // ✅ работаем с классом
console.log(PI);                  // ✅ 3.14
console.log(add(2, 3));           // ✅ 5 (sum переименовали в add)

// После второго импорта (если сделали import *):
console.log(all.PI);              // ✅ 3.14
console.log(all.sum(2, 3));       // ✅ 5
const obj2 = new all.MyClass();   // ✅ работаем с классом

----------------------------------------

```

## Опциональная цепочка
```js
const city = user?.address?.city; // если user или address нет - вернет undefined
// Вместо: user && user.address && user.address.city
```

## Таймеры (кроме setTimeout)
**это ВСТРОЕННЫЙ ТАЙМЕР, который выполняет функцию ПОВТОРНО через заданные промежутки времени
```js

setInterval(() => {
    console.log('каждые 2 секунды');
}, 2000);

// Чтобы остановить:
const timer = setInterval(...);
clearInterval(timer);

------
пример

#Карусель/слайдер


const images = ['1.jpg', '2.jpg', '3.jpg', '4.jpg'];
let currentIndex = 0;

const slider = setInterval(() => {
    console.log(`Показываю слайд: ${images[currentIndex]}`);
    currentIndex = (currentIndex + 1) % images.length; // зацикливаем
}, 3000);

// Остановим через 15 секунд
setTimeout(() => {
    clearInterval(slider);
    console.log("Карусель закончилась");
}, 15000);


---------- пример пинг-понг с сервером -------
// Каждые 30 секунд проверяем, жив ли сервер
setInterval(async () => {
    try {
        const response = await fetch('https://мой-api.site.com/ping');
        if (response.ok) {
            console.log('✅ Сервер жив');
        } else {
            console.log('⚠️ Сервер барахлит');
        }
    } catch (error) {
        console.log('❌ Сервер недоступен');
    }
}, 30000);
```


## localStorage (сохраняем данные между сессиями)
```js
// Сохранить
localStorage.setItem('user', JSON.stringify(user));

// Достать
const savedUser = JSON.parse(localStorage.getItem('user'));

// Удалить
localStorage.removeItem('user');
```


## Шорткаты, которые используют везде
```js
// && вместо if
isAdmin && renderAdminPanel(); // выполнится только если isAdmin true

// || для дефолтных значений (старый способ)
const name = inputName || "Аноним";

// ?? только для null/undefined (новый, безопаснее)
const count = inputCount ?? 10; // если 0 или "" - останется 0 или ""

// Тернарник в строке
const status = age >= 18 ? 'взрослый' : 'мелкий';// как в swift прям!!
```

## Новые методы массивов
```js
// flatMap - мап + разглаживание
const words = ['hello', 'world'];
const letters = words.flatMap(word => word.split('')); // ['h','e','l','l','o','w','o','r','l','d']

// groupBy (новый) - группировка
const grouped = Object.groupBy(users, user => user.age > 18 ? 'adult' : 'child');
```

## Дебаггинг top ❤️‍🔥
```js
// Вместо console.log
console.log('тут всё сломалось', someVar);

// Используй:
console.table(arr); // красивая таблица для массивов объектов
console.time('загрузка'); // засечь время
// ... код ...
console.timeEnd('загрузка'); // показать сколько прошло

// Прямо в коде останавливаемся
debugger; // если открыта консоль - выполнение заморозится
```

## Дата и время
```js
const now = new Date();
// Форматирование
new Intl.DateTimeFormat('ru-RU', { 
    dateStyle: 'full', 
    timeStyle: 'long' 
}).format(now); // "пятница, 19 февраля 2026 г., 14:30:45 GMT+3"
```

## Регулярки для валидациииии
```js
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
emailRegex.test('test@mail.ru'); // true

/разбор
      /^[^\s@]+@[^\s@]+\.[^\s@]+$/
// 1.  ^                            - начало строки
// 2.    [^\s@]+                    - один или больше символов, но не пробел и не @
//                                   (это будет локальная часть email: "test")
// 3.            @                  - символ @
// 4.              [^\s@]+          - один или больше символов, не пробел и не @
//                                  (это домен: "mail")
// 5.                     \.        - точка (экранированная)
// 6.                       [^\s@]+ - один или больше символов, не пробел и не @
//                                   (это доменная зона: "ru")
// 7.                            $  - конец строки









---------

// Проверка телефона
const phoneRegex = /^\+7\d{10}$/;
```

## Современный синтаксис
```js
// Имена свойств из переменных
const field = 'age';
const user = {
    name: 'Петрович',
    [field]: 30, // динамическое имя свойства ЭТО computed properties // возьми значение переменной field ('age') и сделай это именем свойства
    [`is_${field}_valid`]: true // 'is_age_valid': true
};


пример --------

const propName = 'color';
const car = {
    brand: 'BMW',
    [propName]: 'black'  // создаст свойство 'color': 'black' позволяют использовать **ЗНАЧЕНИЕ ПЕРЕМЕННОЙ** как имя свойства объекта! зачем? хз пока 👍  !
};

console.log(car); // { brand: 'BMW', color: 'black' }
console.log(car.color); // 'black'





---------

// Приватные поля в классах (новое)
class BankAccount {
    #balance = 0; // реально приватное, только внутри класса
    
    deposit(amount) {
        this.#balance += amount;
    }
}
```