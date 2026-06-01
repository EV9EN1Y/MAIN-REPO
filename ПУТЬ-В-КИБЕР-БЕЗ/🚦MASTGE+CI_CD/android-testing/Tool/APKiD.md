## сканер APKiD - это идентификатор защиты

Это **Android Application Identifier** - инструмент, который определяет, с помощью чего сделан APK: какой компилятор, обфускатор или упаковщик использовался


APKiD использует **Yara-правила** - сигнатуры, по которым ищет известные паттерны в APK/DEX файлах . То есть у него внутри база знаний: "если вижу такой байткод - значит, использовали ProGuard; если такой - это DexGuard; если вижу шифрование - это какой-то упаковщик"

APKiD - кратко показыват, какой уровень защиты есть в приложении, если есть

короче говоря, на первом этапе разведки имеет смысл запустить этот инструмент и глянуть, че он покажет

Это узкоспециализированный сканер. **APKiD не показывает тебе сам код, он анализирует, ЧЕМ и КАК этот код защитили**


**Что он делает (по факту):**

- Определяет, каким компилятором собран код (`dx`, `r8` или `unknown`)
- Показывает, есть ли обфускация (`unreadable field names`)
- **Самое важное:** Детектирует анти-дебаг (`anti_debug`) и анти-эмуляторные проверки (`anti_vm` - это твои `Build.FINGERPRINT check`) 
- Может определить, использовались ли пакеры (типа коммерческих защит) 
-------

### установка

```q
# 1. Качаем форк yara-python от RedNaga (не из PyPI!)
git clone --recursive https://github.com/rednaga/yara-python-1 yara-python
cd yara-python

# 2. Собираем с поддержкой DEX
python setup.py build --enable-dex install

pyenv global 3.13.13
exec $SHELL
python --version
pip --version

# 3. Возвращаемся домой и ставим apkid
cd ..
pip install apkid

// гляну версию
pip show apkid

(у меня Version: 3.1.0)
отлично!
```

## запуск!
на рабочем столе у меня apk файл приложения соц сети - вот так достал его  [[скачать_APK_на-комп-с-телефона]]

запускаю  apkid
```q
apkid ~/Desktop/meetway.apk
```

вижу в выводе

++ мой разбор 
```q
evgeniy@Evgeniys-MacBook-Pro ~ % apkid ~/Desktop/meetway.apk
[+] APKiD 3.1.0 :: from RedNaga :: rednaga.io  🔶(тут инфа о приложении)
[*] /Users/evgeniy/Desktop/meetway.apk!classes10.dex 🔶(начало анализа конкрет файла)

classes10.dex - это DEX-файл (код приложения). APK - это архив. Внутри код разбит на classes.dex, classes2.dex, classes3.dex и т.д. тут их 20 штук  приложение немаленькое

 |-> compiler : unknown (please file detection issue!) 🔶(не понял что за компилятор собрал)

 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes2.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes4.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes16.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes6.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes9.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes17.dex


 |-> anti_vm : Build.FINGERPRINT check, Build.MANUFACTURER check 🔶(проверка на эмулятор есть в коде)
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes19.dex
 |-> anti_vm : Build.BOARD check, Build.FINGERPRINT check, Build.MANUFACTURER check
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes7.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes12.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes14.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes3.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes11.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes5.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes.dex
 |-> anti_vm : Build.FINGERPRINT check, Build.HARDWARE check, Build.MANUFACTURER check, Build.MODEL check, Build.PRODUCT check, possible VM check
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes20.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes18.dex
 |-> anti_vm : Build.FINGERPRINT check, Build.MANUFACTURER check, Build.TAGS check
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes13.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes8.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
[*] /Users/evgeniy/Desktop/meetway.apk!classes15.dex
 |-> compiler : unknown (please file detection issue!)
 |-> yara_issue : yara issue - dex file recognized by apkid but not yara module
evgeniy@Evgeniys-MacBook-Pro ~ %
```

все что нашлось - это просто части фреймворков!
наличие анти-эмуляторных проверок - ничего особенного...



