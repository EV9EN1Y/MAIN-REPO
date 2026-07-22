# MASTG-TEST-0242: Missing Certificate Pinning in Network Security Configuration
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0242/

## что проверяет этот тест

Тест MASTG-TEST-0242 проверяет, настроен ли **certificate pinning** через Network Security Configuration (NSC). Pinning – это механизм, который говорит: «доверять можно только конкретному сертификату/публичному ключу для этого домена». Даже если CA скомпрометирован – MITM не пройдёт.

NSC-пиннинг настраивается через `<pin-set>` в `<domain-config>`. Если пингов нет – атакующий с доступом к CA может подменить сертификат.

**Важно:** тест проверяет именно NSC-пиннинг, не programmatic (OkHttp CertificatePinner). Pinning может быть реализован и в коде – это перекрывается тестом MASTG-TEST-0244.

## какие инструменты использую

беру meetway.apk, открываю NSC через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

во вкладке Resources → res/xml/ открываю `network_security_config.xml` и проверяю `<pin-set>`.

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Resources → res/xml/ → `network_security_config.xml`**

**шаг 3 – ищу `<pin-set>` внутри `<domain-config>`**

## что нашёл

**NSC (`network_security_config.xml`):**

```xml
<domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">functions.yandexcloud.net</domain>
    <domain includeSubdomains="true">storage.yandexcloud.net</domain>
    <domain includeSubdomains="true">firestore.googleapis.com</domain>
    <domain includeSubdomains="true">identitytoolkit.googleapis.com</domain>
    <trust-anchors>
        <certificates src="system"/>
    </trust-anchors>
</domain-config>
```

| Элемент | Наличие pin-set |
|--|--|
| `<domain-config>` (4 домена) | ❌ нет `<pin-set>` |

**Домены, где стоило бы добавить `<pin-set>`:**
| Домен | Назначение |
|--|--|
| `functions.yandexcloud.net` | Cloud Functions (API-запросы) |
| `storage.yandexcloud.net` | Object Storage (фото) |
| `baket-ivaaro.storage.yandexcloud.net` | Object Storage bucket (фото) |
| `firestore.googleapis.com` | Firebase Firestore |
| `identitytoolkit.googleapis.com` | Firebase Auth |
| `login.yandex.ru` | Yandex OAuth |

**NSC не содержит ни одного `<pin-set>`.** Однако приложение использует собственный programmatic pinning через OkHttp `CertificatePinner` в двух местах:
- `SslPinningInterceptor` – основной перехватчик с проверкой сертификатов для 5 доменов
- `GRPCInterceptor` – для Yandex SDK (4 домена)

Это programmatic pinning, а не через NSC.

## вывод

**Тест НЕ ПРОЙДЕН ❌** (L2)

Причина: NSC не содержит `<pin-set>` ни для одного домена. Certificate pinning реализован программно (через OkHttp CertificatePinner), но не через Network Security Configuration. Для L2 это считается нарушением, так как NSC-пиннинг проще поддерживать и обновлять без изменения кода приложения.

**Рекомендация:** добавить `<pin-set>` с теми же хешами в `network_security_config.xml` для дублирования защиты на уровне системы.

Уровень теста: L2.
Профиль: MASVS-NETWORK.

---
