# MASTG-TEST-0330: References to APIs for Keys used in Biometric Authentication with Extended Validity Duration
https://mas.owasp.org/MASTG/tests/android/MASVS-AUTH/MASTG-TEST-0330/

не устанавливает ли приложение слишком долгий срок действия ключа, который был получен после биометрической аутентификации

----------------

В Android есть механизм, который позволяет установить время жизни ключа после успешной биометрической аутентификации. Если этот срок установлен в `0` секунд, то для каждой операции требуется повторная аутентификация (самый безопасный вариант)

Если срок установлен больше `0`, то после успешной аутентификации ключ остается разблокированным на заданное количество секунд (например, 30 секунд, 5 минут, 1 час). Это удобно, если нужно выполнить несколько операций подряд, но если срок слишком долгий (минуты или часы), злоумышленник, получивший доступ к разблокированному телефону, может выполнить критическую операцию без повторной биометрии

За это отвечают методы:

- `setUserAuthenticationParameters(int timeout, int type)` - современный способ.
   
- `setUserAuthenticationValidityDurationSeconds(int)` - устаревший способ.
   

Если `timeout > 0` для критической операции (платеж, доступ к личным данным) - это потенциальная уязвимость


------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

ищу


- `setUserAuthenticationParameters(int timeout, int type)` - современный способ.
   
- `setUserAuthenticationValidityDurationSeconds(int)` - устаревший способ
-----

нашел

```q
            static MasterKey build(Builder builder) throws GeneralSecurityException, IOException {  
                if (builder.mKeyScheme == null && builder.mKeyGenParameterSpec == null) {  
                    throw new IllegalArgumentException("build() called before setKeyGenParameterSpec or setKeyScheme.");  
                }  
                if (builder.mKeyScheme == KeyScheme.AES256_GCM) {  
                    KeyGenParameterSpec.Builder keyGenBuilder = new KeyGenParameterSpec.Builder(builder.mKeyAlias, 3).setBlockModes(CodePackage.GCM).setEncryptionPaddings("NoPadding").setKeySize(256);  
                    if (builder.mAuthenticationRequired) {  
                        keyGenBuilder.setUserAuthenticationRequired(true);  
                        Api30Impl.setUserAuthenticationParameters(keyGenBuilder, builder.mUserAuthenticationValidityDurationSeconds, 3);  
                    }  
                    if (builder.mRequestStrongBoxBacked && builder.mContext.getPackageManager().hasSystemFeature("android.hardware.strongbox_keystore")) {  
                        Api28Impl.setIsStrongBoxBacked(keyGenBuilder);  
                    }  
                    builder.mKeyGenParameterSpec = keyGenBuilder.build();  
                }  
                if (builder.mKeyGenParameterSpec == null) {  
                    throw new NullPointerException("KeyGenParameterSpec was null after build() check");  
                }  
                String keyAlias = MasterKeys.getOrCreate(builder.mKeyGenParameterSpec);  
                return new MasterKey(keyAlias, builder.mKeyGenParameterSpec);  
            }

  static class Api30Impl {  
                private Api30Impl() {  
                }  
  
                static void setUserAuthenticationParameters(KeyGenParameterSpec.Builder builder, int timeout, int type) {  
                    builder.setUserAuthenticationParameters(timeout, type);  
                }  
            }
            
            



```

setUserAuthenticationValidityDurationSeconds  - не обнаружено в коде!

-------

setUserAuthenticationParameters - найден только в методах библиотеки!
и в самом коде - метод не вызывается! 
По умолчанию
mAuthenticationRequired = false → весь блок с setUserAuthenticationParameters
никогда не выполняется

------

## вывод - тест пройден
