### скачиваю архив apksigner
```q
wget https://dl.google.com/android/repository/build-tools_r30.0.1-macosx.zip
```

### расспаковка
```q
unzip build-tools_r30.0.1-macosx.zip
```

### запуск проверки подписи!!
```sh
java -jar ~/android-11/lib/apksigner.jar verify --verbose ~/Desktop/meetway.apk
```

## результат проверки

```q
evgeniy@Evgeniys-MacBook-Pro-2 ~ % java -jar ~/android-11/lib/apksigner.jar verify --verbose ~/Desktop/meetway.apk

Verifies
Verified using v1 scheme (JAR signing): false
Verified using v2 scheme (APK Signature Scheme v2): true
Verified using v3 scheme (APK Signature Scheme v3): false
Verified using v4 scheme (APK Signature Scheme v4): false
Verified for SourceStamp: false
Number of signers: 1


```

все работает