# MASTG-TEST-0266: Ссылки на API для биометрической аутентификации на основе событий

**MASTG-TEST-0266 (статический)** - смотрит, _написан ли_ код для биометрии, и как он реализован
**MASTG-TEST-0267 (динамический)** - проверяет, _действительно ли приложение правильно вызывает_ этот код во время работы

#### Если в Keychain хранятся какие-то данные (токены/пароли)

##### *то есть, есть два подхода при получении данных из кейчан*

🟡 первый подход - это- сохраняю  простым способом данные в кейчейн и потом запрашиваю данные из него - и получаю их (это решает проблему безопасного хранения данных), но это не спасается от Frida.
```swift
Сохраняю: SecItemAdd(токен, nil)
Запрашиваю: SecItemCopyMatching(...)
Результат: → токен получен (без Face ID)
```


🟢 второй подход - это сделать так, чтобы при сохранении в  кейчан сохранить данные с флагом что требуется фейс-айди при попытке запроса этих данных, и тогда, чтобы данные вытащить - кейчан запросит фейс-id
```swift
Сохраняю: SecItemAdd(токен, with: kSecAttrAccessControl + .biometryCurrentSet)
Запрашиваю: SecItemCopyMatching(...)
Результат: → Keychain САМ спросит Face ID → только потом отдаёт токен
```
не получиться получить токен, не пройдя Face ID. Даже если Frida подменит значение твоей функции, она не подменит системный диалог Keychain


---------

найти все места - где требуется face id и проверить - какой способ там используется для проверки (ручной или через Keychain)

теперь - нужно найти в коде все места - где используется кейчейн и понять, нужно ли там сделать так - чтобы данные сохранялись с требованием Face-id

а также найти места - где используется LAContext.evaluatePolicy - и понять, нужно ли там заменить это на метод через Keychain

также можно поискать в коде места - где требуется защитить данные и рассмотреть варианты защиты таких мест

-------

**Неправильно:**
```swift
//  хранение токена без биометрии
SecItemAdd(token, nil)

//  проверка Face ID перед запросом токена
if (LAContext.evaluatePolicy(.deviceOwnerAuthentication)) {
    let token = getFromKeychain()
    makeRequest(with: token)
}
```

**Правильно:**
```swift
//  сохранение токена С биометрией
let access = SecAccessControlCreateWithFlags(..., .biometryCurrentSet, ...)
SecItemAdd(token, [kSecAttrAccessControl: access])

//  при запросе - просто достаём токен
let token = getFromKeychain()  // ← Keychain сам спросит Face ID
makeRequest(with: token)

теперь - даже если у меня стырят разблокированный телефон - то без моего фейса - доступ к важным данным никто не получит (наверно...)

```


---------

### как считаются результаты?

если в коде используются токены - и доступ к ним разрешен только через LAContext.evaluatePolicy - то ,возможно, следует заменить этот метод на кейчейн
чтобы данные сохранялись в Keychain с флагом`kSecAccessControlBiometryCurrentSet` (или подобным)

-------

вот так выглядит в коде хранение токена без биометрии
```swift
SecItemAdd(token, nil)
```

-------
### как тестировать

статически - радар2

```q
#  вызовы биометрии
[0x00000000]> izz~LAContext
[0x00000000]> izz~evaluatePolicy


#  вызовы кейчан
[0x00000000]> izz~SecItemAdd
[0x00000000]> izz~SecItemCopyMatching
[0x00000000]> izz~SecAccessControlCreateWithFlags

# импорты
[0x00000000]> ii~LAContext
[0x00000000]> ii~SecItem
[0x00000000]> ii~SecAccess
[0x00000000]> ii~LocalAuthentication

/ SecAccess
/ SecItem
/ LAContext
/ LocalAuthentication

ну а потом смтореть где вызывается axt 
или полностью просматрить
```

-----

#### начинаю тест
тестирую приложение вот это - > [[0_MeetWay]]

открываю бинарник и начинаю анализ
```shell
MacBook-Pro MeetWay.app % ls -la
total 125928
-rwxr-xr-x   1 evgeniy  staff     35024 29 апр 14:31 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96 28 апр 14:29 _CodeSignature
drwxr-xr-x@ 16 evgeniy  staff       512 29 апр 14:31 .
drwxr-xr-x   5 evgeniy  staff       160 29 апр 14:38 ..
-rw-r--r--   1 evgeniy  staff     14412 29 апр 13:46 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132 29 апр 13:46 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696 29 апр 13:46 Assets.car
-rw-r--r--   1 evgeniy  staff     15136 28 апр 14:28 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864 28 апр 14:29 Frameworks
-rw-r--r--   1 evgeniy  staff       996 28 апр 14:28 GoogleService-Info.plist
-rw-r--r--   1 evgeniy  staff      3901 29 апр 14:20 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664 29 апр 14:31 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41606464 29 апр 14:31 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8 28 апр 14:28 PkgInfo
drwxr-xr-x   4 evgeniy  staff       128 28 апр 14:28 TrustKit_TrustKit.bundle
-rw-r--r--   1 evgeniy  staff     49852 28 апр 14:28 words.txt
MacBook-Pro MeetWay.app % r2 -A ./MeetWay.debug.dylib
```

# первичный анализ 

![[Снимок экрана 2026-04-30 в 13.12.49.png]]

```shell
[0x00004000]> izz~LAContext

58101  0x00ab8f2a 0x00ab8f2a 23   24 ascii   _OBJC_CLASS_$_LAContext
70802  0x00db018e 0x00db018e 23   24 ascii   _OBJC_CLASS_$_LAContext
82325  0x018172a9 0x018172a9 22   23  ascii   _$sSo9LAContextCABycfC
82345  0x01817935 0x01817935 18   19 ascii   _$sSo9LAContextCMa
82414  0x0181a641 0x0181a641 24   25   ascii   _$sSo9LAContextCABycfcTO
102003 0x02627016 0x02627016 18   19   ascii   _$sSo9LAContextCML
```
вижу _OBJC_CLASS_$_LAContext - значит в коде используется  LAContext

------

```shell
[0x00004000]> izz~evaluatePolicy

54400  0x00a46cba 0x00a46cba 37   38   8.__TEXT.__objc_methname   ascii   evaluatePolicy:localizedReason:reply:
```
вижу evaluatePolicy:localizedReason:reply: - значит в коде вызывается вручную фейс-id, которую можно обойти через фриду

----------

```shell
[0x00004000]> izz~SecItemAdd

\56462  0x00aa7f0e 0x00aa7f0e 11   12                              ascii   _SecItemAdd

70885  0x00db0afb 0x00db0afb 11   12                              ascii   _SecItemAdd
```

вижу _SecItemAdd - это системаня функция ios которая сохраняет в **Keychain** - значит - в коде используется это 

----------

```shell
[0x00004000]> izz~SecItemCopyMatching

56463  0x00aa7f1a 0x00aa7f1a 20   21                              ascii   _SecItemCopyMatching

70886  0x00db0b07 0x00db0b07 20   21                              ascii   _SecItemCopyMatching
```
вижу _SecItemCopyMatching - это уже функция системная - чтение из Keychain

-------

```shell
[0x00004000]> izz~SecAccessControlCreateWithFlags
[0x00004000]> izz~SecAccessControl
[0x00004000]>
```
не найден SecAccessControlCreateWithFlags  и  SecAccessControl - значит - что данные которые сохраняются в кейчан - не оборачиваются в Face-id (без флага биометрии)

```q
[0x00004000]> axt 0x00921f6c
sym.MeetWay.KeychainManager.saveKey.allocator_...SbSSFZ_ 0x39f2ec [CALL:--x] bl sym.imp.SecItemAdd

sym.MeetWay.KeychainHelper.save.allocator.for...SbSS_SStF 0x3a144c [CALL:--x] bl sym.imp.SecItemAdd
```

```q
[0x00004000]> ii~SecItem
1685 0x00921f6c NONE FUNC               SecItemAdd
1686 0x00921f78 NONE FUNC               SecItemCopyMatching
1687 0x00921f84 NONE FUNC               SecItemDelete
1688 0x00921f90 NONE FUNC               SecItemUpdate

[0x00004000]> / LocalAuthentication
0x00001d13 hit5_0 .rary/Frameworks/LocalAuthentication.framework/Local.
0x00001d31 hit5_1 .ation.framework/LocalAuthenticationX.

[0x00004000]> / LAContext
0x00ac4f38 hit6_0 .t_OBJC_CLASS_$_LAContext_OBJC_CLASS_$_F.
0x00dbc19c hit6_1 .p_OBJC_CLASS_$_LAContext_OBJC_CLASS_$_M.
0x018232af hit6_2 .4ChatVWOb_$sSo9LAContextCABycfC_$s7Meet.
0x0182393b hit6_3 .ageCSgGvs_$sSo9LAContextCMa_$sSo7NSErro.
0x01826647 hit6_4 .hatRowVMr_$sSo9LAContextCABycfcTO_$s7Me.
0x0263301c hit6_5 .ierR_rlWL_$sSo9LAContextCML_$sSo12FIRTi.

[0x00004000]> / SecItem
0x00ab3f0f hit7_0 .SubjectSummary_SecItemAdd_SecItemCopy.
0x00ab3f1b hit7_1 .ry_SecItemAdd_SecItemCopyMatching_Se.
0x00ab3f30 hit7_2 .emCopyMatching_SecItemDelete_SecItemU.
0x00ab3f3f hit7_3 ._SecItemDelete_SecItemUpdate_SecKeyCo.
0x00dbcafc hit7_4 .SubjectSummary_SecItemAdd_SecItemCopy.
0x00dbcb08 hit7_5 .ry_SecItemAdd_SecItemCopyMatching_Se.
0x00dbcb1d hit7_6 .emCopyMatching_SecItemDelete_SecItemU.
0x00dbcb2c hit7_7 ._SecItemDelete_SecItemUpdate_SecKeyCo.
0x00aa7f0f hit7_8 .SubjectSummary_SecItemAdd_SecItemCopy.
0x00aa7f1b hit7_9 .ry_SecItemAdd_SecItemCopyMatching_Se.
0x00aa7f30 hit7_10 .emCopyMatching_SecItemDelete_SecItemU.
0x00aa7f3f hit7_11 ._SecItemDelete_SecItemUpdate_SecKeyCo.
[0x00004000]> / SecAccess
[0x00004000]>
```

--------

# вывод по поиску: (предварительно)

1 - в коде где-то вызываются ручные проверки face-id
2 - в коде используется Keychain

--------


теперь нужно узнать подробнее про это все
нужно понять, где и для чего вызывается FACE-id и где и для чего используется Keychain

---------

проверяю кейчейн
```q
[0x00004000]> axt 0x00921f6c
sym.MeetWay.KeychainManager.saveKey.allocator_...SbSSFZ_ 0x39f2ec [CALL:--x] bl sym.imp.SecItemAdd

sym.MeetWay.KeychainHelper.save.allocator.for...SbSS_SStF 0x3a144c [CALL:--x] bl sym.imp.SecItemAdd
```

вижу два обьекта  KeychainManager и  KeychainHelper
гляну подробнее их дизсасемб код
```c
[0x00004000]> 0x39f2ec
[0x0039f2ec]> pdf

там 2700 строк дизасемблера, 
```
вижу тут две функции - сохраняет ключ и удаляет его

0x0039f2ec      200b1694       bl sym.imp.SecItemAdd
  0x0039f28c      3e0b1694       bl sym.imp.SecItemDelete


![[Снимок экрана 2026-04-30 в 13.48.40.png]]

также вижу вызов другой функции
```q
0x0039f044      29ffff97       bl sym.MeetWay.KeychainManager.keyAccount.allocator__Swift.String__String:_allocator__keyAccount__String:_allocatorS.au ; func.0039ece8


│       │   0x0039efb4      1bffff97       bl sym.MeetWay.KeychainManager.service.allocator__Swift.String__String:_allocator__service__String:_allocatorS.au ; func.0039ec20
```

здесь нет нигде  SecAccessControlCreateWithFlags  и  SecAccessControl - значит - что данные которые сохраняются в кейчан - не оборачиваются в Face-id (без флага биометрии)

но это пока что ни очем особо не говорит, может там фигня какая-то сохраняется в кейчейн

--------

хочу глянуть, кто вызывает эти две функции  KeychainManager и  KeychainManager!

```q
[0x00004000]> / Keychain
0x0094ac99 hit7_84 .  Keychain, .
0x0094afdd hit7_85 .  Keychain J.
0x0095032d hit7_86 . JWT  Keychain J.
```
что-то интересное с jwt
```q
[0x00004000]> 0x0095032d
[0x0095032d]> pd

 ; STRN XREF from func.003f71ac @ 0x3f721c(r)
            0x00950400     .string "Failed to save JWT" ; len=19
            
смотрю
[0x0095032d]> axt 0x00950400
sym.MeetWay.YandexDBManager.exchange.allocator.TokenForJWTWithAction._7108AB7FE2C72FC07BC02864048A0FEF__String__5 0x3e735c [STRN:r--] add x0, x0, str.Failed_to_save_JWT

sym.MeetWay.YandexDBManager.exchange.allocator.TokenForJWT._7108AB7FE2C72FC07BC02864048A0FEF__String__5 0x3f721c [STRN:r--] add x0, x0, str.Failed_to_save_JWT

=---------------

[0x0095032d]> axt 0x00959e70
sym.MeetWay.ViewModel.saveLastMessageTimestamps.allocator_...yF_ 0x6b5f10 [STRN:r--] add x0, x0, str.lastMessageTimestamps

-----------

sym.MeetWay.FirstScrin.body.SwiftUI.TupleView...E0I0PAEE16allowsHitTestingyQrSbFQOyAiEE15ignoresSafeArea_5edgesQrAE0nO7RegionsV_AE4EdgeO3SetVtFQOyAE5ColorV_Qo__Qo__AA15LoadingOverlay2VtGSg_AiEE7overlay_9alignmentQrqd___AE9AlignmentVtAeHRd__lFQOyAiEE8onAppear7performQryycSg_tFQOyAiEE7paddingyQrAR_12CoreGraphics7CGFloatVSgtFQOyAiEEAK_ALQrAN_ARtFQOyAE6VStackVyAGyAiEE5frame5width6heightA0_QrA10__A10_A2_tFQOyAE6ZStackVyAGyA17_yAGyAiEEA3_A4_QrA5__tFQOyAiEEA6_yQrAR_A10_tFQOyA17_yAE19_ConditionalContentVyAiEE15fullSc__159 0x73a524

```

КОРОЧЕ ГОВОРЯ - КАК МИНИМУМ - ВИДНО, что есть код который сохраняет в кейчейн JWT в классах FirstScrin /  ViewModel /  YandexDBManager
по сути - для этого - не требуется face-id ! так что - то что я нашел - нормально!!!

----------




# смотрю - где используется face-id


```q
[0x00004000]> izz~evaluatePolicy
54400  0x00a46cba 0x00a46cba 37   38   8.__TEXT.__objc_methname   ascii   evaluatePolicy:localizedReason:reply:


[0x00004000]> 0x00a46cba

[0x00a46cba]> axt 0x00a46cba

(nofunc) 0xa909b0 [DATA:r--] invalid

[0x00a46cba]> axt 0xa909b0

sym.MeetWay.listAllMyChats.authenticateUser.completion_...F_ 0x599f0c [DATA:r--] ldr x1, reloc.fixup.evaluatePolicy:localizedReason:

[0x00a46cba]> 0x599f0c
[0x00599f0c]> pdf

ВИЖУ - ЧТО ВЫЗЫВАЕТСЯ МЕТОД В ОБЬЕКТЕ listAllMyChats
ТЕПЕРЬ ГЛЯНУ ЕГО КОД
```

вот это место где вызывается evaluatePolicy
```q
   0x00599f08      a82700f0       adrp x8, reloc.fixup.setBackgroundSession: ; 0xa90000
   
   
│       │   0x00599f0c      01d944f9       ldr x1, [x8, 0x9b0]         ; [0xa909b0:4]=0xa46cba str.evaluatePolicy:localizedReason:reply: ; reloc.fixup.evaluatePolicy:localizedReason: ; char *selector


```

![[Снимок экрана 2026-04-30 в 15.01.59.png]]


```c
# Посмотреть, в какой функции находится эта инструкция
[0x00004000]> s 0x00599f0c
[0x0000599f0c]> pdf
# Узнать имя функции
[0x0000599f0c]> f~0x599f0c
# Посмотреть, кто вызывает ЭТУ функцию
[0x0000599f0c]> axt 0x00599f00  # чуть выше

```

вижу вот что 
в классе listAllMyChats вызывается метод  authenticateUser
и в самом методе authenticateUser используется  evaluatePolicy (вызов FACE-id)

` MeetWay.listAllMyChats.authenticateUser.completion(...F)`

```
  ; [0xa909b0:4]=0xa46cba str.evaluatePolicy:localizedReason:reply: ; reloc.fixup.evaluatePolicy:localizedReason: ; char *selector
```
  
![[Снимок экрана 2026-04-30 в 15.09.46.png]]

короче говоря, что функция используется , судя по имени класса listAllMyChats  - это класс/структура/вьюха - в которой используется face-id - например - для доступа к чатам!
я полагаю - что если это не какие-то секретные чаты - то этого обычного , ручного face-id будет достаточно!
но согласно тесту, чтобы обеспечить безопасность - нужно - при входе в приложение - при обновлении JWT токена - сохранять флаг в кейчан "мол - авторизация успешна" и потом - face id настроить через Keychain, тогда вызов проверки фейса - будет происходить изнутри системы в Keychain! 
и фридой уже не получится так легко обойти проверку!



----------------

### Итоговое заключение

Приложение использует Keychain для хранения чувствительных данных (JWT токенов), что является правильным подходом.  Однако:

Критическое нарушение: данные сохраняются в Keychain **без флага биометрии**, что делает возможным их извлечение без Face ID
   
Уязвимость: Ручная проверка Face ID через `LAContext.evaluatePolicy` может быть обойдена с помощью Frida , но,  я посмотрел, в контексте моего приложения - это не опасно, но с точки зрения безопастности - лучше сделать проверку Face ID через Keychain! 
вот так
```swift
let token = getChatsTokenFromKeychain()  // ← Keychain сам запросит Face ID
showChats()
```
   
Рекомендация: Все чувствительные данные (JWT токены) должны сохраняться в Keychain с флагом `kSecAccessControlBiometryCurrentSet`, а ручная проверка Face ID должна быть полностью заменена на механизм, встроенный в Keychain


----------

динамическое продолжение теста 
здесь [[_L2__DAST_Keychain+Face-id__access-control]]
