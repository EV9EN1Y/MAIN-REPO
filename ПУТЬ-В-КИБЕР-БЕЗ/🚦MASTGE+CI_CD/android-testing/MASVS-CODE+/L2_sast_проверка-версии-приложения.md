# MASTG-TEST-0392: References to Enforced Updating APIs
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0392/

------

Тест проверяет, содержит ли код приложения признаки реализации принудительного обновления: либо через Google Play In-App Updates API, либо через кастомную проверку на сервере.

Принудительное обновление в реальном времени" - это когда приложение прямо во время работы проверяет, не устарела ли его версия, и, если это так, заставляет пользователя обновиться прямо сейчас, прежде чем он сможет продолжать им пользоваться

Тест выполняется с помощью **статического анализа** в JADX-GUI



откр  APK в JADX

а потом искать места:

- `AppUpdateManagerFactory` — создание менеджера обновлений.
    
- `AppUpdateManager` — основной класс для работы с обновлениями.
    
- `getAppUpdateInfo` — получение информации об обновлении.
    
- `startUpdateFlowForResult` — запуск процесса обновления.
    
- `AppUpdateType.IMMEDIATE` — немедленное обновление.
    
- `UpdateAvailability` — статус доступности обновления.
    
- `DEVELOPER_TRIGGERED_UPDATE_IN_PROGRESS` — статус обновления.
    
- `BuildConfig.VERSION_NAME` — имя версии приложения.
    
- `BuildConfig.VERSION_CODE` — код версии приложения.
    
- `PackageManager.getPackageInfo` — получение информации о пакете (версия).
    
- `minVersion` — минимальная версия.
    
- `updateRequired` — признак необходимости обновления

и смотреть - есть ли в приложении код который блокирует функции работы приложения проверив актуальность версии приложения, или нет!

--------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

------

##  результат (через jadx cli)

разобрал APK, прошёлся по всем маркерам:

### Google Play In-App Updates API

ничего из этого в коде **не найдено**:
- `AppUpdateManagerFactory` - -❌
- `AppUpdateManager` - ❌
- `getAppUpdateInfo` -- ❌
- `startUpdateFlowForResult` - ❌
- `AppUpdateType.IMMEDIATE` --- ❌
- `UpdateAvailability` / `DEVELOPER_TRIGGERED_UPDATE_IN_PROGRESS` – ❌

вывод: приложение **не использует** Google Play In-App Updates API

поиск других ключей:
- `BuildConfig.VERSION_NAME = "1.0"` / `VERSION_CODE = 1` - базовые константы, нигде не сравниваются с серверными значениями
- `PackageManager.getPackageInfo` – только в `SecurityDetector` для проверки рут-пакетов, не для проверки своей версии
- `minVersion` / `updateRequired` – ❌ не найдено
- `forceUpdate` / `updateRequired` / `app_version` / `min_version` -– ❌ не найдено нигде в app-коде

------

##  вывод

по тесту MASTG-TEST-0392 - **PASS**. Принудительного обновления нет.

но с точки зрения MASVS-L2 -то **надо смотреть в контексте**:
- если бизнес-требование требует контроля версий - сейчас его нет
- старые/скомпрометированные версии апки будут работать вечно
- серверный `loadAppAlert` выдаёт только текст, без блокировки функционала

---

фактически: MeetWay не имеет никакой защиты от использования устаревшей версии приложения. Всё, что есть - пассивный алерт с сервера, который ни к чему не обязывает пользователя

-------

## Результаты SAST

### Что искал:
- `AppUpdateManagerFactory`, `AppUpdateManager`, `getAppUpdateInfo` - Google Play In-App Updates API
- `startUpdateFlowForResult`, `AppUpdateType.IMMEDIATE`, `UpdateAvailability` - признаки немедленного обновления
- `BuildConfig.VERSION_NAME / VERSION_CODE` - версия приложения
- `PackageManager.getPackageInfo` – получение информации о пакете
- `minVersion`, `updateRequired`, `forceUpdate`, `min_version` - кастомная проверка
- `getAppAlert` - возможный кастомный механизм алертов с сервера


принудительное обновление НЕ РЕАЛИЗОВАНО

Ни через Google Play In-App Updates API, ни через кастомную логику на сервере. Приложение не проверяет актуальность версии и не блокирует работу при устаревшей версии
