
сперва через радар 2 и string - нашел адрес `0x004f4544` функции - которая меня интересует 

пускаю 
```shell
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % strings MeetWay.debug.dylib | grep -iE "cydia|sileo|zebra|jailbreak|jail|break|frida|substrate|tweak|inject"

/Applications/Cydia.app
/Applications/Sileo.app
/Applications/Zebra.app
/private/var/tmp/cydia.log
/private/jailbreak_test_
cydia://
sileo://
MeetWay/JailbreakDetector.swift
JailbreakDetector  👈

blackfriday
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```

теперь найду это же через радар и найду адресс функции чтобы залочить ее

```shell
MacBook-Pro MeetWay.app % r2 -A ./MeetWay.debug.dylib

afl~JailbreakDetector  👈

```

вот это красота - которая блокирует запуск
```
[0x00004000]> afl~JailbreakDetector
0x004f4544 👈  45   3396 sym.MeetWay.JailbreakDetector.isJailbroken_...yFZ_ 👈
0x004f5288    1      4 sym.MeetWay.JailbreakDetector...VACycfC
0x004f528c    1     20 sym.MeetWay.JailbreakDetector...VMa
0x00a4e680    1      4 sym.MeetWay.JailbreakDetector...VMF
[0x00004000]>
```


```shell
MacBook-Pro MeetWay.app % r2 -A ./MeetWay.debug.dylib

afl~JailbreakDetector

```

```
вот так еще можно проверять

[0x00004000]> izz~cydia
[0x00004000]> izz~sileo
[0x00004000]> izz~jailbreak
[0x00004000]> izz~frida
[0x00004000]> izz~substrate
[0x00004000]> izz~tweak
[0x00004000]> izz~inject

```


```
[0x00004000]> afl~JailbreakDetector
0x004f4544   45   3396 sym.MeetWay.JailbreakDetector.isJailbroken_...yFZ_
0x004f5288    1      4 sym.MeetWay.JailbreakDetector...VACycfC
0x004f528c    1     20 sym.MeetWay.JailbreakDetector...VMa
0x00a4e680    1      4 sym.MeetWay.JailbreakDetector...VMF
[0x00004000]>
```

можно еще и pdf смотреть подробно всю функцию , чтобы убедиться - что это нужная функция

вот здесь почитать подробнее -> [[_R_(SAST+DAST)-Jailbreak-Detection-in-Code+Runtime(radare2-Frida-LLDB-patch-Objection)]]

--------
 далее:
 
#### запуск LLDB:
вот тут можно посмотреть первый запуск (при установке еще на телефон) - 👉 [[LLdb]] 👈

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
 не поможет - так как приложение падает сразу и pid узнать невозможно вот так : // debugserver 0.0.0.0:12345 -a 1475

поэтмоу вот так
debugserver localhost:12345 --waitfor MeetWay


вижу Listening - отлично - теперь дебаг сервер ждет запуска приложения ручками


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


теперь включаю приложеине в телефоне

---

  вижу в терминале LLDB это ниже - значит - все окей
 
evgeniy@Evgeniys-MacBook-Pro ~ % lldb
(lldb) process connect connect://localhost:12345
Process 1851 stopped
* thread #1, stop reason = signal SIGSTOP
    frame #0: 0x0000000100e3c8b0 dyld`dyld3::MachOFile::trieWalk(Diagnostics&, unsigned char const*, unsigned char const*, char const*) + 140
dyld`dyld3::MachOFile::trieWalk:
->  0x100e3c8b0 <+140>: ldrsb  w8, [x9], #0x1
    0x100e3c8b4 <+144>: str    x9, [sp, #0x10]
    0x100e3c8b8 <+148>: tbnz   w8, #0x1f, 0x100e3c8c4 ; <+160>
    0x100e3c8bc <+152>: and    x24, x8, #0xff
Target 0: (MeetWay) stopped.
(lldb)


LLDB подключился к процессу MeetWay с PID 1851, и процесс ЗАМОРОЖЕН на самой ранней стади

приложение - висит с белым экраном

------------------------

нужно найти адресс бинарника 

image list -o -f MeetWay.debug.dylib

вижу 
[  0] 0x0000000100ed0000 /private/var/containers/Bundle/Application/7ED22691-DC92-497F-BA18-4BB68280EF8F/MeetWay.app/MeetWay.debug.dylib(0x0000000100ed0000)


вот адресс 0x0000000100ed0000


----------------

гляну - где находится сама функция которую нашел через радар 

image lookup -rn "JailbreakDetector.isJailbroken"

вижу
1 match found in /private/var/containers/Bundle/Application/7ED22691-DC92-497F-BA18-4BB68280EF8F/MeetWay.app/MeetWay.debug.dylib:
        Address: MeetWay.debug.dylib[0x00000000004f4544] (MeetWay.debug.dylib.__TEXT.__text + 5178692)
        Summary: MeetWay.debug.dylib`static MeetWay.JailbreakDetector.isJailbroken() -> Swift.Bool at JailbreakDetector.swift:13

----------

теперь нужно поставить брейк на этот адресс (со смещением)

br set -a 0x0000000100ed0000+0x004f4544

вижу - что брейк установился
Breakpoint 1: where = MeetWay.debug.dylib`static JailbreakDetector.isJailbroken() at JailbreakDetector.swift:13, address = 0x00000001013c4544

------------

далее продолжу выполнение программы для брейка

c

вижу место - где сраболал брейк!
это функция в структуре! идеально, удалось четко попасть в ту самую функцию!


(lldb) c
Process 1851 resuming
Process 1851 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001013c4544 MeetWay.debug.dylib`static JailbreakDetector.isJailbroken() at JailbreakDetector.swift:13
   10
   11  	struct JailbreakDetector {
   12
-> 13  	    static func isJailbroken() -> Bool {
   14
   15
   16  	        let jailbreakPaths = [
Target 0: (MeetWay) stopped.

--------

теперь посмотрю первые штук 5 инструкций этой функции

dis -f -c 5

вижу

MeetWay.debug.dylib`static JailbreakDetector.isJailbroken():
->  0x1013c4544 <+0>:  stp    x22, x21, [sp, #-0x30]!
    0x1013c4548 <+4>:  stp    x20, x19, [sp, #0x10]
    0x1013c454c <+8>:  stp    x29, x30, [sp, #0x20]
    0x1013c4550 <+12>: add    x29, sp, #0x20
    0x1013c4554 <+16>: sub    sp, sp, #0x430

------------

теперь можно сделать так - чтобы не выполнять то - что внутри функции и при этом - вернуть значение - напрмиер false (так как функция называется isJailbroken - логично - что если вернет функция false - типо не нашла джейл)

итак - по порядку - эти команды

register write x0 0
br delete 1
thread return 0

разбор команд
register write x0 0 - Записывает значение 0 в регистр x0 - - В ARM64 (процессор iPhone) регистр x0 используется для возврата значений из функций -
В Swift Bool представляется как 0 = false, 1 = true
то есть - Записывая 0 в x0, мы "кладём" туда значение false


после этих трех команд - вижу

(lldb) register write x0 0
(lldb) br delete 1
1 breakpoints deleted; 0 breakpoint locations disabled.
(lldb) thread return 0
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010175d784 MeetWay.debug.dylib`AppDelegate.application(application=0x0000000602808340, launchOptions=nil) at AAIVAApp.swift:59:30
   56
   57  	        // ---------------------
   58
-> 59  	        if JailbreakDetector.isJailbroken() {
   60  	            // Показываем алерт СИНХРОННО (без DispatchQueue)
   61  	            let alert = UIAlertController(
   62  	                title: "⚠️ Внимание, епта!",
(lldb)


то есть - я выше вернул функции значение false -  и следующий шаг - это проверка в if JailbreakDetector.isJailbroken() - и так как я выше - дал значение функции false - то условие это вообще не будет выполняться!

---------
поэтому - просто продолжаю выполнение!

c

---------------

и ура!!!!!!!!!!! епта!!!!!!!! получилось обойти защиту!!!!!!!

приложение запустилось!!!

```

приложение запустилось!!!
я обошел защиту!!! 
господи - спасибо
<img src="../../../assets/Снимо2026-04-2910.55.59.png" alt="Скрин" style="width: 99%; max-width: 1000px;" />
