# MASTG-TEST-0243: Expired Certificate Pins in the Network Security Configuration
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0243/

## что проверяет этот тест

Тест MASTG-TEST-0243 проверяет, **не истекли ли сроки** у certificate pins в NSC. Если у `<pin>` указан `expiration`, и дата в прошлом – пиннинг перестаёт работать, и приложение возвращается к обычной проверке через доверенные CA. Разработчик может думать, что защита есть, а на деле её уже нет.

## какие инструменты использую

беру meetway.apk, через jadx-gui смотрю NSC:

```bash
jadx-gui ~/Desktop/meetway.apk
```

в `network_security_config.xml` ищу `<pin expiration="...">`.

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Resources → res/xml/ → `network_security_config.xml`**

**шаг 3 – проверяю наличие `<pin-set>` и `<pin expiration="...">`**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `<pin-set>` в NSC | ❌ нет |
| `<pin expiration="...">` | ❌ нет |

Поскольку NSC не содержит pin-set вообще (см. MASTG-TEST-0242), проверять нечего.

## вывод

**Тест пройден ✅** (L2)

Причина: нет pin-set в NSC – нет и срока действия. Программный pinning через OkHttp (SslPinningInterceptor, GRPCInterceptor) не использует expiration – хеши живут до следующего обновления приложения.

Уровень теста: L2.
Профиль: MASVS-NETWORK.

---
