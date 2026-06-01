

скачивать буду свою соц сеть [[0_MeetWay]]

```c
adb shell pm list packages --user 0

```

вижу  свое приложение   
package:com.evgeniy.meetway

теперь нужно найти путь к APK

```q
adb shell pm path com.evgeniy.meetway
```

вижу путь
`package:/data/app/~~pQz_nexOXDLHhfrDh34lkA==/com.evgeniy.meetway-q1fPFditJ4sQC0m5so7TOA==/base.apk`


теперь скачаю на копм на десктоп
```q
adb pull /data/app/~~pQz_nexOXDLHhfrDh34lkA==/com.evgeniy.meetway-q1fPFditJ4sQC0m5so7TOA==/base.apk ~/Desktop/meetway.apk
```

проверю быстро
ls -la ~/Desktop/meetway.apk
 да, скачалось!
 весит 100 мб


все, apk файл на рабочем столе!

-----------























