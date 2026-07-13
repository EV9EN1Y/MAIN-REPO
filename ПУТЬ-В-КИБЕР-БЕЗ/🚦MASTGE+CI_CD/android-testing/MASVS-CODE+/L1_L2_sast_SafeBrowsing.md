# MASTG-TEST-0399: SafeBrowsing Disabled
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0399/

----------

**не отключена ли защита SafeBrowsing** в WebView. Если она выключена, приложение не будет предупреждать пользователя о переходе на опасные сайты (фишинг, вредоносное ПО)

SafeBrowsing - это встроенная защита, которая предупреждает, если пользователь пытается открыть опасный сайт. На Android 8.1+ она включена по умолчанию. Но разработчик может отключить её:

- В `AndroidManifest.xml` через мета-данные `android.webkit.WebView.EnableSafeBrowsing` со значением `false`
 
- В коде через `WebSettings.setSafeBrowsingEnabled(false)`


Тест проверяет, отключена ли защита. Если да - это нарушение, так как пользователь лишается важной защиты

-------------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```


-------


1)  гляну в коде  через JADX- есть ли там - `setSafeBrowsingEnabled`
- если есть - то защита скорее всего отключена!

2) нужно смотреть манифест и искать там **`AndroidManifest.xml`**

```q
<meta-data android:name="android.webkit.WebView.EnableSafeBrowsing"
           android:value="false" />
```

--------

результаты:


setSafeBrowsingEnabled - в коде не обнаружил!

смотрю манифест

нет там EnableSafeBrowsing

значит = = = защита не отключена на уровне манифеста (нет мета-данных `android.webkit.WebView.EnableSafeBrowsing` со значением `false`)

---------

## вывод

- В `AndroidManifest.xml` не найдено мета-данных для отключения SafeBrowsing.
   
- В коде не найден вызов `setSafeBrowsingEnabled(false)` (если это так после проверки в JADX)
   

**Вывод:**  
Приложение не отключает защиту SafeBrowsing. Тест **ПРОЙДЕН**


