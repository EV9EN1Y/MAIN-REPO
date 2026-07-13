# MASTG-TEST-0223: Stack Canaries Not Enabled
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0223/

-----

Stack Canaries - это специальные "сторожевые" значения, которые кладутся в стек перед вызовом функции. Если происходит переполнение буфера, canary повреждается, и программа падает, не давая злоумышленнику выполнить свой код

Это стандартная защита, которую компилятор включает по умолчанию для NDK-библиотек с флагами `-fstack-protector-strong` или `-fstack-protector-all`

Тест проверяет, есть ли в `.so` файлах символ `__stack_chk_fail` (это функция, которая вызывается, если canary поврежден). Если символа нет - скорее всего, защита отключена.

### Как проводить тест

Тест статический - анализ `.so` файлов из APK

**1. Извлеки `.so` файлы** из APK (через jadx-gui или `unzip`)

**2.  `readelf`** для поиска символа `__stack_chk_fail`

**Важно:** Для `.so` файлов (shared libraries) секция `.symtab` часто вырезана.
Используй `--dyn-syms` вместо `-s`:

```bash
readelf --dyn-syms имя_библиотеки.so | grep __stack_chk_fail
```

Или через `nm`:

```bash
nm -D имя_библиотеки.so | grep __stack_chk_fail
```

---

##  Проверка Stack Canaries

### Все библиотеки одной командой:

```bash
for lib in /Users/evgeniy/Desktop/Новая\ папка/arm64-v8a/*.so; do \
  result=$(nm -D "$lib" 2>/dev/null | grep __stack_chk_fail); \
  if [ -n "$result" ]; then \
    echo "✅ $(basename $lib): канарейка есть"; \
  else \
    echo "❌ $(basename $lib): канарейки НЕТ"; \
  fi; \
done
```

### Или по одному:

```bash
nm -D /Users/evgeniy/Desktop/Новая\ папка/arm64-v8a/libandroidx.graphics.path.so | grep __stack_chk_fail
nm -D /Users/evgeniy/Desktop/Новая\ папка/arm64-v8a/libdatastore_shared_counter.so | grep __stack_chk_fail
nm -D /Users/evgeniy/Desktop/Новая\ папка/arm64-v8a/libimage_processing_util_jni.so | grep __stack_chk_fail
nm -D /Users/evgeniy/Desktop/Новая\ папка/arm64-v8a/libsurface_util_jni.so | grep __stack_chk_fail
```

### Интерпретация:

- `__stack_chk_fail` **найден** (даже как U/UNDEF - импорт) - stack canaries включены ✅
- `__stack_chk_fail` **не найден** - защиты нет ❌

---

##  Результат:

```
✅ libandroidx.graphics.path.so: U __stack_chk_fail@LIBC
✅ libdatastore_shared_counter.so: U __stack_chk_fail@LIBC
✅ libimage_processing_util_jni.so: U __stack_chk_fail@LIBC
✅ libsurface_util_jni.so: U __stack_chk_fail@LIBC


✅ libandroidx.graphics.path.so: канарейка есть
✅ libdatastore_shared_counter.so: канарейка есть
✅ libimage_processing_util_jni.so: канарейка есть
✅ libsurface_util_jni.so: канарейка есть
```



---

##  Вывод: тест пройден!

Stack canaries включены во всех нативных библиотеках MeetWay.
**Моя проверка не совпала с перепроверкой** - из-за неверной команды `readelf -s` вместо `readelf --dyn-syms`

