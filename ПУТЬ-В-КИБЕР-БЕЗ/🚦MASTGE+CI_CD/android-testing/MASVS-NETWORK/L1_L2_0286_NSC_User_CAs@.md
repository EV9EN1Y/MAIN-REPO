# MASTG-TEST-0286: Network Security Configuration Allowing Trust in User-Provided CAs
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0286/

## что проверяет этот тест

Тест MASTG-TEST-0286 проверяет **Network Security Configuration** на предмет доверия пользовательским CA. Даже если `minSdkVersion >= 24`, разработчик может явно переопределить поведение и включить `<certificates src="user"/>` в NSC. Это разрешит MITM через user-сертификаты, что сломает защиту Android 7+.

## какие инструменты использую

беру meetway.apk, через jadx-gui смотрю NSC-файл (тот же, что и в тесте 0235)

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – во вкладке Resources → res/xml/ открываю `network_security_config.xml`**

**шаг 3 – проверяю `<trust-anchors>` на наличие `<certificates src="user"/>`**

## что нашёл

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">functions.yandexcloud.net</domain>
        <domain includeSubdomains="true">storage.yandexcloud.net</domain>
        <domain includeSubdomains="true">firestore.googleapis.com</domain>
        <domain includeSubdomains="true">identitytoolkit.googleapis.com</domain>
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </domain-config>
</network-security-config>
```

| `<trust-anchors>` | Значение | Статус |
|--|--|--|
| `<base-config>` | `<certificates src="system"/>` | ✅ только системные |
| `<domain-config>` | `<certificates src="system"/>` | ✅ только системные |
| `<certificates src="user"/>` | ❌ не используется | ✅ |

## вывод

**Тест пройден ✅**

Причина: NSC доверяет только системным CA (`<certificates src="system"/>`). Пользовательские сертификаты не принимаются. MITM через установку своего CA на телефоне не пройдёт без обхода через Frida/unpin.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
