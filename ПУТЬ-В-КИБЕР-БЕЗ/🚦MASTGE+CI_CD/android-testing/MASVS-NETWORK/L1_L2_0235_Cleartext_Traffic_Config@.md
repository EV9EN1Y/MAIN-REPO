# MASTG-TEST-0235: Android App Configurations Allowing Cleartext Traffic
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0235/

## что проверяет этот тест

Тест MASTG-TEST-0235 проверяет, разрешает ли приложение **незашифрованный HTTP-трафик** (cleartext). Начиная с Android 9 (API 28) HTTP-трафик заблокирован по умолчанию, но разработчик может включить его двумя способами:

1. **AndroidManifest.xml** – атрибут `android:usesCleartextTraffic="true"` у `<application>`
2. **Network Security Configuration (NSC)** – файл `res/xml/network_security_config.xml`, где в `<base-config>` или `<domain-config>` стоит `cleartextTrafficPermitted="true"`

Если хотя бы один из этих флагов включён, трафик приложения можно перехватить в открытом виде через MITM – пароли, токены, сообщения уйдут как есть, без шифрования.

## какие инструменты использую

беру meetway.apk и декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через jadx смотрю:
1. Вкладка AndroidManifest.xml – атрибут `usesCleartextTraffic`
2. Ссылка `networkSecurityConfig` – открываю XML-файл
3. В NSC проверяю `cleartextTrafficPermitted` в `<base-config>` и `<domain-config>`

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui:**
  - Проверяю наличие `android:usesCleartextTraffic` у `<application>`
  - Проверяю наличие `android:networkSecurityConfig`

**шаг 2 – если есть networkSecurityConfig, открываю файл через jadx (Resources → res/xml/):**
  - Проверяю `<base-config cleartextTrafficPermitted="...">`
  - Проверяю `<domain-config cleartextTrafficPermitted="...">` для каждого домена

## что нашёл


<img src="../../../assets/Снимо2026-07-2221319.14.32.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

**AndroidManifest.xml:**

| Атрибут | Значение |
|--|--|
| `android:usesCleartextTraffic` | ❌ не установлен (по умолчанию false для API 28+) |
| `android:networkSecurityConfig` | ✅ установлен – `@xml/network_security_config` |

**Network Security Configuration (`network_security_config.xml`):**

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

| Элемент | cleartextTrafficPermitted | Статус |
|--|--|--|
| `<base-config>` | `false` | ✅ безопасно |
| `<domain-config>` (все домены) | `false` | ✅ безопасно |

## вывод

**Тест пройден ✅**

Причина: cleartext-трафик явно запрещён на всех уровнях. Ни `usesCleartextTraffic`, ни `cleartextTrafficPermitted` не включены. Весь трафик приложения идёт через HTTPS. Доверяются только системные CA (`<certificates src="system"/>`), user-сертификаты не принимаются – MITM через подстановку своего CA не пройдёт без дополнительного обхода (Frida/unpin).

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
