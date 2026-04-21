SSh_palera1n_iphone8_ios16_7_14
шаг 0
 С телефона убрать все пароли все Touch ID сделать полный сброс и запустить его при запуске не подключать никакие пароли нигде., Даже Apple ID не подключать.

Если стоял джейлбрейк какой-либо, то можно его убрать таким способом:
и поставить новый
(почему нужно именно в этой комбинации так делать, потому что, если установить sileo и debugserver, тогда будут проблемы с подключением ssh) 

1 
`palera1n --force-revert -f`
2 режим FakeFS
`palera1n -c -f` (5-10 мин)
3
`palera1n -f`

4
перв терминал
`iproxy 44444 44`
второй терминал
`ssh -p 44444 root@localhost`
стандарт пароль: alpine

и вуаля - я в системе
```q
evgeniy@Evgeniys-MacBook-Pro ~ % ssh -p 44444 root@localhost
root@localhost's password:
#
# whoami
root
#
```
но если установить sileo - то больше не получиться подключиться к ssh
пишет, что типо неверный пароль...

попробую через Filza https://tigisoftware.com/cydia/  , перейти в папку root - далее -  /private/etc/master.password
и вручную  сменить пароль!
открываю файл через тектовый редактор и вижу
там вместо пароля - восклицательный знак!

вместо знака ставлю /smx7MYTQIi2M  это хеш alpine

и от mobile тоже лучше поменять пароль на такойже


вот мой файл
```c
# User Database

#

# This file is the authoritative user database.

##

nobody:*:-2:-2::0:0:Unprivileged User:/var/empty:/usr/bin/false

root:!:0:0::0:0:System Administrator:/var/root:/usr/bin/zsh

❌❌❌🟢 воооот выше проблемка, вместо пароля стоит восклицательный знак!
нужно заменить его на это: /smx7MYTQIi2M
это хеш слова alpine , оно и пусть будет как пароль




mobile:$6$k2XAnEHBqQ1Ct2aM$IzPj/Mw0ptA.34cQ9KGGYC1DBaY/E2JTMaBGpRkWwzxV6mqIbJlDN3fWuEpItyuVjfX6CcIkKq/qE7Lo1r/ST0:501:501::0:0:Mobile User:/var/mobile:/usr/bin/zsh

daemon:*:1:1::0:0:System Services:/var/root:/usr/bin/false

_ftp:*:98:-2::0:0:FTP Daemon:/var/empty:/usr/bin/false

.... так еще куча всего ниже , неважно

```


вот так должно быть

```q
...
..
...
root:/smx7MYTQIi2M:0:0::0:0:System Administrator:/var/root:/usr/bin/zsh
mobile:/smx7MYTQIi2M:501:501::0:0:Mobile User:/var/mobile:/usr/bin/zsh
...
..
..
```

<img src="../../../assets/IMG_935A94276B74-1.jpeg" alt="Скрин" style="width: 90%; max-width: 1000px;" />




пробую снова подрубиться 
перв терминал
`iproxy 44444 44`
второй терминал
`ssh -p 44444 root@localhost`
пароль: alpine

победа!
```q
evgeniy@Evgeniys-MacBook-Pro ~ % ssh -p 44444 root@localhost
root@localhost's password:
\h:\w \u$ whoami
root
\h:\w \u$
```
-----