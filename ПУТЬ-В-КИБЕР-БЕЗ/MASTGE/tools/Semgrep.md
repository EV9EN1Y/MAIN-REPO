
**Semgrep** (Semantic Grep) - это быстрый, open-source инструмент для статического анализа кода (когда есть исходный  код), который ищет уязвимости, баги и антипаттерны прямо в исходниках

Semgrep поддерживает 
- **GA (стабильно):** Swift, C, C++, Go, Java, JavaScript, Python, Ruby, Rust, TypeScript, Kotlin, PHP и др
- **Beta/Experimental:** Objective‑C, Dart, Bash, Dockerfile

-------

Semgrep встраивается **на ранних этапах SDLC** (Software Development Life Cycle) - прямо в процесс разработки
в том числе - в CI/CD пайплайнах (GitHub Actions, GitLab CI, Jenkins)
   - автоматический скан каждого PR на наличие security-флагов




```c

// просто semgrep - мощнее
//Semgrep CE (macOS)
brew install semgrep

//клонирую iOS-правила
git clone https://github.com/akabe1/akabe1-semgrep-rules.git

//запуск сканирование исходников (если есть доступ)
// 👉🍺 запускает статический анализ кода, проверяя Swift-файлы по правилам из папки akabe1-semgrep-rules/swift/
semgrep scan --config akabe1-semgrep-rules/swift/ /путь/к/проекту/

semgrep scan --config ~/Desktop/akabe1-semgrep-rules/ios /Users/evgeniy/Desktop/крайняя\ версия\ meetWay\ по\ телеграмму/IVAARO1\ 2

//(так-как отчет не маленький)
//можно сохранить в разных форматах сразу в файлик 
//  JSON формат
--json --output ~/Desktop/semgrep_results.json

// SARIF формат
--sarif --output ~/Desktop/semgrep_results.sarif

// текстовый
--output ~/Desktop/semgrep_results.txt

// на выходе список найденных проблем с указанием файла, строки и типа уязвимости

---------------------------

// mobsfscan - это надстройка над semgrep, имеющая спец правила конкретно для IOS, типо - это проще, заточеен конкретно под правила IOS
///или с мобсфсканом для удобства
pip install mobsfscan
mobsfscan --type ios /путь/к/исходникам/
```


-------

есть различные готовые правила - для поиска нужных паттернов
правила можно найти здесь: [https://github.com/akabe1/akabe1-semgrep-rules](https://github.com/akabe1/akabe1-semgrep-rules)

##### че умеет:

_1_ Biometric Authentication – ищет небезопасную биометрию (MASVS-AUTH)

_2-4_ Certificate Pinning – ищет ошибки в AFNetworking, Alamofire, Trustkit (MASVS-NETWORK)

_5_ XXE – ищет XML external entities (MASVS-CODE)

_6_ SQL Injection – ищет небезопасные SQL-запросы (MASVS-CODE)

_7_ Hardcoded Secrets – ищет пароли и токены прямо в коде (MASVS-STORAGE)

_9_ NoSQL Injection – ищет NoSQL-инъекции (MASVS-CODE)

_10-11_ WebView – ищет небезопасный WKWebview и устаревший UIWebview (MASVS-PLATFORM)

_12-16_ Insecure Storage – ищет хранение данных в открытом виде и слабые protection classes (MASVS-STORAGE)

_17-18_ Keychain – ищет экспортируемые и слабые настройки Keychain (MASVS-STORAGE)

_20-24_ Broken Cryptography – ищет небезопасную криптографию (CommonCrypto, CryptoSwift и другие) (MASVS-CRYPTO)


------------


## запускаю  semgrep scan на приложении MeetWay

на вот этом приложении  👉 [[0_MeetWay]] (рабочая соц сеть)

```
semgrep scan --config ~/Desktop/akabe1-semgrep-rules/ios /Users/evgeniy/Desktop/крайняя\ версия\ meetWay\ по\ телеграмму/IVAARO1\ 2
```

результаты сканирования
```c
Scanning 7187 files (only git-tracked) with 24 Code rules:

  CODE RULES
  Scanning 260 files with 24 swift rules
┌───────────────────┐
│ 336 Code Findings │
└───────────────────┘
┌──────────────┐
│ Scan Summary │
└──────────────┘
✅ Scan completed successfully.
 • Findings: 336 (336 blocking)
 • Rules run: 24
 • Targets scanned: 260
 • Parsed lines: ~99.9%
 • Scan skipped:
   ◦ Files larger than  files 1.0 MB: 13
   ◦ Files matching .semgrepignore patterns: 401
 • For a detailed list of skipped files and lines, run semgrep with the --verbose flag
Ran 24 rules on 260 files: 336 findings.
```

нашел оч много всего...
для сравнения, MobSF нашел порядка 15 предупреждений!

каждая находка выглядит вот так
(адрес файла, ❯❱ обьясняется сработавшее правило, и потом уже номера строк, где это было обнаружено)
![[Снимок2026-04-2416.37.31.png]]
```c
    ❯❱ akabe1-semgrep-rules.ios.swift.storage.hardcoded_secret
          This iOS mobile application seems containing some hardcoded information, this
          storage mode is insecure because does not guarantee the confidentiality of data.
          An attacker could be able to retrieve the hardcoded data from the code of iOS
          mobile application.  When saving reserved data into the device, it is recommended
          to adopt any of the encryption methods/tools internationally recognized as strong
          for iOS (adapt to the specific mobile application context).

          495┆ let accessKey = "УКСHFHJESl8Vc6eod-HByugYFuJh"
            ⋮┆----------------------------------------
          496┆ let secretKey = "llУКHGJDJRYFjgXIJupQenfJGUYFufUYfuyij17ep2"

    /Users/evgeniy/Desktop/крайняя версия meetWay по телеграмму/IVAARO1
  2/IVAARO/View/authorization/Registration.swift
```


	интересные находки: но и оч много ложных срабатываний!
```c
kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock

UserDefaults.standard.set(updatedAlerts, forKey: "key_1_enter")

UserDefaults.standard.set(apnsToken, forKey: "apnsDeviceToken")
   
UserDefaults.standard.set(fcmToken, forKey: "fcmToken")

private let notificationsKey = "processedNotifications_v2"

let accessKey = "УКСHFHJESl8Vc6eod-HByugYFuJh"
let secretKey = "УКHGJDJRYFjgXIJupQenfJGUYFufUYfuyij17ep2"

let kTokenKey = "token"
private let kIDTokenKey = "idToken"

self.safariViewController = SFSafariViewController(url: url)

```

### разбор найденных кейсов

(конечно, каждый найденный кейс - нужно, в коде найти и понять, что это конкретно и понять, насколько это важно)

🔴 **КРИТИЧЕСКАЯ УЯЗВИМОСТЬ**  
**Находка:** Жёстко зашитые `accessKey` и `secretKey` в коде (`AAIVAApp.swift`, строки 495-496)  ЭТО ЖЕСТОЧАЙШЕ ЗАХАРКОЖЕННЫЕ КЛЮЧИ ОТ ЯНДЕКСА СЕРВЕРА ПРЯМО В КОДЕ...
**Почему опасно?** Это прямой доступ к внешнему API . Любой, кто получит бинарник приложения, через 5 минут скормит его `strings` или откроет в `radare2` и вытащит ключи. Злоумышленник сможет накручивать статистику, воровать данные или тратить ваш бюджет на облачные сервисы^ получить полный доступ к бекенд - сервису.  
**Че делать?** Никаких ключей в коде. Хранить в Keychain с флагом `ThisDeviceOnly` или, что правильнее, гонять все запросы через свой бэкенд, где ключи лежат на сервере

---

🟠 **ВЫСОКАЯ УЯЗВИМОСТЬ**  
**Находка:** Настройка Keychain без суффикса `ThisDeviceOnly` (`KeychainManager.swift`, строки 172, 222)
**Почему опасно?** Данные из Keychain станут частью резервной копии iCloud или iTunes. Если пользователь сделает бэкап на компьютер, любой, кто получит доступ к этому бэкапу (например, при утере ноутбука), сможет достать хранящиеся токены, пароли и другую чувствительную информацию
**Что делать?** Всегда добавлять `ThisDeviceOnly`. Например, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`. Это запретит миграцию ключей на другие устройства

---

🟠 **СРЕДНЯЯ УЯЗВИМОСТЬ**  
**Находка:** Использование `SFSafariViewController` для открытия ссылок (`YandexLoginSDK.swift`, строка 306; `AuthURLPresenter.swift`, строка 61) 
**Почему опасно?** Этот браузер шарит куки и данные автозаполнения с основным приложением Safari. Можно перехватить сессионную куку авторизации через атаку XSS в открытой ссылке или подменить страницу входа, чтобы украсть пароль!!!! нужно конечно же проверять..
**Что делать?** Использовать `WKWebView` с изолированным хранилищем (без шаринга с Safari) и отключённым JavaScript, если он не нужен

---

🟡 **СРЕДНЕ-НИЗКАЯ УЯЗВИМОСТЬ**  
**находка:** Хранение `apnsToken` и `fcmToken` в `UserDefaults` (`AAIVAApp.swift`, строки 163, 188)
**опасно ли?** Сам по себе токен пуш-уведомлений - не секрет, но это уникальный идентификатор устройства. В связке с другими уязвимостями может помочь деанонимизировать пользователя или отправлять ему поддельные пуши, нужно проверять конкретно!
**Что делать?** Пуши-токены - тоже данные, которые лучше хранить в Keychain. Это правильная практика защиты от утечек через бэкапы логов

---

🔴 **ВАЖНО ПОНИМАТЬ**

## У меня в коде есть жестко защитая обфусцированная соль, и ее Semgper не обнаружил!

> Но вот утилитка - strings - нашла ее без проблем по паттерну - рандомных комбинаций в бинарнике (строки с высокой «случайностью» (энтропией))

**Вывод:** Статический анализ исходников (Semgrep/mobsfscan) - это первый этап, но он не заменяет реверс-инжиниринг бинарника (Radare2, strings, Ghidra, Frida). Если секрет обфусцирован или размазан по коду хитрым способом, Semgrep его не увидит. А в бинарнике он всё равно рано или поздно проявит себя как читаемая строка или последовательность инструкций.

Никогда не доверяй полностью ни одному инструменту. Только комбинация методов даёт полную картину!

##### например так: через strings - нашел тоже захардкоженные секреты в коде

это результат другого теста
который вот тут можно прочитать: 👉  [[L2_L1_Insufficient_Key_Sizes]]


Скрипт, который ищет строки с высокой «случайностью» (энтропией)
```Shell
strings MeetWay.debug.dylib | while read line; do
    entropy=$(echo -n "$line" | python3 -c "import sys, math; s=sys.stdin.read(); print(-sum((s.count(c)/len(s))*math.log2(s.count(c)/len(s)) for c in set(s)))")
    if (( $(echo "$entropy > 4.5" | bc -l) )); then
        echo "HIGH ENTROPY: $line"
    fi
done
```


```c
HIGH ENTROPY: $s7MeetWay19postAchivementsDataV10CodingKeys33_30F118A04066D921D14E82012D529912LLO

HIGH ENTROPY: $s7MeetWay19ResourceBundleClass33_94BB64A8071F57B6B2C38F32C62C60E2LLC

HIGH ENTROPY: https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg

HIGH ENTROPY: vkewjrbg@#$%$**)(_&^%4257(&^&^$%&^**_ch35ha56ht_encryp56ti6534yon_safhd457_202346hy5_4h4v3_ertb356uiu563h__

HIGH ENTROPY: v16@?0@"UIGraphicsImageRendererContext"8

HIGH ENTROPY: /IVAARO1 2/IVAARO/MANAGER_S/VideoCompanents.swift

HIGH ENTROPY: v40@0:8@16^{opaqueCMSampleBuffer=}24@32

HIGH ENTROPY: v24@?0@"FIRQuerySnapshot"8@"NSError"16

HIGH ENTROPY: arn:aws:sns::b1ggdrk2tndoa2t1f6jf:app/APNS_SANDBOX/testSandboxPush

HIGH ENTROPY: v24@?0@"<NSItemProviderReading>"8@"NSError"16

HIGH ENTROPY: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789

HIGH ENTROPY: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789

HIGH ENTROPY: YFEYGfF2XrewgfBJV^F&ruf5xjaKtGj17ep2

HIGH ENTROPY: _TtC7MeetWayP33_94BB64A8071F57B6B2C38F32C62C60E219ResourceBundleClass
```

