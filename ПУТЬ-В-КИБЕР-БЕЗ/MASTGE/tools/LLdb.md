
### 1️⃣ Что такое LLDB?



**LLDB** - это стандартная тема которую каждый iOS разработчик использует внутри Xcode, когда запускает приложение через него. А также тогда, когда разработчик может в коде ставить breakpoint, и когда приложение доходят на этой точке, то в этом отладчике как раз таки и появляется стек Trace вызовов., но если у нас есть Jailbreak устройства, тогда я могу провести практически ту же самую манипуляцию но вообще с любым приложением, что Apple как бы запрещает, но злоумышленники и этим пользуются.


-------

**LLDB и Frida - // Frida удобнее для скриптов (JS), LLDB мощнее для низкоуровневой отладки (регистры, память). Но оба могут делать одно и то же: подменять проверки, менять значения и взламывать "защиту", если она реализована криво**

----

**LLDB**  - отладчик (debugger) от Apple для macOS и iOS. Позволяет:

- Останавливать выполнение приложения в любой момент
   
- Смотреть и менять значения переменных и регистров
   
- Исследовать стек вызовов
   
- Обходить защиту типа `ptrace(PT_DENY_ATTACH)`

- **Поиск секретов:** Посмотреть, что лежит в переменных (токены, ключи)

- Смотреть, почему упало приложение (посмотреть стек вызовов)
    
- Изучать, как работает сторонний код (библиотеки)
- 
- **Reverse Engineering:** Понять, какие функции вызываются при нажатии кнопок


> **LLDB по умолчанию работает ТОЛЬКО с приложениями, которые ты сам собрал в Xcode под своим сертификатом разработчика**

**На неджейлбрейкнутом устройстве:**

- Ты НЕ можешь подключиться к чужому приложению из App Store
   
- Apple запрещает отладку чужих процессов на системном уровне
   
- Получаешь ошибку: `attach failed: not allowed to attach to process`
   

**На джейлбрейкнутом устройстве (твой случай):**

- Ограничения сняты, можно отлаживать ЛЮБОЕ приложение
   
- Но приложение может поставить дополнительную защиту (ptrace) — именно это ты и тестируешь


### 2️⃣ Основные команды LLDB

|Команда|Сокращение|Что делает|

|`process attach --pid <PID>`|`attach <PID>`|Подключиться к запущенному процессу|

|`process attach --name <AppName>`|`attach -n <AppName>`|Подключиться по имени приложения|

|`breakpoint set --name <func>`|`b <func>`|Поставить точку остановки на функции|

|`continue`|`c`|Продолжить выполнение после остановки|

|`thread backtrace`|`bt`|Показать стек вызовов|

|`register read`|`re r`|Прочитать все регистры|

|`register write <reg> <value>`|`re w`|Записать значение в регистр|

|`expression -- <code>`|`e <code>`|Выполнить код в контексте приложения|

|`quit`|`q`|Выйти из LLDB|

--------------



### 3️⃣ Как подключиться к приложению первый раз

в конце этого файла - есть краткая инструкция по подключению
***быстрый запуск LLDB***



**LLDB на Mac не может напрямую подключиться к процессу на iPhone.** Это разные машины. Нужен посредник - **debugserver** на самом iPhone

у меня айфон 8 ios 16.7.14

нужен  rootful режим
`palera1n --force-revert -f`
`palera1n -c -f`
`palera1n -f`

ставлю Sileo
потом
качаю ресурс https://apt.procurs.us/
**Sileo** → поиск → `debugserver-16` → установить (16й - так как ios у меня 16)

debugserver - который я только что установил через sileo по умолчанию не может подключаться к любому приложению на телефоне, поэтому:
ему нужен файл **entitlements** - который даст для debugserver право подключаться к любому процессу!!! 
  

подрубаю ssh
(подробнее про это вот тут 👉- >> [[SSh_palera1n_iphone8_ios16_7_14]])

перв терминал
`iproxy 44444 44`
второй терминал
`ssh -p 44444 root@localhost`


после подключения
ищу свое нужное мне приложение `ps aux | grep DVIA-v2`
```bash
evgeniy@Evgeniys-MacBook-Pro ~ % ssh -p 44444 root@localhost
root@localhost's password:
\h:\w \u$ ps aux | grep DVIA-v2
mobile             310   0.0  0.0 449455968     16   ??  Ss    4:13PM   0:00.00 /var/containers/Bundle/Application/A52FC348-2E73-453A-8EA6-FF19A3448BB0/DVIA-v2.app/DVIA-v2
root               393   0.0  0.0        0      0 s000  ?+    4:21PM   0:00.00 grep DVIA-v2
\h:\w \u$
```



убедимся, что дебаг сервер стоит на телефоне и виден (это через ssh телефона)
`debugserver --version`


(если первый раз - то )
теперь нужно закодировать дебаг сервер (через ssh телефона )
`base64 /usr/libexec/debugserver > /tmp/debugserver.b64`

(можно проверить, создался ли файл `ls -la /tmp/debugserver.b64`)

теперь нужно сделать файл **entitlements** для приложения
на маке создаю файл 
```xml
cat > ~/entitlements.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>get-task-allow</key>
    <true/>
    <key>task_for_pid-allow</key>
    <true/>
</dict>
</plist>
EOF
```



теперь качаю на мак файлы debugserver с телефона и декодирую их (локально в терминале мака)
`ssh -p 44444 root@localhost "cat /tmp/debugserver.b64" | base64 -D > ~/debugserver`

(проверю, перенесся ли файл `ls -la ~/debugserver`)


теперь переподписываю дебаг сервер тем созданным сертификатом
`codesign -s - --entitlements ~/entitlements.xml -f ~/debugserver`

отправляю файл обратно на телефон
`cat ~/debugserver | base64 | ssh -p 44444 root@localhost "base64 -d > /tmp/debugserver"`

теперь в терминале ssh для телефона
заменяю файл, декодирую и запускаю дебаг сервер
```c
cp /tmp/debugserver /usr/libexec/debugserver
chmod 755 /usr/libexec/debugserver
```

запускаю приложение целевое (в моем случае - DVIA-v2)
и запоминаю его PID (это через shh)
`ps aux | grep DVIA-v2 | grep -v grep`

запуск дебаг-сервера lldb к нужному приложению по его PID
(это через shh)
`debugserver 0.0.0.0:12345 -a 468`

вот так выглядит успех
```c
\h:\w \u$ debugserver *:12345 -a 310
debugserver-@(#)PROGRAM:LLDB  PROJECT:lldb-1403.2.3.13
 for arm64.
Attaching to process 310...
Listening to port 12345 for a connection from *...
```

теперь на маке в отдельном терминале подрубаюсь к lldb

`iproxy 12345 12345`
и в еще одном терминале
```q
lldb
process connect connect://localhost:12345
```

итого - 4 терминала

терминал с iproxy для ssh
терминал с shh айфона
терминал с iproxy для дебагсервера lldb
терминал c LLdb

<img src="../../../assets/Снимок2026-04-2117.02.01.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



в самом терминале lldb
после подключения вижу это - значит - успешно все получилось подключить!
```c
(lldb) process connect connect://localhost:12345
Process 468 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x00000001d0099030 libsystem_kernel.dylib`mach_msg2_trap + 8
libsystem_kernel.dylib`mach_msg2_trap:
->  0x1d0099030 <+8>: ret

libsystem_kernel.dylib`macx_swapon:
    0x1d0099034 <+0>: mov    x16, #-0x30 ; =-48
    0x1d0099038 <+4>: svc    #0x80
    0x1d009903c <+8>: ret
Target 0: (DVIA-v2) stopped.
(lldb)
```

теперь можно работать с lldb
наприер, проверить 
`b ptrace`

видно, что брекпоинт есть
<img src="../../../assets/Снимок2026-04-2117.07.33.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



теперь можно узнать, вызывает ли приложение его
`c`
но защита не сработала
```q
(lldb) c
Process 468 resuming
(lldb)
```

теперь в приложении DVIA-v2 во вкладке -анти-дебагинг
включаю disable debuggind
и тут же вижу в терминале, как приложение поймало процесс

сработала защита приложения!! и она сработала!! и я поймал места в коде - где используется эта защита!!!!!

```q
Process 468 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001d00a7448 libsystem_kernel.dylib`__ptrace
libsystem_kernel.dylib`__ptrace:
->  0x1d00a7448 <+0>:  adrp   x9, 128789
    0x1d00a744c <+4>:  add    x9, x9, #0x9c0 ; errno
    0x1d00a7450 <+8>:  str    wzr, [x9]
    0x1d00a7454 <+12>: mov    x16, #0x1a ; =26
Target 0: (DVIA-v2) stopped.
```

теперь нужно посмотреть, че там находится в регистре x0
```q
(lldb) register read x0
      x0 = 0x000000000000001f
```
(это классическая защита apple)
теперь вот так можно обойти защиту
`register write x0 0`
потом
`c`
защита обойдена!!
приложение сделало то - что я ему сказал, подменив значение в памяти!

ответ
```c
(lldb) c
Process 468 resuming
(lldb)
```
чтобы убедиться -  смотрю поток
сперва остановлю процесс `process interrupt`
`thread list`

и вуаля - я виже процессы-потоки, и вообще, lldb работает без проблем!!!!!
И ПРИ ЭТОМ - ПРИЛОЖЕНИЕ ПИШЕТ, ЧТО ЗАЩИТА ВКЛЮЧЕНА ))
НО ПО ФАКТУ - Я ОТКЛЮЧИЛ ЕЕ!

<img src="../../../assets/Снимок2026-04-2117.24.15.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




--------


###  Как обойти `ptrace(PT_DENY_ATTACH)` через LLDB


```bash
# 1. Подключиться к процессу ДО того, как сработает ptrace
lldb -n DVIA-v2
# 2. Поставить брейк на ptrace
(lldb) b ptrace
# 3. Продолжить выполнение
(lldb) c
# 4. Когда сработает брейк, посмотреть регистр x0 (содержит PT_DENY_ATTACH=31)
(lldb) re r x0
# 5. Изменить значение на 0 (PTRACE_TRACEME или безвредный)
(lldb) re w x0 0
# 6. Продолжить выполнение — защита обойдена!
(lldb) c
```

------
### Полезные сниппеты

```bash
# Запустить LLDB и сразу поставить брейк на ptrace
lldb -n DVIA-v2 -o "b ptrace" -o "c"
# Посмотреть, какие библиотеки загружены в процесс
(lldb) image list
# Посмотреть ассемблерный код функции
(lldb) disassemble --name ptrace
# Установить брейк на системный вызов (обход через syscall)
(lldb) b syscall
(lldb) re w x0 0  # когда сработает с аргументом 26 (ptrace)

```


> **LLDB — это стандартный отладчик Apple. 


-------------




------------

# быстрый запуск LLDB

#### запуск LLDB:
вот тут можно посмотреть первый запуск - [[LLdb]] 
```c
-----------
отдельный терминал для фриды
узнать бандл id
frida-ps -Uai
MeetWay         AIVARO22-2025-1.0



--------
перв терминал прокси для ssh айфона
iproxy 44444 44

-------

второй терминал ssh айфона
ssh -p 44444 root@localhost



запуск дебаг-сервера lldb к нужному приложению по его PID
debugserver 0.0.0.0:12345 -a 1475


вижу Listening - отлично


----------


третий терминал  - прокси для дебагсервера
прокси для дебаг сервер
iproxy 12345 12345

вижу - waiting for connection - = окей

-----------

четвертый терминал - запуск lldb

lldb

process connect connect://localhost:12345
---------

 вижу  - значит - все окей
 
 Process 1475 stopped
* thread #1, stop reason = signal SIGSTOP
    frame #0: 0x0000000100698914 dyld`dyld3::MachOFile::trieWalk(Diagnostics&, unsigned char const*, unsigned char const*, char const*) + 240
dyld`dyld3::MachOFile::trieWalk:
->  0x100698914 <+240>: ldrb   w13, [x8]
    0x100698918 <+244>: cbz    w13, 0x100698988 ; <+356>
    0x10069891c <+248>: mov    w11, #0x0 ; =0
    0x100698920 <+252>: add    x10, x8, #0x1
Target 0: (MeetWay) stopped.

```

теперь можно работать с LLDB


Target 0: (MeetWay) stopped - приложение остановлено на самом старте!
