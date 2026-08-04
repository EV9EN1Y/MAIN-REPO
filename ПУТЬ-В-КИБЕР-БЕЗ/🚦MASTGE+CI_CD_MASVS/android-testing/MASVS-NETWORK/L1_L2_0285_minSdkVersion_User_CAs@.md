# MASTG-TEST-0285: Outdated Android Version Allowing Trust in User-Provided CAs
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0285/

## что проверяет этот тест

Тест MASTG-TEST-0285 проверяет **`minSdkVersion`** приложения. Если `minSdkVersion < 24` (Android 7.0), приложение может быть установлено на устройства, где по умолчанию доверяются **пользовательские CA-сертификаты**. Это означает, что любой, кто физически имеет доступ к телефону, может установить свой корневой сертификат и перехватывать HTTPS-трафик через MITM.

Начиная с API 24 (Android 7.0) приложения по умолчанию **не доверяют** user-сертификатам – только системным.

## какие инструменты использую

беру meetway.apk и декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный просмотр AndroidManifest.xml проверяю `<uses-sdk android:minSdkVersion="...">`

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – вкладка AndroidManifest.xml → ищу `<uses-sdk>` или `minSdkVersion`**

## что нашёл

| Атрибут | Значение |
|--|--|
| `android:minSdkVersion` | **33** |
| `android:targetSdkVersion` | **36** |

Минимальная версия Android – **13** (API 33). Это значительно выше порога API 24.

## вывод

**Тест пройден ✅**

Причина: `minSdkVersion=33` (Android 13). Приложение не может быть установлено на устройства с API < 24, где user-сертификаты доверяются по умолчанию. Уязвимость не применима.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
