https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0226/

#  включен ли флаг отладки в AndroidManifest

проверяет флаг `android:debuggable` в файле `AndroidManifest.xml`
тест провожу на приложении 👉 [[0_MeetWay]]

------

Когда флаг `android:debuggable` установлен в `true`, приложение **разрешает отладку**. Это значит, что:

- Кто угодно (с физическим доступом к устройству или через ADB) может подключиться к процессу приложения отладчиком, например, через **Android Studio** или **JDB**
   
- Отладчик позволяет **смотреть** все переменные, **читать** память, **менять** значения на лету и **обходить** проверки безопасности (например, пропустить экран логина или изменить баланс)

В **продакшн-версии** этот флаг должен быть **`false`** или отсутствовать (по умолчанию `false`)

-----------


проверка через **`aapt2`**
`~/android-11/aapt2 dump badging ~/Desktop/meetway.apk | grep debuggable`

проверка через **`apkanalyzer`**
`apkanalyzer manifest print ~/Desktop/meetway.apk | grep debuggable`


результаты:

```c
evgeniy@Evgeniys-MacBook-Pro-2 ~ % ~/android-11/aapt2 dump badging ~/Desktop/meetway.apk | grep debuggable

application-debuggable


evgeniy@Evgeniys-MacBook-Pro-2 ~ % apkanalyzer manifest print ~/Desktop/meetway.apk | grep debuggable
Exception in thread "main" java.lang.IllegalStateException: Cannot locate latest build tools
	at com.android.tools.apk.analyzer.AaptInvoker.getPathToAapt(AaptInvoker.java:129)
	at com.android.tools.apk.analyzer.AaptInvoker.<init>(AaptInvoker.java:48)
	at com.android.tools.apk.analyzer.ApkAnalyzerCli.getAaptInvokerFromSdk(ApkAnalyzerCli.java:284)
	at com.android.tools.apk.analyzer.ApkAnalyzerCli.main(ApkAnalyzerCli.java:132)
evgeniy@Evgeniys-MacBook-Pro-2 ~ %
```

видно что есть application-debuggable! 
значит отладка разрешена!

в продакт не должна она попасть!

открываю андроид студию

ищу там файл androidManifest - но там нет флага application , а значит - по умолчанию должен быть false для продакт сборок (а для дебаг сборок - в тру), но тесты выше показали , что дебаг установлен в debuggable

этот же параметр можно поискать в конфигурации сборки

открываю файл  ### `build.gradle.kts` (module app)
вот нужная секция  -  но в ней нет ничего про дебаг... (а по умолчанию для дебаг сборок она ставится в tru)
```js
buildTypes {
    release {
        isMinifyEnabled = false
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
}
```

сам этот флаг ставится в AndroidManifest.xml финальной версии
и просмотреть его можно как раз с помощью декомпиляторов - aapt2 или apkanalyzer или полуавтоматически через **`apktool`**

--------

### вывод:
 тест как бы провален - так как дебаг разрешен
 но технически - это не ошибка - так как это дефаг сборка - в которой  debuggable` установлен в `true`


