# MASTG-TEST-0230: 
когда автоматическое подсчетное управление (ARC) не включено

(в swift - всегда включено)

Этот тест проверяет, включена ли в приложениях для iOS функция ARC (автоматическое управление памятью) ARC - это функция компилятора в Objective-C и Swift, которая автоматизирует управление памятью, снижая вероятность утечек памяти и других связанных с этим проблем. Включение ARC крайне важно для обеспечения безопасности и стабильности приложений для iOS

- **Код на Objective-C:** ARC можно включить, скомпилировав код с флагом `-fobjc-arc` в Clang
- **Код на Swift:** ARC включен по умолчанию
- **Код на C/C++:** ARC не применим, так как он предназначен только для Objective-C и Swift

Если включена функция ARC, в двоичных файлах будут присутствовать такие символы, как `objc_autorelease` или `objc_retainAutorelease`

> Причём если приложение полностью у нас Swift написано, то его и выключить нельзя есть ARC там работает всегда

------

приступаю к тесту

тестирование на приложении MeetWay
👉 (о приложении) [[0_MeetWay]]

Так как подписки Apple Developer у меня сейчас нет, поэтому я просто через Xcode установил приложение на телефон потыкал его, и теперь в папке Delivery Data я могу найти файлы моего приложения здесь

~/Library/Developer/Xcode/DerivedData/

 И там нахожу в билдах моего приложения нужного бинарник он будет называться также как название приложения с расширением.app, ну и весить много, а внутри бинарщина

-------
Перекинул на рабочий стол бинарник MeetWay.app

открываю папку
```q
cd /Users/evgeniy/Desktop/MeetWay.app

и смотрю, че там по файлам внутри есть (для инфы, чтобы понять, что попал по адрессу)
```

для понимания, где нахожусь и че внутри!

```c
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la
total 124904
-rwxr-xr-x   1 evgeniy  staff     35024  9 апр 10:07 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96  7 апр 00:34 _CodeSignature
drwxr-xr-x@ 17 evgeniy  staff       544 14 апр 13:51 .
drwx------@ 39 evgeniy  staff      1248 14 апр 22:10 ..
-rw-r--r--@  1 evgeniy  staff         0 11 апр 10:39 aaaa
-rw-r--r--   1 evgeniy  staff     14412  7 апр 01:18 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132  7 апр 01:18 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696  7 апр 01:19 Assets.car
-rw-r--r--   1 evgeniy  staff     15136  7 апр 00:25 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864  7 апр 00:34 Frameworks
-rw-r--r--   1 evgeniy  staff       996  7 апр 00:25 GoogleService-Info.plist
-rw-r--r--@  1 evgeniy  staff      5458 14 апр 13:51 Info_readable.plist
-rw-r--r--   1 evgeniy  staff      3717  7 апр 00:26 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664  9 апр 10:07 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41070784  9 апр 10:07 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8  7 апр 00:26 PkgInfo
-rw-r--r--   1 evgeniy  staff     49852  7 апр 00:25 words.txt
```


## как провести тест:

нужно проверить бинарник на objc_autorelease и  objc_retainAutorelease
если есть objc_autorelease и  objc_retainAutorelease - значит - все окей!
### базовые команды проверки (рекурсивно по всем библиотекам)

###### Проверка главного бинарника

`nm -u ~/Desktop/MeetWay.app/MeetWay | grep objc_autorelease`
`nm -u ~/Desktop/MeetWay.app/MeetWay | grep objc_retainAutorelease`

---
###### Команда 2: Проверка всех .dylib и .framework (рекурсивно)

`find ~/Desktop/MeetWay.app -name "*.dylib" -o -name "*.framework" -type d | while read lib; do echo "=== $lib ==="; nm -u "$lib" 2>/dev/null | grep objc_autorelease; done`

`find ~/Desktop/MeetWay.app -name "*.dylib" -o -name "*.framework" -type d | while read lib; do echo "=== $lib ==="; nm -u "$lib" 2>/dev/null | grep objc_retainAutorelease; done`

---
###### Команда 3: Проверка всех Mach-O файлов (включая скрытые)

`find ~/Desktop/MeetWay.app -type f -perm +111 -exec sh -c 'file "$0" | grep -q Mach-O && echo "=== $0 ===" && nm -u "$0" 2>/dev/null | grep objc_autorelease' {} \;`

`find ~/Desktop/MeetWay.app -type f -perm +111 -exec sh -c 'file "$0" | grep -q Mach-O && echo "=== $0 ===" && nm -u "$0" 2>/dev/null | grep objc_retainAutorelease' {} \;`

---
###### Команда 4: Альтернативная проверка через otool (для главного бинарника)

`otool -Iv ~/Desktop/MeetWay.app/MeetWay | grep objc_retainAutorelease`
`otool -Iv ~/Desktop/MeetWay.app/MeetWay | grep objc_autorelease`

-----
#### результаты теста

тест успешно пройден
так как много где были найдены включения целевых строк

например вот здесь

- GoogleDataTransport
- FirebaseFirestore
- GTMSessionFetcher
- AWSS3
- FirebaseCore
- GoogleUtilities
- FirebaseAuth
- AWSCore
- FirebaseFirestoreInternal
- YandexLoginSDK
- FirebaseInstallations
- MeetWay.debug.dylib

а вот тест nm -u - не нашел ничего, значит, бинарник сам по себе написан на swift, где по умолчанию arc включен! Причём если приложение полностью у нас Swift написано, то его и выключить нельзя есть ARC там работает всегда

------------

