Эта техника используется для того, чтобы сделать множество запросов к целевому ресурсу, не будучи заблокированным по IP-адресу. Это критично при:

- **Брутфорсе паролей** на формах входа
    
- **Фаззинге** (переборе) параметров и путей
    
- **Обходе rate limiting** (ограничений по частоте запросов)
    
- **Обходе WAF**, который банит по IP

-------


###  🟢🟢🟢 Метод 1: AWS API Gateway (самый популярный и надёжный)

Суть метода в том, что ты создаёшь несколько API Gateway-энпоинтов в разных регионах AWS, и каждый твой запрос идёт через случайный энпоинт [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate). Так как IP-адреса у AWS API Gateway берутся из огромного пула, твой IP будет меняться на каждом запросе [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator).

**Преимущества:**

- Огромный пул IP-адресов
    
- Высокая скорость работы
    
- Почти бесплатно (первые 1 млн запросов на регион в месяц бесплатны) [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)
    
- Автоматическая ротация на каждый запрос (IP меняется на каждом запросе) [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)
    

**Недостатки:**

- Запросы легко идентифицируются по заголовкам AWS (например, `X-Amzn-Trace-Id`) и могут быть заблокированы целенаправленно [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)
    
- Нужен аккаунт AWS и ключи доступа
    

**Инструменты:**

|Инструмент|Применение|Ссылка|
|---|---|---|
|**IP Rotate (Burp Suite Extension)**|Расширение для Burp Suite, создаёт энпоинты в AWS и ротирует IP на каждый запрос|[](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)|
|**requests-ip-rotator (Python)**|Python-библиотека для автоматической ротации IP через AWS API Gateway|[](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)|

**Настройка через Burp Suite:**

1. Установи расширение **IP Rotate** в Burp Suite [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)
    
2. Введи свои AWS-ключи (Access Key ID и Secret) [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)
    
3. Укажи целевой домен
    
4. Выбери протокол (HTTP или HTTPS)
    
5. Отметь нужные AWS-регионы (чем больше, тем шире пул IP)
    
6. Нажми **Enable** – расширение создаст энпоинты и начнёт ротировать IP [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)
    
7. После завершения работы нажми **Disable**, чтобы удалить созданные ресурсы [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate)
    

**Настройка через Python:**


```q
import requests
from requests_ip_rotator import ApiGateway
gateway = ApiGateway("https://site.com")
gateway.start()
session = requests.Session()
session.mount("https://site.com", gateway)
response = session.get("https://site.com/index.php")
print(response.status_code)
gateway.shutdown()  # обязательно удали энпоинты, чтобы не платить
```




----------

### 🟢🟢🟢🟢🟢🟢 Метод 2: Tor Network (бесплатно, но медленно)

Ты используешь сеть Tor как прокси и меняешь свой exit-узел (а значит и IP) через определённые интервалы или по команде 

**Преимущества:**

- Бесплатно
- Высокая анонимность

**Недостатки:**

- Низкая скорость (запросы идут через несколько узлов)
   
- Многие сайты и WAF блокируют известные выходные ноды Tor
   
- Запросы медленные
   

**Инструменты:**

|Инструмент|Применение|Ссылка|
|---|---|---|
|**IPChanger (CLI)**|Меняет IP через Tor с заданным интервалом, выводя новый IP в консоль|[](https://github.com/Cappricio-Securities/ipchanger/#start-of-content)|
|**Tor-ip-changer**|Меняет IP через `NEWNYM`-сигнал без перезапуска Tor, устанавливается как systemd-сервис|[](https://github.com/abdullah-x909/Tor-ip-changer)|
|**IPSpoofer**|Python-скрипт с автоматической установкой Tor и настройкой прокси|[](https://github.com/OnyxDeveploment/IPSpoofer)|
|**Zero-Tor (GUI)**|Инструмент с графическим интерфейсом для управления Tor и ротации IP|[](https://github.com/zeroxorv/Zero-Tor)|

**Настройка Tor:**

1. Установи Tor:
    
    - Linux: `sudo apt install tor`
        
    - Mac: `brew install tor`
        
    - Windows: скачай с сайта Tor Project
        
2. Настрой прокси-сервер в системе или приложении на `socks5://127.0.0.1:9050` [](https://github.com/Cappricio-Securities/ipchanger/#start-of-content)[](https://github.com/OnyxDeveploment/IPSpoofer)
    
3. Для автоматической ротации используй один из инструментов выше



------------


### 🟢🟢🟢🟢🟢🟢🟢🟢🟢 Метод 3: Пулы прокси-серверов (гибко, но сложно)

Ты собираешь список рабочих прокси (HTTP, SOCKS4, SOCKS5) и отправляешь каждый запрос через случайный прокси из этого списка [](https://github.com/mubeng/mubeng).

**Преимущества:**

- Полный контроль над пулом IP
    
- Можно использовать прокси из разных стран
    
- Высокая скорость (если использовать быстрые прокси)
    

**Недостатки:**

- Нужно постоянно обновлять список рабочих прокси
    
- Хорошие прокси часто платные
    
- Сложность в поддержании и проверке
    

**Инструменты:**

|Инструмент|Применение|Ссылка|
|---|---|---|
|**mubeng**|Быстрый прокси-чекер и ротатор; проверяет прокси, фильтрует по стране, автоматически ротирует|[](https://github.com/mubeng/mubeng)|
|**Burp Proxy Rotate**|Расширение для Burp Suite, которое ротирует IP через список прокси|(ищи в BApp Store)|

**Настройка через mubeng:**

1. Создай файл `proxies.txt` со списком прокси
    
2. Запусти проверку и отфильтруй живые:
    

 ```q
   mubeng -f proxies.txt --check --output live.txt
 ```
    
3. Запусти прокси-сервер с ротацией:
    

  ```c
   mubeng -f live.txt -a 127.0.0.1:8080
  ```
    




### 🔵 Метод 4: Ротация на уровне ОС (для локальной сети)

Этот метод меняет IP-адрес самого компьютера в локальной сети (через перезапуск сетевого интерфейса или MAC-адреса), но **не меняет внешний IP**, который видит сайт [](https://github.com/MilesPhoka/IP-Vortex). Он бесполезен для обхода блокировок в интернете, поэтому я его не рассматриваю

--------

## 3. Сравнительная таблица

|Метод|Сложность|Скорость|Стоимость|Эффективность обхода WAF|
|---|---|---|---|---|
|**AWS API Gateway**|Средняя|Высокая|Почти бесплатно [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)|Высокая, но легко детектится [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)|
|**Tor Network**|Низкая|Низкая|Бесплатно|Низкая (многие блокируют Tor)|
|**Пул прокси**|Высокая|Высокая|От бесплатно до дорого|Зависит от прокси|

## 4. Важные нюансы

1. **AWS-запросы легко детектятся** по заголовкам, и их могут целенаправленно блокировать
   
2. **Tor часто блокируется** на уровне WAF и сайтов
   
3. **Прокси должны быть рабочими** - используй инструменты для проверки (как `mubeng`)
   
4. **Всегда чисть за собой AWS-ресурсы** (disable gateways) после использования, чтобы не платить 
   
5. **Используй эти техники этично** - только на своих ресурсах или с письменного разрешения владельца
   

## 5. Рекомендуемый сетап для разных задач

| Задача                           | Рекомендуемый метод | Инструмент                                                                                                                                                                                                                       |
| -------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Брутфорс в Burp Suite**        | AWS API Gateway     | IP Rotate (Burp Extension) [](https://portswigger.net/bappstore/2eb2b1cb1cf34cc79cda36f0f9019874#:~:text=This%20extension%20allows%20you%20to,be%20different%20on%20each%20request.)[](https://github.com/portswigger/ip-rotate) |
| **Скрипт на Python**             | AWS API Gateway     | requests-ip-rotator [](https://pypi.org/project/requests-ip-rotator/)[](https://github.com/Ge0rg3/requests-ip-rotator)                                                                                                           |
| **Быстрый и бесплатный вариант** | Tor                 | IPChanger [](https://github.com/Cappricio-Securities/ipchanger/#start-of-content)                                                                                                                                                |
| **Гибкий пул прокси**            | Свой пул прокси     | mubeng [](https://github.com/mubeng/mubeng)                                                                                                                                                                                      |

---

-----

