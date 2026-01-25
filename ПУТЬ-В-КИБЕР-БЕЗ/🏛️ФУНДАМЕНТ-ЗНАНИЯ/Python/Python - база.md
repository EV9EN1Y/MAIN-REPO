Зная хорошо Swift , должно быть не так сложно и питон освоить на уровне для сетевых скриптов!


В Python все коллекции по умолчанию гетерогенные — 
они могут хранить элементы любых типов одновременно. 
Это одно из ключевых отличий от Swift.
 еб*шки-воробушки, можно мне обратно на свифт!?



# Базовые конструкции

```Python
# Переменные и типы данных
target = "192.168.1.1"          # Строка
port = 8080                     # Целое число
is_vulnerable = True            # Булево значение
open_ports = [80, 443, 8080]    # Список
config = {"host": target, "port": port}  # Словарь /люб типы внутри

# Условия (проверка статуса)
if port == 80:
    print("HTTP-сервис")
elif 1024 < port < 49151:
    print("Зарегистрированный порт")
else:
    print("Нестандартный порт")

# Циклы (перебор объектов)
for port in open_ports:
    print(f"Проверяю порт {port}")  # f-строки (Python 3.6+)
    
# Функции
def check_vulnerability(url):
    """Проверяет уязвимость целевого URL"""
    # Здесь будет логика проверки
    return True if "vuln" in url else False

result = check_vulnerability("http://test.com/admin")
```

# Работа с сетевыми библиотеками

```Python
import socket
import requests

# Создание TCP-сокета
def port_scanner(host, port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(1)
    result = sock.connect_ex((host, port))
    sock.close()
    return result == 0  # True если порт открыт

# HTTP-запросы (самая частая операция)
response = requests.get("http://target.com", timeout=5)
print(f"Статус: {response.status_code}")
print(f"Заголовки: {response.headers}")
if "admin" in response.text:
    print("Найдена админка!")

# Отправка данных (POST для форм)
data = {"username": "admin", "password": "' OR '1'='1"}
response = requests.post("http://target.com/login", data=data)
```


# Работа с регулярными выражениями (парсинг ответов)
```Python
import re

# Поиск email в тексте
text = "Контакты: admin@target.com, support@site.ru"
emails = re.findall(r'[\w\.-]+@[\w\.-]+', text)
# ['admin@target.com', 'support@site.ru']

# Извлечение всех ссылок из HTML
links = re.findall(r'href="(https?://[^"]+)"', response.text)
```


# Логи
```Python
import logging
logging.basicConfig(level=logging.INFO)
logging.info(f"Начато сканирование {target}")
```


# ошибки обработка
```Python
try:
    response = requests.get(url, timeout=5)
except requests.exceptions.ConnectionError:
    print(f"[!] Не удалось подключиться к {url}")
```



# Цикл for
```Python
# 1. Перебор списка (самый частый случай)
open_ports = [22, 80, 443, 8080]
for port in open_ports:
    print(f"Проверяю порт {port}")

# 2. Перебор словаря
config = {"host": "192.168.1.1", "port": 80, "timeout": 5}
for key, value in config.items():  # .items() возвращает пары ключ-значение
    print(f"{key} = {value}")

# 3. С range() для числовых диапазонов (сканирование портов 1-1024)
for port in range(1, 1025):  # от 1 до 1024 (включительно)
    if port_scanner(target, port):
        print(f"Открыт порт: {port}")

# 4. С enumerate() если нужен индекс
payloads = ["' OR '1'='1", "<script>alert(1)</script>", "../../etc/passwd"]
for index, payload in enumerate(payloads):
    print(f"Тестирую payload #{index}: {payload}")
```




# Цикл while
```Python
# 1. Попытка подключения до успеха (макс 5 попыток)
attempts = 0
max_attempts = 5
connected = False

while not connected and attempts < max_attempts:
    print(f"Попытка подключения {attempts + 1}...")
    connected = port_scanner(target, 22)  # наша функция из прошлого ответа
    attempts += 1
    time.sleep(1)  # пауза 1 секунда между попытками

# 2. Чтение файла построчно до конца
with open("wordlist.txt", "r") as f:
    line = f.readline()
    while line:
        print(f"Тестирую: {line.strip()}")
        line = f.readline()

# 3. Бесконечный цикл с break (осторожно!)
import time
while True:
    response = requests.get("http://target.com/health")
    if response.status_code != 200:
        print("[!] Сервер упал!")
        break
    time.sleep(60)  # проверка каждые 60 секунд
```



# Управление циклами: break, continue, else
```Python
# break - досрочный выход из цикла
for port in range(1, 1000):
    if port == 80:
        print("Найдён HTTP-порт, прекращаю сканирование")
        break  # выходит из цикла полностью

# continue - пропуск текущей итерации
for port in [21, 22, 80, 443, 8080]:
    if port in [21, 22]:  # пропускаем FTP и SSH
        continue  # переходим к следующей итерации
    scan_service(port)  # проверяем только веб-сервисы

# else в цикле - выполняется если не было break
for port in known_web_ports:
    if is_vulnerable(port):
        print(f"Найдена уязвимость на порту {port}")
        break
else:  # выполнится только если НИ ОДНА уязвимость не найдена
    print("Уязвимостей на веб-портах не обнаружено")
```


# словарь сравнение 
```Python
# Python (словарь)
params = {"user": user, "pass": password}
# Тип: dict (сокращение от dictionary)

# Фигурные скобки {} вместо квадратных [] для словаря

# Swift (словарь)
let params: [String: Any] = ["user": user, "pass": password]
```




# Что значит f перед строкой: f"[+] Успех! {user}:{password}"
```Python
# Python (f-строка)
user = "admin"
password = "123456"
message = f"[+] Успех! {user}:{password}"
# Результат: "[+] Успех! admin:123456"

# Swift (строковая интерполяция)
let user = "admin"
let password = "123456"
let message = "[+] Успех! \(user):\(password)"
```



# Ох! отступы КРИТИЧЕСКИ важны в Python! 
```Python
# Python (отступы = 4 пробела стандартно)
if port == 80:
    print("HTTP порт")  # Входит в if
    scan_http()         # Входит в if
print("Сканирование завершено")  # Уже ВНЕ if (нет отступа)

# Swift (фигурные скобки)
if port == 80 {
    print("HTTP порт")  // Входит в if
    scanHTTP()          // Входит в if
}
print("Сканирование завершено")  // Вне if

#Правила:

#Используйте 4 пробела на уровень отступа (рекомендация)
#Никогда не смешивайте табы и пробелы
#Все операторы в одном блоке должны иметь одинаковый отступ
#Конец блока определяется уменьшением отступа
# мда

```



# начало и конец циклов
```Python
# Цикл FOR
for port in ports:                     # ← НАЧАЛО цикла (двоеточие!)
    print(f"Сканирую {port}")          # ← ТЕЛО цикла (отступ!)
    result = scan(port)                # ← ТЕЛО цикла (тот же отступ!)
print("Цикл завершён")                 # ← УЖЕ ВНЕ цикла (меньший отступ!)

# Цикл WHILE
while not connected:                   # ← НАЧАЛО (двоеточие!)
    print("Попытка подключения...")    # ← ТЕЛО (отступ!)
    connected = try_connect()          # ← ТЕЛО (тот же отступ!)
print("Подключено!")                   # ← ВНЕ цикла


```


# Вложенные циклы
```Python
for user in users:                     # ← ВНЕШНИЙ цикл
    for password in passwords:         # ← ВНУТРЕННИЙ цикл (больший отступ)
        attempt = try_login(user, password)  # Тело внутреннего цикла
        if attempt:                    # ← Условие ВНУТРИ внутреннего цикла
            print("Успех!")            # ← Тело условия
    print(f"Пользователь {user} проверен")  # ← Вне внутреннего, но во внешнем цикле
```


# Сравнение коллекций Swift ↔ Python
```Python
Список на Swift (List)	Array<Element>	 /// Список на Python list = [1, 2, 3] индексы с 0

Кортеж Swift  (Tuple)	(Element, Element) /// Кортеж  Python	tuple = (1, 2, 3)	Неизменяемый после создания, быстрее списка

Словарь  Swift (Dict)	Dictionary<Key, Value> /// Словарь Python	dict = {"key": "value"}	Ключи любые хешируемые типы, порядок с Python 3.7

Множество Swift (Set)	Set<Element> //// Множество Python	set = {1, 2, 3}	Только уникальные элементы, нет порядка
```

# примеры коллекций и операциии

# Список (List) 
```Python
# Создание
payloads = ["' OR '1'='1", "<script>alert(1)</script>", "../../etc/passwd"]
open_ports = []  # пустой список

# Методы (аналоги Array в Swift)
payloads.append("<?php system($_GET['cmd']); ?>")  # добавить в конец
last_payload = payloads.pop()  # удалить и вернуть последний элемент
payloads.insert(0, "test")  # вставить на позицию
if "' OR " in payloads:  # проверка наличия
    print("SQLi payload найден")

# Срезы (слайсы) — мощная фишка Python
first_three = payloads[:3]  # первые 3 элемента
without_first = payloads[1:]  # все кроме первого
reverse = payloads[::-1]  # развернуть список
```




# Кортеж (Tuple)
```Python
# Создание (часто без скобок)
target = "192.168.1.1", 80  # это кортеж!
config = ("admin", "pass123", 5)  # логин, пароль, timeout

# Использование
host, port = target  # распаковка (destructuring)
print(f"Сканирую {host}:{port}")

# Кортеж vs список
coordinates_list = [10, 20]  # можно изменить
coordinates_list[0] = 15  # OK

coordinates_tuple = (10, 20)  # НЕЛЬЗЯ изменить
# coordinates_tuple[0] = 15  # ОШИБКА! TypeError
```




# Словарь (Dict)
```Python
# Создание
config = {
    "target": "192.168.1.1",
    "ports": [80, 443, 8080],
    "timeout": 5,
    "verbose": True
}

# Методы
config["threads"] = 10  # добавить/изменить
del config["verbose"]  # удалить ключ
ports = config.get("ports", [])  # безопасное получение (если нет — [])
if "target" in config:  # проверка ключа
    print(f"Цель: {config['target']}")

# Итерация
for key, value in config.items():  # .items() вместо .forEach
    print(f"{key}: {value}")
```




# Множество (Set)
```Python
# Создание
unique_ips = {"192.168.1.1", "10.0.0.1", "192.168.1.1"}  # дубли удаляются
vulnerabilities = set()  # пустое множество

# Методы
unique_ips.add("172.16.0.1")  # добавить
unique_ips.remove("10.0.0.1")  # удалить (ошибка если нет)
unique_ips.discard("10.0.0.1")  # удалить (без ошибки)

# Операции над множествами (отличная фишка!)
scanned = {"192.168.1.1", "10.0.0.1"}
vulnerable = {"192.168.1.1", "172.16.0.1"}

common = scanned & vulnerable  # пересечение (AND)
all_ips = scanned | vulnerable  # объединение (OR)
only_scanned = scanned - vulnerable  # разность
```





# конвертация типов
```Python
# Часто используется в пентесте
port_list = [80, 443, 80, 8080]
unique_ports = set(port_list)  # [80, 443, 8080] - убрали дубли

ips_tuple = ("192.168.1.1", "10.0.0.1")
ips_list = list(ips_tuple)  # можно изменить

config_list = [["host", "target.com"], ["port", 80]]
config_dict = dict(config_list)  # {"host": "target.com", "port": 80}
```






```Python

```







```Python

```







```Python

```







```Python

```


