https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0352/
# MASTG-TEST-0352: References to Debugging Detection APIs

**есть ли в коде приложения механизмы обнаружения отладки (debugging detection)**

-------
ест проверяет, пытается ли приложение понять, что к нему подключен отладчик (например, через Android Studio, JDB или Frida). Если такие проверки есть - это защита от динамического анализа. Если нет - приложение уязвимо к отладке

------

Приложения могут проверять отладку на двух уровнях:

- **Java/Kotlin уровень:** Использование API `Debug.isDebuggerConnected()` и проверка флага `ApplicationInfo.FLAG_DEBUGGABLE`
   
- **Нативный уровень (C/C++):** Проверка `TracerPid` в файле `/proc/self/status`, вызовы `ptrace`, проверка `ro.debuggable` и других свойств

========================================================

### Как проводить тест

Тест выполняется с помощью **статического анализа** (просмотр кода)

**Для Java/Kotlin кода (JADX-GUI):**

1. APK в JADX
   
2. глобальный поиск (`Ctrl+Shift+F` или `Cmd+Shift+F`)
   
3. поиск по ключ слова:
   
    - `Debug.isDebuggerConnected`
      
    - `FLAG_DEBUGGABLE`
      
    - `isDebuggerConnected`
      

**Для нативных библиотек (`.so` файлы):**

1. Извлечь из APK папку `lib/` (содержит `.so` файлы)
    
2. далее - дизассемблер (например, `objdump` или `readelf` или Ghidra/IDA)
    
3. и искать паттерны:
    
    - Вызовы `ptrace`
      
    - Чтение файла `/proc/self/status` и поиск строки `TracerPid:`
      
    - Проверка системных свойств (например, `ro.debuggable`)

-------------


##  запускаю  jadx
```c
 jadx-gui /Users/evgeniy/Desktop/apk-meetway-с\=защитой-от\=отладки\=и\=взлома/meetway_with_protection.apk
```


# выполняю поиск


Generic-поиск по JADX (для любого APK)

### Уровень 1 — Отладчик (Debug Detection)

Эти методы стандартные, разработчики редко их переименовывают:

```bash
  # В JADX Cmd+Shift+F:

  android.os.Debug                      # весь класс Debug
  Debug.isDebuggerConnected              # прямой вызов
  isDebuggerConnected                    # может быть без префикса
  FLAG_DEBUGGABLE                        # флаг манифеста
  ApplicationInfo.FLAG_DEBUGGABLE        # полный путь
  ro.debuggable                          # системное свойство
  TracerPid                              # /proc/self/status
```

### Уровень 2 — Смертельные методы (куда приводят проверки)

Даже если класс обозвали MySuperProtection, он в конце вызывает:

```bash
  Process.killProcess                    # убить процесс
  Process.myPid                          # получить свой PID
  System.exit                            # выйти
  Runtime.getRuntime().exit              # альтернативный выход
  killProcess                            # без префикса
```

Если нашёл killProcess или System.exit — смотри stack trace, кто вызвал.

### Уровень 3 — Root-детекция (generic strings)

Эти пути разработчики не меняют, они жёстко зашиты:

```bash
  /system/bin/su                         # путь к su
  /system/xbin/su
  /system/bin/failsafe/su
  /sbin/su
  /su/bin/su
  /data/local/tmp/su                     # временный su
  /data/local/tmp/frida-server           # Frida
  /data/local/tmp/re.frida.server        # Frida (альт)
  /data/local/tmp/frida-agent            # Frida gadget
  /data/local/tmp/linjector              # Frida альтернатива
  /system/jailbreak_test                 # проверка записи в system
  /system/recovery-from-boot.p           # кастомная recovery
  /system/etc/recovery.img
  /cache/recovery
```

### Уровень 4 — Опасные пакеты (PackageManager)

Разработчики проверяют эти package names, они стандартные:

```bash
  com.topjohnwu.magisk                   # Magisk
  com.noshufou.android.su                # SuperSU
  eu.chainfire.supersu
  com.thirdparty.superuser
  de.robv.android.xposed.installer       # Xposed
  io.va.exposed                          # VirtualXposed
  me.weishu.exp                          # Exposed
  com.saurik.substrate                   # Cydia Substrate
```

### Уровень 5 — Нативные проверки (в .so файлах)

Если приложение тянет кастомный .so (не AndroidX, не Firebase):

```bash
  # strings по .so файлам (терминал):
  strings lib/*.so | grep -i ptrace
  strings lib/*.so | grep -i "TracerPid\|/proc/self/status"
  strings lib/*.so | grep -i "ro\.debuggable\|getprop"
  strings lib/*.so | grep -i "frida\|gadget\|agent"
  strings lib/*.so | grep -i "jdwp\|debugger"
  strings lib/*.so | grep -i "antidebug\|anti_debug\|antiDebug"
```

### Уровень 6 — Самодельные классы (остаётся только искать по логике)

Если разработчик назвал класс abcABC, generic уже не спасёт. Тогда смотри:

```bash
  # Поиск импорта НЕ из Android SDK
  import                                            # все импорты в странных классах

  # Поиск JNI нативных методов (подгрузка кастомной .so)
  System.loadLibrary                                # загрузка нативной библиотеки
  native                                            # JNI метод

  # Поиск рефлексии (если проверки спрятаны)
  Class.forName                                     # загрузка класса по строке
  getDeclaredMethod                                 # получение метода по строке
  invoke                                            # вызов через рефлексию
```

-----

#### теперь нужно проверить нативные .so библиотеки


---

**APK:** `meetway_with_protection.apk` (сборка 01.07.2026)
**Инструмент:** JADX + strings

работаю совместно с open claw
---

## Уровень 1 - Debug Detection (стандартные API)

| Ключевое слово | Результат |
|---|---|
| `android.os.Debug` | ❌ Не найдено |
| `Debug.isDebuggerConnected()` | ❌ Не найдено |
| `isDebuggerConnected` | ❌ Не найдено |
| `FLAG_DEBUGGABLE` | ✅ **Найдено** (в MeetWayApp — для лога, не для kill) |
| `ApplicationInfo.FLAG_DEBUGGABLE` | ✅ **Найдено** (там же) |
| `ro.debuggable` | ❌ Не найдено |
| `TracerPid` | ❌ Не найдено |

**Вывод:** Стандартные `Debug.isDebuggerConnected()` не используются. `FLAG_DEBUGGABLE` используется только для логирования

**Generic search strings (на что жать в JADX):**
- ❌ `android.os.Debug` — не найден
- ❌ `Debug.isDebuggerConnected` — не найден
- ✅ `FLAG_DEBUGGABLE` — **найден** (но это лог, не защита)

---

## Уровень 2 — Смертельные методы (крашат аппку)

| Ключевое слово | Результат | Где |
|---|---|---|
| `Process.killProcess` | ✅ **Найдено** | MeetWayApp.java:135 |
| `Process.myPid()` | ✅ **Найдено** | MeetWayApp.java:135 |
| `System.exit(1)` | ✅ **Найдено** | MeetWayApp.java:136 |
| `Runtime.getRuntime().exit` | ❌ Не найдено | — |

**Контекст вызова:**
```java
private void detectCompromisedDevice() {
    if (SecurityDetector.INSTANCE.isDeviceCompromised(this)) {
        Log.w(TAG, "⚠️⚠️⚠️ ОБНАРУЖЕНО ВЗЛОМАННОЕ УСТРОЙСТВО!");
        Process.killProcess(Process.myPid());  // ← строка 135
        System.exit(1);                         // ← строка 136
    }
}
```

прям четко все видно! 
и логи есть!
и метод ясно дает понять что он делает! System.exit

**Generic search strings (на что жать в JADX):**
- ✅ `Process.killProcess` — найден
- ✅ `System.exit` — **найден**

---

## Уровень 3 — Root-детекция (generic строки)

| Путь/строка | Результат |
|---|---|
| `/system/bin/su` | ❌ Не найдено (через другой механизм) |
| `/sbin/su` | ❌ Не найдено (через другой механизм) |
| `/data/local/tmp/frida-server` | ❌ Не найдено (другой механизм) |
| `/data/local/tmp/frida-agent` | ❌ Не найдено |
| `/system/jailbreak_test` | ❌ Не найдено |
| `/system/recovery-from-boot.p` | ❌ Не найдено |
| `frida` (как строка) | ❌ Не найдено в строках |

**Вывод:** Приложение НЕ использует File.exists() для root-путей. Вместо этого использует: exec(`su -c id`), TCP-соединение на порт 27042 (Frida), `cat /proc/1/mounts`

**Generic search strings (на что жать в JADX):**
- ❌ Ни одна из жёстких строк не нажалась

---

## Уровень 4 - Опасные пакеты

| Пакет | Результат |
|---|---|
| `com.topjohnwu.magisk` | ❌ Не найдено (другой механизм) |
| `com.noshufou.android.su` | ❌ Не найдено |
| `eu.chainfire.supersu` | ❌ Не найдено |
| `de.robv.android.xposed.installer` | ❌ Не найдено |
| `com.saurik.substrate` | ❌ Не найдено |

**Вывод:** PackageManager не используется для проверки пакетов. Magisk детектится через файлы `/data/adb/magisk/` и mount table.

**Generic search strings (на что жать в JADX):**
- ❌ Ни один из пакетов не найден
- ✅ `magisk` — **найден** (в строках `/data/adb/magisk/...`)
- ✅ `/data/adb` — **найден**

---

## Уровень 5 - Нативные .so библиотеки

Все `.so` файлы в APK (только AndroidX/Jetpack):

| .so файл | Размер | ptrace | TracerPid | ro.debuggable | frida | anti-debug |
|---|---|---|---|---|---|---|
| `libandroidx.graphics.path.so` | 10 KB | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто |
| `libdatastore_shared_counter.so` | 7 KB | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто |
| `libimage_processing_util_jni.so` | 29 KB | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто |
| `libsurface_util_jni.so` | 5 KB | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто | ✅ Чисто |

**Команда проверки (терминал):* все сразу по очереди!! удобно!*
```bash
for f in /path/to/lib/arm64-v8a/*.so; do
    name=$(basename $f)
    found=$(strings "$f" | grep -ciE "ptrace|tracerpid|/proc/self/status|ro\.debuggable|frida|gadget|jdwp|antidebug")
    echo "$name: $found matches"
done
```

**Вывод:** Кастомных `.so` библиотек нет. Только стандартные AndroidX. Вся защита — в Java/Kotlin слое

---

## Уровень 6 — Самодельные классы (поиск по логике)

### `System.loadLibrary` / native JNI
| Ключевое слово | Результат |
|---|---|
| `System.loadLibrary` | ❌ Не найдено в app-коде |
| `native` методы | ❌ Не найдено в app-коде |

### `Class.forName` рефлексия
| Ключевое слово | Результат |
|---|---|
| `Class.forName` | ❌ Не найдено в util/ |
| `getDeclaredMethod` | ❌ Не найдено в util/ |
| `.invoke()` | ❌ Не найдено в util/ |

---

## ИТОГО

### Тест  **ЧАСТИЧНО ПРОЙДЕН**

### Что обнаружено (6 проверок в SecurityDetector):

| № | Проверка | Метод | Детектит |
|---|---|---|---|
| 1 | `checkRootViaExec()` | `Runtime.exec("su -c id")` | Root доступ |
| 2 | `checkMagiskFiles()` | File.exists(`/data/adb/magisk/...`) | Magisk |
| 3 | `checkMagiskMount()` | Runtime.exec(`cat /proc/1/mounts`) | Magisk mount |
| 4 | `checkTestKeys()` | `Build.TAGS.contains("test-keys")` | Userdebug/eng прошивка |
| 5 | `checkSelinux()` | `Runtime.exec("getenforce")` + /sys/fs/selinux/enforce | SELinux permissive |
| 6 | `checkFridaPort()` | Socket(`127.0.0.1:27042`) | Frida-server |

### Чего НЕТ:

| Проверка | Отсутствует |
|---|---|
| `Debug.isDebuggerConnected()` | ❌ |
| `ptrace` / `TracerPid` | ❌ |
| Кастомная .so с анти-отладкой | ❌ |
| Проверка `ro.debuggable` | ❌ |
| Проверка `isUnderTest` | ❌ |

### Вывод:
- ❌ Классический `Debug.isDebuggerConnected()` не используется - обычный JDB отладчик не детектится
- ✅ Frida-server детектится через TCP-порт 27042
- ✅ Root/Magisk детектится через exec + проверку файлов
- ✅ Userdebug прошивка детектится через Build.TAGS
- ✅ SELinux permissive детектится

Приложение защищено от **рута/модификации системы/Frida-сервера**, но не защищено от **JDB/Android Studio отладчика** напрямую

---








