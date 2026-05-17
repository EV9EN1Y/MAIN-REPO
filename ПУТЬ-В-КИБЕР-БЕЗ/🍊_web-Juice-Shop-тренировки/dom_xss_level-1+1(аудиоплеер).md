#### juice-shop
через докер у себя локально на маке 
```q
docker run --rm -p 127.0.0.1:3003:3000 --name juice-shop bkimminich/juice-shop
```
ну и локально открываю
http://localhost:3001

-----

этот тест продолжение теста [[dom_xss_level-1]]

тут просто - идет продолжение эксплуатации обычной самой банальной XSS

по заданию дают ссылочку на плеер

```js
<iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>
```

<img src="../../assets/Снимок2026-05-1717.51.40.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

ну собственно все ) хз конечно ) - типо возможности xss посмотреть, но это и так ясно, что можно любой код внедрить! абсолютно!
### то как нашел эту уязвимость и выводы по ней вот тут нажать 👉 [[dom_xss_level-1]]
