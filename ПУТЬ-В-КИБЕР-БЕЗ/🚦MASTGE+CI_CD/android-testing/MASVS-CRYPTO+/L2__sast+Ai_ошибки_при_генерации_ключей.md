# MASTG-TEST-0307: References to Asymmetric Key Pairs Used For Multiple Purposes
https://mas.owasp.org/MASTG/tests/android/MASVS-CRYPTO/MASTG-TEST-0307/

Этот тест - **MASTG-TEST-0307** - проверяет, **не используется ли одна и та же асимметричная ключевая пара (например, RSA) для нескольких разных целей** (например, и для шифрования, и для подписи). Это нарушает принцип разделения обязанностей и может привести к компрометации ключа, если одна из функций будет взломана

Тест проверяет, как приложение создает асимметричные ключи (обычно через `KeyPairGenerator`и `KeyGenParameterSpec`) и какие **назначения (purposes)** задает для ключа. Ключ должен использоваться только для одной конкретной цели: шифрование/дешифрование, подпись/верификация или обертка ключей. Если ключу разрешено делать несколько разных вещей (например, и шифровать, и подписывать), это считается плохой практикой

**Правильные (безопасные) комбинации:**

- Только шифрование: `PURPOSE_ENCRYPT` (1)
    
- Только дешифрование: `PURPOSE_DECRYPT` (2)
    
- Шифрование и дешифрование: `PURPOSE_ENCRYPT | PURPOSE_DECRYPT` (3)
    
- Только подпись: `PURPOSE_SIGN` (4)
    
- Только верификация: `PURPOSE_VERIFY` (8)
    
- Подпись и верификация: `PURPOSE_SIGN | PURPOSE_VERIFY` (12)
    
- Только обертка ключа: `PURPOSE_WRAP_KEY` (32)
   

**Опасная комбинация:** `15` (все четыре цели: шифрование, дешифрование, подпись, верификация)

-------
- **Тест пройден:** Все найденные ключи имеют корректные, ограниченные наборы назначений.
    
- **Тест провален:** Найден ключ, который используется для нескольких ролей (например, и для шифрования, и для подписи).

-----


пример нарушения



```java
KeyGenParameterSpec spec = new KeyGenParameterSpec.Builder(
        alias,
        KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_SIGN | KeyProperties.PURPOSE_DECRYPT | KeyProperties.PURPOSE_VERIFY )
        .setKeySize(keySize)
        .build();
KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA", "AndroidKeyStore");
keyGen.initialize(spec);
```

Здесь ключу разрешено делать всё, что является явным нарушением!

-------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```


начинаю искать основные точки входа

- `KeyGenParameterSpec.Builder` - класс для создания спецификаций ключа
   
- `KeyPairGenerator` - для генерации ключевых пар
   
- `KeyProperties` - содержит константы для назначений (например, `PURPOSE_ENCRYPT`

```q
- Если передается `KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT` (или другие разрешенные комбинации) — это безопасно.
    
- Если передается что-то, что включает `PURPOSE_SIGN` вместе с `PURPOSE_ENCRYPT` или `PURPOSE_DECRYPT` — это критично.
```

-----

## результаты теста

Тест **пройден** 🟢

**Факт:** в коде приложения (com.evgeniy.meetway) **отсутствует** использование асимметричных ключей:
- `KeyPairGenerator` -- не используется
- `KeyGenParameterSpec.Builder` -- не используется
- `PURPOSE_ENCRYPT / SIGN / DECRYPT / VERIFY` -- не используются
- `AndroidKeyStore` -- не используется

**Что используется вместо этого:**
- `EncryptionUtil.java` - `SecretKeySpec` (симметричный AES-GCM) для шифрования сообщений чата
- `ObjectStorageService.java` - `SecretKeySpec` (HMAC-SHA256) для подписи S3-запросов
- `SecureStorage.java` - `MasterKey.Builder` (Android EncryptedSharedPreferences, AES256-GCM)


Нарушений нет

