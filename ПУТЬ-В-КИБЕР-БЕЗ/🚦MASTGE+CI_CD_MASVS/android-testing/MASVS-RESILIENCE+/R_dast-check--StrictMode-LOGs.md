https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0263/
# MASTG-TEST-0263: Logging of StrictMode Violations

тест провожу на приложении 👉 [[0_MeetWay]]

ДАННЫЙ ТЕСТ про то, не оставили ли разработчики включенным режим **StrictMode** в продакшн-версии приложения


**StrictMode** - это инструмент разработчика в Android, который помогает находить проблемы в коде, например, выполнение медленных операций ввода-вывода или сетевых запросов в главном потоке (UI-потоке). Когда StrictMode включен, приложение может логировать такие нарушения, показывать предупреждения или даже крашиться, чтобы разработчик заметил проблему

В продакшн-версии этот режим должен быть выключен, потому что:

1. Он создает лишнюю нагрузку на устройство, замедляя работу приложения
   
2. Логи StrictMode могут содержать детали реализации приложения (например, названия файлов, структуру базы данных, сетевые запросы). Злоумышленник, имеющий доступ к логам устройства (например, через ADB или сторонние приложения для чтения логов), может использовать эту информацию для понимания внутреннего устройства приложения и поиска уязвимостей
   

Тест проверяет, нет ли в продакшн-сборке логов StrictMode

--------
тест выполняется с помощью мониторинга системных логов на устройстве с установленным приложением

-------

меня интресуют логи связанные с    StrictMode

поэтому 

сперва подкл телефон  к пк проводом 
в терминале пк 
```q
adb devices

вижу девайс

проверка работоспособности

adb logcat -d | head -n 10
```

вижу в выводе 10 строк логов, -начит можно приступать к делу!

1 - очистка старых логов:
```sh
adb logcat -c
```


запускаю проверку логов с фильтром  StrictMode
```sh
adb logcat -s StrictMode
```


и теперь запустил приложение, и полностью использовал весь его функционал. нажал на каждую кнопку и был на каждом экране!

```
evgeniy@Evgeniys-MacBook-Pro-2 ~ % adb devices
List of devices attached
6b62e119	device

evgeniy@Evgeniys-MacBook-Pro-2 ~ % adb logcat -c
evgeniy@Evgeniys-MacBook-Pro-2 ~ % adb logcat -s StrictMode

```

но выводов в логах нет, значит, тест пройден, логи StrictMode не обнаружены!

--------

для самопроверки просомтрю логи запущенного приложения

```q
 adb shell ps | grep meetway
 
 вижу:
 
u0_a300      26253  1114    6724676 207500 0                   0 S com.evgeniy.meetway

запускаю просмотр логов:

 adb logcat --pid=26253
 
 ну и вижу тут кучу логов 
 
 6-26 17:07:42.360 26253 26253 I System.out: ❌ Биометрия не настроена – пропускаем
06-26 17:07:42.361 26253 26285 I okhttp.OkHttpClient: --> POST https://functions.yandexcloud.net/d4efavboiige7c7leqkf/
06-26 17:07:42.361 26253 26285 I okhttp.OkHttpClient: Content-Length: 330
06-26 17:07:42.362 26253 26285 I okhttp.OkHttpClient: Content-Type: application/json
06-26 17:07:42.362 26253 26253 I Quality : Skipped: false 2 cost 44.148434 refreshRate 16606013 bit true processName com.evgeniy.meetway
06-26 17:07:42.362 26253 26285 I okhttp.OkHttpClient: {"action":"getUserChatsWithPreview","jwt":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ5YW5kZXhfaWQiOiIxNDk3MjQzNzgyIiwibG9naW4iOiJzb2FwMjIyMjIyIiwiZW1haWwiOiJzb2FwMjIyMjIyQHlhbmRleC5ydSIsIm5hbWUiOiLQldCy0LPQtdC90LjQuSDQp9C10YDQvdC40LrQvtCyIiwiaWF0IjoxNzgyNDc2NDU1LCJleHAiOjE3ODI1NjI4NTV9.EklQF9nPVkeykf40-DCgBdy11TaMlrZMcrUphW8DvDo"}

помимо всего тут еще и jwt живет в логах ))) ну это уже другой тест, и конечно, крит уяза!

```


---------

## Пример кода для включения StrictMode
```js
// Пример кода для включения StrictMode
StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
        .detectAll()
        .penaltyLog()
        .build());

StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
        .detectAll()
        .penaltyLog()
        .build());
```


------

### вывод:

### тест пройден , в логах нет упоминания StrictMode !


