
«безопасность по неизвестности» - когда url странные, кривые и запутанные с неожиданными URL , но эти URL могут давать доступ к чему - либо

например админка по адресу
`https://insecure-website.com/administrator-panel-yb556`
хрен подберешь такое, но если этот адресс где-то всплывает, в html или JS , или через сканирование - то досвидос


лаба https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url

задание - попасть в админку и удалить карлоса

-------

на главное стр вижу это

```html
<script>

var isAdmin = false;

if (isAdmin) {
   var topLinksTag = document.getElementsByClassName("top-links")[0];
   var adminPanelTag = document.createElement('a');
   
   👉
  👉 adminPanelTag.setAttribute('href', '/admin-behc8g');
   👉
   
   adminPanelTag.innerText = 'Admin panel';
   topLinksTag.append(adminPanelTag);
   var pTag = document.createElement('p');
   pTag.innerText = '|';
   topLinksTag.appendChild(pTag);
}

</script>
```


перехожу по 
```
https://0ad60008039e3f3e8070497f009400be.web-security-academy.net/admin-behc8g
```

получаю доступ к админке - удаляю карлоса 
#### лаба решена

# вывод
не хранить вообще никакой служебной и доп инфы в открытом коде
и вообще, в продакте вообще не хранить такие вещи.. детская лаба
это было первое

а второе 
то что без логина и пароля попал в админ панель
