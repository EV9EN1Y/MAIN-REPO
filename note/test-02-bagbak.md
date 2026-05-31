поставил купер приложение на айфон 8 ios 16.7.14 - c palerain
на телефоне и на маке фрида Frida 17.9.1

приложение купер запускается с джейлбрейком и без него 
(детекции нет или отключена)

-------
смотрю запущ процессы
`frida-ps -Uai`

вижу 
 Купер           ru.insttek.kuperapp

------

## качаю расшифрованный бинарник

кидаю ssh

перв терминал
`iproxy 44444 44`
второй терминал
`ssh -p 44444 root@localhost`
стандарт пароль: alpine

все рут получен

на терминале телефона 
запуск фрида сервера
`sudo frida-server -l 0.0.0.0`

теперь на терминале макбука - запуск фриды

`frida-ps -H 192.168.0.102`  (ip можно глянуть в настр wifi айфона на айфоне)

все - фрида подрублена!
есть контакт!
```
evgeniy@Evgeniys-MacBook-Pro ~ % frida-ps -H 192.168.0.102
 PID  Name
----  ------------------------------------------------------------
 191   PosterBoard
 252   Spotlight
 346   Календарь
1127   Купер
```
![[Снимок2026-05-2319.49.30.png]]


ищу точное название PID
```sh
evgeniy@Evgeniys-MacBook-Pro ~ % frida-ps -H 192.168.0.102
 PID  Name
----  ------------------------------------------------------------
 191   PosterBoard
 252   Spotlight
 346   Календарь
1127   Купер
```


### пытаюсь вытащить бинарник  через  `bagbak`

 `bagbak`

на мак ставлю 
`npm install -g bagbak`

приложение купер открыто на айфоне PID::1169

запуск программы в терминале мака 
```q
bagbak -H 192.168.0.102 ru.insttek.kuperapp
```

```q
evgeniy@Evgeniys-MacBook-Pro ~ % bagbak -H 192.168.0.102 ru.insttek.kuperapp
[info] Preparing app bundle...
[info] Decrypting main app...
[decrypt] KuperApp
[info] Decrypting extension ru.insttek.kuperapp.KuperAppLive...
[decrypt] PlugIns/KuperAppLiveExtension.appex/KuperAppLiveExtension
[info] Decrypting extension ru.insttek.kuperapp.MindboxNotificationServiceExtension...
[decrypt] PlugIns/MindboxNotificationServiceExtension.appex/MindboxNotificationServiceExtension
[info] Decrypting extension ru.insttek.kuperapp.MindboxNotificationContentExtension...
[decrypt] PlugIns/MindboxNotificationContentExtension.appex/MindboxNotificationContentExtension
[info] Packaging IPA...
[info] Downloading...
[springboard] info stream: /var/tmp/.bagbak-KuperApp.app/app.ipa size: 77745802
[info] Streaming 74.1 MB from device...
  ██████████████████████████████ 100% 74.1/74.1 MB
Saved to ru.insttek.kuperapp-17.0.515.ipa
evgeniy@Evgeniys-MacBook-Pro ~ %
```

получилось! 

кидаю все в папку на десктоп
```sh
mkdir ~/Desktop/kuper_analysis
cd ~/Desktop/kuper_analysis
и расспакую
unzip ~/ru.insttek.kuperapp-17.0.515.ipa
```

открываю бинарник-папку

```q
evgeniy@Evgeniys-MacBook-Pro kuper_analysis % cd /Users/evgeniy/Desktop/kuper_analysis/Payload/KuperApp.app
evgeniy@Evgeniys-MacBook-Pro KuperApp.app % ls -la
total 185912
drwxr-xr-x@ 67 evgeniy  staff      2144 23 май 20:42 .
drwxr-xr-x@  3 evgeniy  staff        96 23 май 20:42 ..
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 Adjust.bundle
-rw-r--r--@  1 evgeniy  staff     18397  5 май 20:26 AppIcon60x60@2x.png
-rw-r--r--@  1 evgeniy  staff     27043  5 май 20:26 AppIcon76x76@2x~ipad.png
-rw-r--r--@  1 evgeniy  staff   1739176  5 май 20:26 Assets.car
drwxr-xr-x@  3 evgeniy  staff        96 23 май 20:42 boost_privacy.bundle
drwxr-xr-x@  7 evgeniy  staff       224 23 май 20:42 CloudpaymentsSDK.bundle
-rw-r--r--@  1 evgeniy  staff     25702  5 май 20:26 config.ini
-rw-r--r--@  1 evgeniy  staff      2204  5 май 20:26 CPCAViewController.nib
-rw-r--r--@  1 evgeniy  staff      2170  5 май 20:26 CPCAViewControllerIPhone.nib
-rw-r--r--@  1 evgeniy  staff      2203  5 май 20:26 CPROCertviewViewController.nib
-rw-r--r--@  1 evgeniy  staff      2040  5 май 20:26 CPROCertviewViewControllerIPhone.nib
-rw-r--r--@  1 evgeniy  staff  43112595  5 май 20:26 customer.dec
drwxr-xr-x@ 11 evgeniy  staff       352 23 май 20:42 DGisModule.bundle
-rw-r--r--@  1 evgeniy  staff       784  5 май 20:26 dgissdk.key
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 en.lproj
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 FBLPromises_Privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 FirebaseCore_Privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 FirebaseCoreInternal_Privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 FirebaseInstallations_Privacy.bundle
drwxr-xr-x@  7 evgeniy  staff       224 23 май 20:42 Frameworks
drwxr-xr-x@  3 evgeniy  staff        96 23 май 20:42 glog_privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 GoogleUtilities_Privacy.bundle
-rw-r--r--@  1 evgeniy  staff      8210  5 май 20:59 Info.plist
-rw-r--r--@  1 evgeniy  staff         0  5 май 20:26 kis_1
-rwxr-xr-x@  1 evgeniy  staff  48723888 23 май 20:39 KuperApp
drwxr-xr-x@  5 evgeniy  staff       160 23 май 20:42 LaunchScreen.storyboardc
-rw-r--r--@  1 evgeniy  staff        58  5 май 20:26 license.enc
drwxr-xr-x@  8 evgeniy  staff       256 23 май 20:42 locale
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 Lottie_React_Native_Privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 LottiePrivacyInfo.bundle
-rw-r--r--@  1 evgeniy  staff   1347216  5 май 20:26 main.jsbundle
drwxr-xr-x@  5 evgeniy  staff       160 23 май 20:42 Mindbox.bundle
drwxr-xr-x@  5 evgeniy  staff       160 23 май 20:42 MindboxLogger.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 MindboxNotifications.bundle
-rw-r--r--@  1 evgeniy  staff      2083  5 май 20:26 MSCARequestViewController.nib
-rw-r--r--@  1 evgeniy  staff      2111  5 май 20:26 MSCARequestViewControllerIPhone.nib
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 nanopb_Privacy.bundle
-rw-r--r--@  1 evgeniy  staff      2083  5 май 20:26 NewUserViewController.nib
-rw-r--r--@  1 evgeniy  staff      2111  5 май 20:26 NewUserViewControllerIPhone.nib
-rw-r--r--@  1 evgeniy  staff      3136  5 май 20:26 PaneViewController.nib
-rw-r--r--@  1 evgeniy  staff      3136  5 май 20:26 PaneViewControllerIPhone.nib
-rw-r--r--@  1 evgeniy  staff         8  5 май 20:26 PkgInfo
drwxr-xr-x@  5 evgeniy  staff       160 23 май 20:42 PlugIns
-rw-r--r--@  1 evgeniy  staff      1098  5 май 20:26 PrivacyInfo.xcprivacy
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 PromiseKit_Privacy.bundle
drwxr-xr-x@  3 evgeniy  staff        96 23 май 20:42 RCT-Folly_privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 React-Core_privacy.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 React-cxxreact_privacy.bundle
-rw-r--r--@  1 evgeniy  staff      2028  5 май 20:26 RequestSelectionViewController.nib
-rw-r--r--@  1 evgeniy  staff      2028  5 май 20:26 RequestSelectionViewControllerIPhone.nib
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 RNCAsyncStorage_resources.bundle
-rw-r--r--@  1 evgeniy  staff      3787  5 май 20:26 RndmBioViewController.nib
-rw-r--r--@  1 evgeniy  staff      3783  5 май 20:26 RndmBioViewControllerIPhone.nib
drwxr-xr-x@  5 evgeniy  staff       160 23 май 20:42 RNSVGFilters.bundle
-rw-r--r--@  1 evgeniy  staff     13257  5 май 20:26 root.sto
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 ru.lproj
-rw-r--r--@  1 evgeniy  staff      1478  5 май 20:26 RussianTrustedRootCA.der
-rw-r--r--@  1 evgeniy  staff      2265  5 май 20:26 SelectionViewController.nib
-rw-r--r--@  1 evgeniy  staff      2165  5 май 20:26 SelectionViewControllerIPhone.nib
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 Sentry.bundle
-rw-r--r--@  1 evgeniy  staff     55079  5 май 20:26 spinner.json
-rw-r--r--@  1 evgeniy  staff      2071  5 май 20:26 TemplateSelectionViewController.nib
-rw-r--r--@  1 evgeniy  staff      2152  5 май 20:26 TemplateSelectionViewControllerIPhone.nib
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 UBSCryptoSDKResources.bundle
drwxr-xr-x@  4 evgeniy  staff       128 23 май 20:42 ZIPFoundation_Privacy.bundle
evgeniy@Evgeniys-MacBook-Pro KuperApp.app %
```

запуск радар2 чтобы понять - что там вообще норм все расшифрованно

```q
r2 -A ./KuperApp

соотв назваиню приложения  - значит главный бинарник
```

анализ проведен 
## теперь приступаю к поиску уязвимостей

```q
# Простой поиск строки
/ текст

# Поиск с игнорированием регистра
/i текст

# Поиск строки в широких символах (UTF-16, часто в iOS)
/w текст

# Поиск шелл-кода/байтов
/x 48 65 6c 6c 6f  # ищет байты "Hello" в hex

# Поиск строки в обратном направлении
?/ текст  # или нажмите Ctrl+Enter после / для поиска назад

# Поиск только в исполняемом коде (не в данных)
/к текст  # (code search)


# Базовое ИЛИ
/ jailbreak|cydia|frida

# Группировка
/ (jailbreak|cydia).*(detect|check)

# Начало строки
/ ^https?://

# Конец строки
/ \.app$

# Любой символ (кроме новой строки)
/ j.il.re.k

# Цифры
/ [0-9]

# Буквы
/ [A-Za-z]

# Специфические паттерны
/ https?://[a-zA-Z0-9./?=&_-]+
/ [A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}
/ [0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}  # IP адреса


# Поиск строк в секции данных (ROData, Data)
/zd текст

# Поиск с учетом размера (поиск больших строк)
/zs 100  # ищет строки длиной 100+ байт

# Поиск только в определенном диапазоне
/e 0x100000000 @ 0x100000000:0x100100000  # поиск в диапазоне

# Поиск только в текущем блоке
// текст  # поиск внутри текущего basic block

# Поиск символьных ссылок
/s символ

# Поиск определений (для декомпилированных функций)
/df функция


# Сохранить результаты в блокнот (notes)
/. jailbreak  # сохранить совпадения в блокнот

# Вывести результаты в виде таблицы
/e jailbreak  # более детальный вывод

# Искать все вхождения (не останавливаться после первого)
// jailbreak  # slash slash для продолжения поиска

# Поиск с ведением лога
/^ текст  # залогировать совпадения


# Перейти к следующему результату
/  # просто нажмите / после первого поиска

# Перейти к предыдущему результату
?/  # или Ctrl+Enter

# Показать все результаты поиска
/  # отобразит все совпадения с номерами

# Перейти к конкретному результату
/ 2  # перейти ко второму результату

# Показать результаты с контекстом
/  # и затем 'V' для визуального режима

# Очистить историю поиска
/.!


## продвинутые темки

# Поиск строк с экранированием (если в строке есть спецсимволы)
/ jailbreak\?cydia

# Поиск с использованием переменных окружения
e search.from = 0x100000000  # установить начальный адрес
e search.to = 0x100100000    # установить конечный адрес
/ jailbreak                   # поиск в заданном диапазоне

# Поиск команд в ассемблере
/a mov x0, x1  # ищет инструкции
/a bl sym.imp.something  # ищет вызовы функций

# Поиск перекрестных ссылок (xrefs)
/axt 0x100123456  # найти все ссылки на адрес

```

ну, собственно, все работает!!
все расшифрованно!!
```q
0x1016bcc5f hit4_242 .MessageerrorIdhttpStatusCodeRawVa.
0x1016c73b0 hit4_243 .httpRedirectionHandl.
0x1016c8380 hit4_244 .allowGostUrlshttpRedirectionHandl.
[0x100005140]> 0x10149e43e
[0x10149e43e]> pd
            ;-- str.http:__localhost:8969_stream:
            ;-- _185:
            ; DATA XREF from reloc.fixup.http:__localhost:8969_stream @
            0x10149e43e     .string "http://localhost:8969/stream" ; len=29
            0x10149e45b      2e             unaligned
            0x10149e45c  ~   2a002f55       invalid
            ;-- str._Users_mishin.gm_customer_app_ios_Pods_Sentry_Sources_Sentry_SentryOptions.m:
            ; XREFS: STRN 0x1006ae9ac  STRN 0x1006aea7c  STRN 0x1006aec60  STRN 0x1006aee8c  STRN 0x1006af010  STRN 0x1006af190
            0x10149e45e     .string "/Users/mishin.gm/customer-app/ios/Pods/Sentry/Sources/Sentry/SentryOptions.m" ; len=77
            ;-- str.Failed_to_initialize_SentryOptions:___:
            ; DATA XREF from aav.0x101853090 @
            0x10149e4ab     .st
            и так далее...
```
![[Снимок6026-05-2320.57.02.png]]




