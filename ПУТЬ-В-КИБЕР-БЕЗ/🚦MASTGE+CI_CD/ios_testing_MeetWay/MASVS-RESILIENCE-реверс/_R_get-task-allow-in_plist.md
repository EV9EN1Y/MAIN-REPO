# MASTG-TEST-0261: в файле entitlements.plist включена возможность отладки

это очень простой тест - который тупо проверяет,  не включена ли случайно возможность отладки в приложении (например, в xcode дебаг версия - всегда включена) - но в продакшн все должно быть отклчюено

называется ключ в инфо-plist так: `get-task-allow`
и должен стоять либо FALSE - либо вообще должен отсутствовать 

но если он есть и значение TRUE - то в продакте такого быть не должно!

----------

проверить оч просто

1- в xcode в info глянуть ключ `get-task-allow`

2 - через радар посмотреть `r2 -c 'izz~get-task-allow; q' ./MeetWay.app/MeetWay.debug.dylib`

3 - codesign
проверить сразу в папке бинарной этот файл

`codesign -d --entitlements :- ./MeetWay.app | grep -E "get-task-allow|false|true"`

у меня вот так - так как это дебаг версия
```xml
evgeniy@Evgeniys-MacBook-Pro 4343434 % codesign -d --entitlements :- ./MeetWay.app | grep -E "get-task-allow|false|true"
Executable=/Users/evgeniy/Desktop/4343434/MeetWay.app/MeetWay
warning: Specifying ':' in the path is deprecated and will not work in a future release
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "https://www.apple.com/DTDs/PropertyList-1.0.dtd"><plist version="1.0"><dict><key>application-identifier</key><string>MF5MXG95XN.AIVARO22-2025-1.0</string><key>com.apple.developer.background-tasks.continued-processing.gpu</key><true/><key>com.apple.developer.team-identifier</key><string>MF5MXG95XN

</string><key>
get-task-allow  👈
</key>
<true/>         👈
</dict></plist>

то есть отладка полностью разрешена! 
например , можно легко подрубить LLDB и делать с функционалом приложения что угодно

```

Apple строго следит, чтобы в Production-сборках **НЕ БЫЛО** `get-task-allow = true`

- Если разработчик случайно отправит на ревью сборку с этим флагом (`true`), Apple **отклонит**приложение

Короче, как я понял этот ключ нужен только для того чтобы перед отправкой в App Store на публикацию приложения, не было никаких неожиданных отклонений. Потому что если телефон без Jailbreak, то отладчик к нему и так невозможно будет подключить к приложению,, а если на телефоне и установлен Jailbreak, то тогда и неважно какой там будет ключ стоять True или False, всё равно это можно будет легко обойти. Поэтому видимо, основная миссия данного теста это – просто на настроить правильно сборку перед публикацией в App Store чтобы не было неожиданного отклонения

Возможно даже не получится на сборку это отправить потому что при архивации проекта, скорее всего Apple сразу выдаст ошибку, мне так кажется

-------

