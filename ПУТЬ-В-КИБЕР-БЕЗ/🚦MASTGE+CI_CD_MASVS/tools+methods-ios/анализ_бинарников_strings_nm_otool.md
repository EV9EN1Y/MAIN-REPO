## 🍺 strings  Извлечение читаемых строк

**Назначение:** Вытаскивает из бинарника все ASCII/UTF-8 последовательности. Находит URL, ключи, пути, имена классов, сообщения.

**Базовая форма:**

`strings -a -n <min_length> <file>`

###### Ключевые флаги

```
|Флаг|Эффект|

|-a|Сканировать весь файл, а не только секции|
|-n <N>|Минимальная длина строки (умолчание 4)|
|-8|UTF-8 (то же что `-n 8`)|
|-|Читать из stdin|
```
###### Практические команды

Поиск всех HTTP/HTTPS URL:
`strings -a -8 MeetWay.debug.dylib | grep -E "https?://"`

Поиск API ключей и токенов:
`strings -a -8 MeetWay.debug.dylib | grep -E "key|secret|token|api|auth" -i`

Поиск доменов (без протокола):
`strings -a -8 MeetWay.debug.dylib | grep -E "[a-zA-Z0-9.-]+\.[a-z]{2,}" | sort -u`

Поиск путей к файлам:
`strings -a -8 MeetWay.debug.dylib | grep -E "/[a-zA-Z0-9/_\-]+"`

Поиск Cloud-сервисов (AWS, Firebase, Yandex):
`strings -a -8 MeetWay.debug.dylib | grep -E "s3|amazonaws|firebase|yandexcloud|storage" -i`

Рекурсивный поиск по всем файлам в .app:
`find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -q "http://" && echo "$0"' {} \;`

---

## 🍺 nm - Таблица символов

**Назначение:** Показывает имена функций и глобальных переменных. Обнаруживает импорт опасных API (socket, connect, send).

**Базовая форма:**

`nm <flags> <binary>`
###### Типы символов (критически важные)

```
|Код|Значение|

|U|Undefined — импортируется извне (системные вызовы)|
|T|Text — определена в этом бинарнике|
|D|Data — глобальная переменная|
|S|Symbol for section|
```

###### Ключевые флаги

```
|Флаг|Эффект|

|-u|Показать только неопределенные символы (U)|
|-g|Только глобальные символы|
|-m|Mach-O формат (показать библиотеку-источник)|
|-U|Не показывать undefine символы|
```
### Практические команды

Найти импорт низкоуровневых сетевых функций:
`nm -u MeetWay.debug.dylib | grep -E "socket|connect|send|recv|close|shutdown"`

Найти импорт CFNetwork/Network.framework:
`nm -u MeetWay.debug.dylib | grep -E "CFNetwork|NWConnection|nw_"`

Найти все импорты с указанием библиотеки:
`nm -mu MeetWay.debug.dylib | grep "U" | head -30`

Рекурсивно по всему .app:
`find . -type f -exec sh -c 'nm -u "$0" 2>/dev/null | grep -q "socket" && echo "$0"' {} \;`

Отфильтровать Swift-манглинг (символы с `$s`):
`nm -u MeetWay.debug.dylib | grep -E "^_+[a-z]" | grep -v "\$s"`

Показать определенные функции (Text segment):
`nm -T MeetWay.debug.dylib | head -50`

---

## 🍺 otool - Анализ Mach-O структуры

**Назначение:** Интроспекция бинарного формата. Показывает зависимости от фреймворков, сегменты, загрузочные команды.

**Базовая форма:**

`otool <flags> <binary>`

### Ключевые флаги

```
|Флаг|Эффект|

|-L|Динамические библиотеки (зависимости)|
|-I|Импортируемые символы (аналог nm)|
|-l|Загрузочные команды (load commands)|
|-tv|Дизассемблировать текст (ассемблер)|
|-V|Вербозный вывод|
|-arch|Выбрать архитектуру (arm64/armv7)|
```

### Практические команды

Показать все динамические библиотеки:
`otool -L MeetWay.debug.dylib`

Найти сетевые фреймворки:
`otool -L MeetWay.debug.dylib | grep -E "CFNetwork|Network|Security"`

Показать импорты (подробно):
`otool -Iv MeetWay.debug.dylib | head -50`

Рекурсивный поиск фреймворков по всему .app:
`find . -name "*.dylib" -o -name "MeetWay" | xargs otool -L 2>/dev/null | grep -E "CFNetwork|Network" -B 1`

Показать архитектуры в fat binary:
`otool -arch arm64 -L MeetWay.debug.dylib`

Дизассемблировать конкретную функцию (требуется знание адреса):
`otool -tv MeetWay.debug.dylib | grep -A 20 "_connect"`

Показать сегменты и секции:
`otool -l MeetWay.debug.dylib | grep -A 5 "sectname"`

---

🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶-🔶
## комбо техники 🍺🍺🍺

### полный анализ одного бинарника

```bash
BIN="MeetWay.debug.dylib"
echo "=== strings (URL) ===" && strings -a -8 $BIN | grep -E "https?://"
echo "=== strings (keys) ===" && strings -a -8 $BIN | grep -iE "key|secret|token"
echo "=== nm (sockets) ===" && nm -u $BIN | grep -E "socket|connect|send|recv"
echo "=== nm (CFNetwork) ===" && nm -u $BIN | grep -i "CFNetwork\|NW"
echo "=== otool (deps) ===" && otool -L $BIN | grep -E "CFNetwork|Network"
```
### Поиск опасных вызовов во всех .dylib

```
find ./Frameworks -name "*.dylib" -exec sh -c 'nm -u "$0" 2>/dev/null | grep -q "socket\|connect" && echo "DANGER: $0"' {} \;
```

### Игнорирование Swift-манглинга при поиске

`nm -u MeetWay.debug.dylib | grep -E "socket|connect|send|recv" | grep -v "\$s"`

### Сравнение символов между debug и release сборками

`comm -23 <(nm -u MeetWay.debug.dylib | sort) <(nm -u MeetWay | sort)`

---
## Автоматизация: однострочники для отчета

Собрать все IP-адреса из бинарника:
`strings -a -8 MeetWay.debug.dylib | grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort -u`

Найти все JWT-токены:
`strings -a -8 MeetWay.debug.dylib | grep -E "eyJ[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*"`

Список всех Objective-C классов:
`strings -a -8 MeetWay.debug.dylib | grep -E "^_OBJC_CLASS_\$_"`

Список всех Swift-классов (деманглинг):
`strings -a -8 MeetWay.debug.dylib | grep -E "\$s.*C$" | sed 's/.*\$s//' | cut -d'C' -f1`

----------