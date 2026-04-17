установка_стороннего_IPA_с_джейлбреком и установка FRIDA

айфон уже с джейл брейком: и уже стоит Sileo
открываю Sileo и скачиваю **Filza** (позволяет видеть все внутр файлы телефона и устанавливать левые программы)

-------

айфон 8 уже с джейлбрейком 👉
[[✅1️⃣-джейл-брейк_Palera1n_iPhone8_ios16.7.14_macOS]]

### задача - поставить на айфон левое приложение DVIA-v2 для обучения по тестам  MASTGE, и установить Frida на айфон

--------

### 🟣 нужно скачать файл приложения

через сафари качаю в файлы, бинарник приложения

https://github.com/prateek147/DVIA-v2/releases/download/v2.0/DVIA-v2-swift.ipa

![[DVIA-v2-swift.ipa]]

скачал , приложение лежит  в файлах айфона

-------

### 🟣 нужно скачать  filza
https://tigisoftware.com/cydia/     тут качаю filza через sileo
устанавливаю его там же в sileo
filza установлена - ярлык появился

-------

### 🟣 нужно скачать Frida

https://github.com/frida/frida/releases/download/16.4.3/frida_16.4.3_iphoneos-arm.deb  это фриду качал на макбук и перекинул на айфон в файлы

![[frida_16.4.3_iphoneos-arm.deb]]

через sileo нашел файл  фриды и установил его 
и на выходе вижу

(успешно установилась фрида)
```c
скачал и нажал установить 
iPhone#
dpkg -i "/var/mobile/Containers/Shared/App
D715ECC4A15B/File Provider Storage/frida_16.4.3_ip honeos-arm.deb" ;
Selecting previously unselected package re.frida.s erver.
(Reading database ... 6103 files and directories c urrently installed.)
Preparing to unpack •../frida_16.4.3_iphoneos-arm. deb
•••
Unpacking re.frida.server (16.4.3)...
Setting up re.frida.server (16.4.3)
```

и тут же - действия - перезапуск! (Respring)

телефон погас - и запустился

--------

так как был перезапуск -то желательно перезапустить джейлбрейк
`palera1n -f`

перезапустил джейл-брейк

----

### 🟣 нужно скачать AppSync Unified
(это чтобы была возможность устанавливать любые приложения на телефон , в том числе и DVIA)

поэтому нужно через sileo скачать это - AppSync Unified 
вот здесь:
https://lukezgd.github.io/repo

и тут же  в sileo - установил AppSync Unified
в конце установки - есть кнопка - перезапуск телефона автоматом
(но в телефон еще и завис наполовину)

теперь , после включения телефона, нужно снова вернуть джейлбрейк
palera1n -f

джейлбрейк установлен!

----
### 🟣 теперь можно установить  DVIA

 иду в filza и найдя файл .ipa приложения - пробую установить DVIA

и, вуаля!!!!!  DVIA-v2 установлен!!!
появился ярлык на обоях!

-----

итого:

джейл брейк - установлен так: palera1n -f
скачан Sileo внутри palera1n 
установил Filza  через Sileo
фрида установлена (скачал файл на комп и перекинул на айфон)
установил AppSync Unified через Sileo
и в конце установил еще и DVIA-v2, который скачал по ссылке в сафари

-------





