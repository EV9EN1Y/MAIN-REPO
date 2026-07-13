# MASTG-TEST-0369: Insufficient Obfuscation of Security-Relevant Native Code
https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0369/

-----

Тест проверяет, использует ли приложение **нативный код (C/C++ через JNI)** для выполнения security-логики, и если да - достаточно ли он обфусцирован

Это дополнение к MASTG-TEST-0368 (который проверяет Java/Kotlin код). Если приложение хранит важную логику в нативных библиотеках (шифрование, проверки безопасности, работа с ключами), эти .so файлы тоже должны быть защищены:

- Символы должны быть удалены (stripped)
- Строки не должны лежать открытым текстом
- Имена функций не должны выдавать назначение
- Обычный `readelf` / `strings` не должен позволить понять, что делает библиотека

Если приложение не использует нативный код - тест **неприменим (N/A)** или **автоматически пройден**

-----

**Тест ПРОВАЛЕН**, если:

- В APK есть кастомные .so библиотеки с security-логикой
- Имена функций вида `checkRoot()`, `verifySignature()`, `decryptKey()`
- Строки в .so читаются через `strings` (URL, ключи, сообщения)
- Debug-символы не удалены
- Есть JNI-методы, ведущие к security-логике

**Тест ПРОЙДЕН**, если:

- Кастомных .so библиотек нет
- Или есть, но: символы удалены, строки зашифрованы, имена обфусцированы

-------

## запускаю анализ

###  Смотрю, есть ли в APK кастомные .so файлы

```bash
unzip -l meetway_with_protection.apk | grep "\.so"
```

результат

```
lib/arm64-v8a/libandroidx.graphics.path.so          <- AndroidX Graphics
lib/arm64-v8a/libdatastore_shared_counter.so         <- Jetpack DataStore
lib/arm64-v8a/libimage_processing_util_jni.so        <- CameraX Image
lib/arm64-v8a/libsurface_util_jni.so                 <- CameraX Surface
lib/armeabi-v7a/libandroidx.graphics.path.so
lib/armeabi-v7a/libdatastore_shared_counter.so
lib/armeabi-v7a/libimage_processing_util_jni.so
lib/armeabi-v7a/libsurface_util_jni.so
lib/x86/libandroidx.graphics.path.so
lib/x86/libdatastore_shared_counter.so
lib/x86/libimage_processing_util_jni.so
lib/x86/libsurface_util_jni.so
lib/x86_64/libandroidx.graphics.path.so
lib/x86_64/libdatastore_shared_counter.so
lib/x86_64/libimage_processing_util_jni.so
lib/x86_64/libsurface_util_jni.so
```

**Наблюдение:** Все 4 библиотеки - **стандартные AndroidX / Google библиотеки**. Ни одной кастомной (`libmeetway.so`, `libnative-lib.so` и т.д.)

###  проверяю JNI-методы в .so

```bash
strings lib/arm64-v8a/*.so | grep "^Java_"
```

**libdatastore_shared_counter.so:**
```
Java_androidx_datastore_core_NativeSharedCounter_nativeTruncateFile
Java_androidx_datastore_core_NativeSharedCounter_nativeCreateSharedCounter
Java_androidx_datastore_core_NativeSharedCounter_nativeGetCounterValue
Java_androidx_datastore_core_NativeSharedCounter_nativeIncrementAndGetCounterValue
```

**libimage_processing_util_jni.so:**
```
Java_androidx_camera_core_ImageProcessingUtil_nativeCopyBetweenByteBufferAndBitmap
Java_androidx_camera_core_ImageProcessingUtil_nativeShiftPixel
Java_androidx_camera_core_ImageProcessingUtil_nativeWriteJpegToSurface
Java_androidx_camera_core_ImageProcessingUtil_nativeConvertAndroid420ToABGR
Java_androidx_camera_core_ImageProcessingUtil_nativeConvertAndroid420ToBitmap
```

**libsurface_util_jni.so:**
```
Java_androidx_camera_core_impl_utils_SurfaceUtil_nativeGetSurfaceInfo
```

**libandroidx.graphics.path.so:**
```
mNativePath
```

**Итог:** Все JNI-методы — от AndroidX. Моего кода (`com.evgeniy.meetway`) нет.

### ищу строки приложения в .so

```bash
strings lib/arm64-v8a/libimage_processing_util_jni.so | grep -i "meetway\|secret\|key\|token"
```

иотого - Ничего не найдено. В библиотеках нет строк, относящихся к MeetWay.

### проверяю debug-символы

```bash
for f in *.so; do readelf -S "$f" | grep debug; done
```

че нашел=
```
libandroidx.graphics.path.so        — debug-символов НЕТ
libdatastore_shared_counter.so      — debug-символов НЕТ
libimage_processing_util_jni.so     — debug-символов НЕТ
libsurface_util_jni.so              — debug-символов НЕТ
```

Все библиотеки stripped (без отладочных символов)

### проверяю, есть ли вообще native-методы в коде MeetWay

```bash
jadx --deobf meetway_with_protection.apk
grep -rn "native " sources/com/evgeniy/
```

результат:** В исходниках MeetWay нет ни одного объявления `native` метода. Приложение **не использует JNI** вообще.

-------

## вывод

Приложение MeetWay **не содержит кастомного нативного кода (C/C++)**. Все .so файлы в APK — стандартные библиотеки AndroidX:

| библиотека                        | назначение                    | это системная?  |
| --------------------------------- | ----------------------------- | --------------- |
| `libandroidx.graphics.path.so`    | Графика Compose               | ✅ да (AndroidX) |
| `libdatastore_shared_counter.so`  | DataStore счётчик             | ✅ да (AndroidX) |
| `libimage_processing_util_jni.so` | Обработка изображений CameraX | ✅ да (AndroidX) |
| `libsurface_util_jni.so`          | Surface utilities CameraX     | ✅ да (AndroidX) |

**никакой security-логики в нативном коде нет:**



- ❌ Нет кастомных .so файлов
- ❌ Нет JNI-методов `com.evgeniy.meetway.*`
- ❌ Нет native-объявлений в Java/Kotlin коде
- ✅ Все системные .so - stripped, без debug-символов

### Тест MASTG-TEST-0369: **ПРОЙДЕН** 

Приложение **не использует нативный код для security-логики**. Обфусцировать нечего.

Если в будущем в MeetWay появится JNI-код (например, для криптографии, проверок безопасности), этот тест нужно будет перезапустить и убедиться, что нативные библиотеки обфусцированы
