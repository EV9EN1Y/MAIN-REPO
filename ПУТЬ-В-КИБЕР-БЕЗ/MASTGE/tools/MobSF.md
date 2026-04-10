запуск докера 

`open -a Docker`

проверка запуска 

`docker ps`

качаю образ MobSF

`docker pull opensecurity/mobile-security-framework-mobsf`


запуск 

`docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf`


открываю в браузере 
http://localhost:8000

Логин: `mobsf`  
Пароль: `mobsf`

-------

открываю новый терминал 
и архивирую бинарную папку .app

`cd ~/Desktop && zip -r MeetWay.app.zip MeetWay.app`

создался файл на рабочем столе `MeetWay.app.zip`

-----

через веб интерфейс открываю этот файл MeetWay.app.zip

но ошибка [ERROR] 09/Apr/2026 15:51:56 - This ZIP Format is not supported
он не может анализировать такие форматы 

конвертирую .app в .ipa
```
cd ~/Desktop
mkdir -p Payload
cp -r MeetWay.app Payload/
zip -r MeetWay.ipa Payload
```

теперь у меня есть файл MeetWay.ipa

загружаю его в скан в веб интерфейсе!

и начался анализ!!!