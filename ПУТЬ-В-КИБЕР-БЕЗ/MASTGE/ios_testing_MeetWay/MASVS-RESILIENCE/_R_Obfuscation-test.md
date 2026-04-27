# MASTG-TEST-0093: Testing Obfuscation

тест проверяет, насколько код приложения запутан (обфусцирован), чтобы затруднить реверс-инжиниринг

-------

### Что обязательно должго быть обфусцировано::

**Имена классов**
**Имена методов**
**Строки с URL**
**API-ключи** (если они вообще там есть .....)
**Строки с инфой важной**
**Криптографические хеши**
**Названия библиотек**

----------
обфускацию обычно выполняют перед отправкой сборки в аппстор
для дебажа будущего - следует использовать  mapping-файл - который программы-обфускаторы создают 
и выглядят типо так:
GRPCInterceptor → x7K

-----------

можно через радар проверить имена всех функций, и кроме системных , все функции должны быть обфусцированны
```q
r2 MeetWay.debug.dylib
aaaa
afl~


или все строки
izz~

во всех случаях будет оч много данных!

также проверить на энтропию , должна быть высокая!
iS~entropy


```

----

### СПОСОБЫ обфускации:

классы вот так можно 
```swift
class a {  
    func b(c d: String) -> Bool  
    func e()  
}

```

-------

строки/хеши вот так шифровать
-XOR-шифрование-
```swift
let url = "https://api.yandex.ru" // было

// стало
let url = decrypt([0x8D, 0x9A, 0x8D, 0x9B, 0x4F, 0x4F, 0x8C, 0x98, 0x91, 0x4F, 0x99, 0x8C, 0x97, 0x8D, 0x8F, 0x9B, 0x4F, 0x9B, 0x9E], key: 0xFF)

-------------
А ВОТ И САМА ФУНКЦИЯ КОТОРАЯ ЗАШИФРУЕТ ЗНАЧЕНИЯ-

func encrypt(_ string: String, key: UInt8 = 0xAB) -> [UInt8] {
    return string.utf8.map { $0 ^ key }
}

func decrypt(_ bytes: [UInt8], key: UInt8 = 0xAB) -> String {
    let decoded = bytes.map { $0 ^ key }
    return String(decoding: decoded, as: UTF8.self)
}

```
или использовать Swift-макрос для автоматического шифрования

или разбить на части

части / куски строк можно хранить в разных местах кода, файлах/ plist , также в кейчейн или юзердефолтс

-------

можно путать логику

```swift
//было
if checkJailbreak() {
    exit(0)
}

//стало (тоже самое)
let x = random() % 10
switch x {
case 0: fallthrough
case 1: if evaluateSecurity() != 42 { fallthrough }
case 2: let _ = performFakeCheck()
case 3: if subtleCheck() { handleViolation() }
case 4...9: break
default: normalFlow()
}

```
можно добавлять ложные строки, ложную логику, разбивать методы на несколько

-------

также есть способы, которыми можно обфусцировать весь код сразу
упаковка/шифрование бинарника
```c
# коммерческие инструменты:
- iXGuard (Guardsquare)
- Arxan
- Appdome

# Open-source альтернативы:
- SwiftShield  0- имена классов и методов на случайный набор букв
-Swift Confidential - для строк
- O-MVLL - мощная штука - меняет логику выполнения кода


```


--------


#### тест буду выполнять на приложении 👉 [[0_MeetWay]] (рабочая соц-сеть)

на данный момент в коде приложения редко где используется обфускация

поэтому, в некоторые места я добавлю несколько вариантов обфускации

и смогу через SAST показать, как получилось защитить те или иные места в коде!

-----------



### план
1) анализ бинарника до обфускации
2) делалаю обфускацию кода приложения
3) повторный анализ бинарника 
4) выводы

---------

### шаг 1  (анализ бинарника до обфускации)

проверю главный бинарник на энтропию

`[0x00004000]> iS entropy`

видно, что максимальное значение 6.48 (ок считается ближе к 8 и выше)
что говорит об обычном коде, который не обфусцирован
и в целом средний показатель наверно в районе ~ 4.6

![[Снимок экрана 2026-04-27 в 21.23.04.png]]

расшифровка файлов (дипсик все раскидал по своим местам)
```q

СЕКЦИЯ                                   ENTROPY   ЧТО СОДЕРЖИТ
------------------------------------------------------------------

0.__TEXT.__text                           6.48     Исполняемый код (функции, методы, вся логика)🔥
1.__TEXT.__stubs                          4.31     Заглушки для вызова внешних функций
2.__TEXT.__objc_stubs                     4.03     Заглушки для Objective-C методов
3.__TEXT.__init_offsets                   2.00     Смещения для инициализации
4.__TEXT.__objc_methlist                  4.86     Списки Objective-C методов
5.__TEXT.__swift5_typeref                 5.21     Ссылки на Swift-типы
6.__TEXT.__swift5_capture                 3.38     Захваченные переменные в Swift-замыканиях
7.__TEXT.__cstring                        5.19     C-строки (текстовые строки в коде) *🔥
8.__TEXT.__objc_methname                  4.88     Имена Objective-C методов (читаемые!) *🔥
9.__TEXT.__const                          5.77     Константные данные
10.__TEXT.__swift5_reflstr                4.46     Swift-строки-отражения
11.__TEXT.__swift5_assocty                4.95     Swift associated types
12.__TEXT.__constg_swiftt                 4.83     Swift-константы
13.__TEXT.__swift5_fieldmd                4.43     Метаданные полей Swift
14.__TEXT.__swift5_builtin                2.71     Встроенные Swift-типы
15.__TEXT.__swift5_proto                  4.95     Протоколы Swift
16.__TEXT.__swift5_types                  4.24     Типы Swift
17.__TEXT.__swift_as_entry                5.18     Swift точки входа
18.__TEXT.__swift_as_ret                  5.55     Swift возвраты
19.__TEXT.__swift5_entry                  2.41     Swift entry point
20.__TEXT.__objc_classname                4.97     Имена Objective-C классов (читаемые!) *🔥
21.__TEXT.__objc_methtype                 5.33     Сигнатуры методов Objective-C
22.__TEXT.__gcc_except_tab                4.76     Таблицы исключений
23.__TEXT.__unwind_info                   6.34     Информация для размотки стека
24.__TEXT.__eh_frame                      3.84     Exception Handling frame
25.__DATA_CONST.__got                     3.23     Global Offset Table (указатели)
26.__DATA_CONST.__const                   2.94     Константы в data-сегменте
27.__DATA_CONST.__cfstring                2.56     CFString (Core Foundation строки)
28.__DATA_CONST.__objc_classlist          3.10     Список ObjC классов
29.__DATA_CONST.__objc_protolist          2.91     Список ObjC протоколов
30.__DATA_CONST.__objc_imageinfo          2.00     Информация об образе
31.__DATA_CONST.__objc_protorefs          2.81     Ссылки на ObjC протоколы
32.__DATA_CONST.__objc_classrefs          2.98     Ссылки на ObjC классы
33.__DATA_CONST.__objc_superrefs          2.37     Ссылки на суперклассы
34.__DATA.__objc_const                    2.86     ObjC константы
35.__DATA.__objc_selrefs                  3.61     Ссылки на селекторы
36.__DATA.__objc_ivar                     1.67     ObjC instance variables
37.__DATA.__objc_data                     3.06     ObjC данные
38.__DATA.__data                          3.68     Глобальные переменные, данные
39.__DATA.__bss                           0.00     Неинициализированные данные
40.__DATA.__common                        0.00     Общие данные

====================================================================================================
* ЗВЕЗДОЧКОЙ ОТМЕЧЕНЫ КРИТИЧЕСКИЕ СЕКЦИИ ДЛЯ ТЕСТА MASTG-TEST-0093
====================================================================================================

СРЕДНЯЯ ЭНТРОПИЯ: ~4.64 из 8.00 (НИЗКИЙ ПОКАЗАТЕЛЬ)

КЛЮЧЕВЫЕ ПОКАЗАТЕЛИ ДЛЯ ОБФУСКАЦИИ:
- Имена классов (20):  4.97 из 8 — ИМЕНА ЧИТАЕМЫ
- Имена методов (8):   4.88 из 8 — ИМЕНА ЧИТАЕМЫ
- C-строки (7):        5.19 из 8 — СТРОКИ НЕ ЗАШИФРОВАНЫ
- Исполняемый код (0): 6.48 из 8 — КОД НЕ ЗАШИФРОВАН

ПОСЛЕ ОБФУСКАЦИИ ДОЛЖНО БЫТЬ:
- Имена классов: 6.5+ (после переименования в a, b, c)
- Имена методов: 6.5+ (после переименования)
- Строки: 7.0+ (после XOR-шифрования)
- Код: 7.0+ (после упаковки/шифрования)

РЕЗУЛЬТАТ: ТЕСТ MASTG-TEST-0093 НЕ ПРОЙДЕН — ОБФУСКАЦИЯ ОТСУТСТВУЕТ
```

гляну все имена классов/методов
```q
afl~
```

и как видно, здесь есть  32 совпадения имени  `TrustKit`
которое используется в функции для SLL Pinning (важная функция)

```
0x00919280    1     28 method.TrustKit.pinningValidatorCallback
0x0091929c    1     56 method.TrustKit.setPinningValidatorCallback:
0x009192d4    1     28 method.TrustKit.pinningValidatorCallbackQueue
0x009192f0    1     28 method.TrustKit.pinFailureReporter
0x0091930c    1     52 method.TrustKit.setPinFailureReporter:
0x00919340    1     28 method.TrustKit.pinFailureReporterQueue
0x0091935c    1     52 method.TrustKit.setPinFailureReporterQueue:
0x00919390    1     28 method.setupTrustKitWithGRPC
```

![[Снимок экрана 2026-04-27 в 21.36.20.png]]

также - гляну быстро все строки с высокой энтропией - ищу токены всякие/хеши

```python
strings MeetWay.debug.dylib | python3 -c "
import sys, math
for line in sys.stdin:
    s = line.strip()
    if len(s) > 20:
        freq = {}
        for c in s:
            freq[c] = freq.get(c, 0) + 1
        entropy = -sum((freq[c]/len(s))*math.log2(freq[c]/len(s)) for c in freq)
        if entropy > 4.5:
            print(f'[ENTROPY {entropy:.2f}] LEN={len(s)}: {s[:100]}')
"

(можно поиграться длинной и стпенью)
```

найдено множество значений (так как приложение мое - я уже знаю - что это и вшитые ключи, и хеши токенов, и соль шифрования)

![[Снимок экрана 2026-04-27 в 21.51.37.png]]



----------

### шаг 2 выполню обфускацию кода 
(в целях экномии времени и возможности последующего обучения - я настрою обускацию для функций SLL Pinning )

затем - я выполню точно такие же тесты и сравню результаты (которые очевидны)

для наглядности - я обфусцирую все - что связано с этой функцией TrustKit - которая настраивает SLL Pinning

вот часть кода - функционала отвечающего за SLL Pinning

(функция setupTrustKitWithGRPC - инициализируется  в didFinishLaunchingWithOptions)

(здесь и сама функция говорит сама за себя + вшиты хеши публичных ключей сертификатов - тоже не гуд)

![[Снимок экрана 2026-04-27 в 22.02.23.png]]


имя метода setupTrustKitWithGRPC меняю на f0o

далее кодирую хеши
```q
func encrypt(_ string: String, key: UInt8 = 0xAB) -> [UInt8] {
    return string.utf8.map { $0 ^ key }
}

--------
2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4= 
превращается в
[153, 202, 238, 252, 209, 229, 249, 197, 225, 193, 250, 202, 204, 255, 254, 218, 210, 194, 237, 198, 242, 206, 233, 248, 154, 234, 237, 128, 146, 219, 197, 217, 250, 221, 223, 155, 229, 152, 207, 157, 238, 249, 159, 150]

а чтобы обратно декодировать

func decrypt(_ bytes: [UInt8], key: UInt8 = 0xAB) -> String {
    let decoded = bytes.map { $0 ^ key }
    return String(decoding: decoded, as: UTF8.self)
}

let aa = decrypt([153, 202, 238, 252, 209, 229, 249, 197, 225, 193, 250, 202, 204, 255, 254, 218, 210, 194, 237, 198, 242, 206, 233, 248, 154, 234, 237, 128, 146, 219, 197, 217, 250, 221, 223, 155, 229, 152, 207, 157, 238, 249, 159, 150])
print(aa)

получаю обратно 
2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4=

ТАКИМ ОБРАЗОМ МОЖНО ЗАШИФРОВАТЬ ХЕШИ И ВШИТЬ ИХ В КОД (ЕСЛИ ЭТО РАЗУМНО), САМИ ХЕШИ МОЖНО ПОМЕСТИТЬ НАПРИМЕР В КЕЙЧЕЙН И БРАТЬ ИХ ОТ ТУДА И ДЕШИФРОВАТЬ ФУНКЦИЕЙ decrypt и имя самой функции тоже обфусцировать

```



и вот так преобразилась функция

![[Снимок экрана 2026-04-27 в 22.40.57.png]]


БЫЛО
```swift
 private func setupTrustKitWithGRPC() {

            let trustKitConfig: [String: Any] = [

                kTSKSwizzleNetworkDelegates: true,
                kTSKPinnedDomains: [
                    "api.yandex.ru": [
                        kTSKPublicKeyHashes: [
	"2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4=",
	"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
],
		TSKIncludeSubdomains: true,
		kTSKEnforcePinning: true
],
						"storage.yandexcloud.net": [
kTSKPublicKeyHashes: [
"CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=",
"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
],
kTSKIncludeSubdomains: **true**,
kTSKEnforcePinning: **true**
],
						"functions.yandexcloud.net": [
kTSKPublicKeyHashes: [
"Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=",
"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
],
kTSKIncludeSubdomains: **true**,
kTSKEnforcePinning: **true**
], и так далее
```
СТАЛО
```swift  
private func f0o() {
let trustKitConfig: [String: Any] = [
kTSKSwizzleNetworkDelegates: true,
                kTSKPinnedDomains: [
                    self.drt(u01): [
                        kTSKPublicKeyHashes: [
                            self.drt(g001),
                            self.drt(g000)
                        ],
                        kTSKIncludeSubdomains: true,
                        kTSKEnforcePinning: true
                    ],
                    self.drt(u02): [
                        kTSKPublicKeyHashes: [
                            self.drt(g002),
                            self.drt(g000)
                        ],
                        kTSKIncludeSubdomains: true,
                        kTSKEnforcePinning: true
                    ],
                    self.drt(u03): [
                        kTSKPublicKeyHashes: [
                            self.drt(g003),
                            self.drt(g000)
                        ],
                        kTSKIncludeSubdomains: true,
                        kTSKEnforcePinning: true
                    ], и так далее
```

теперь имя функци просто так СХОДУ статическим анализом не найти и не понять, чем она занимается
++ ко всему - хеши паблик ключей дополнительно спрятаны 
имена констант и имя функции обфусцировано
сами константы - можно убрать в кейчейн и будет красота!

-------

### шаг 3 ( собираю сборку новую и запускаю анализ повторно)


проверю главный бинарник на энтропию

`[0x00004000]> iS entropy`

там порядка 4000 функций, я поменял пару мест, поэтому - в результате это не отразилось


гляну все имена классов/методов
```q
afl~
```

отлично!! фунции `setupTrustKitWithGRPC` - вы выдаче НЕТ!
(я капитан очевидность)



также - гляну быстро все строки с высокой энтропией - ищу токены всякие/хеши

```python
strings MeetWay.debug.dylib | python3 -c "
import sys, math
for line in sys.stdin:
    s = line.strip()
    if len(s) > 20:
        freq = {}
        for c in s:
            freq[c] = freq.get(c, 0) + 1
        entropy = -sum((freq[c]/len(s))*math.log2(freq[c]/len(s)) for c in freq)
        if entropy > 4.5:
            print(f'[ENTROPY {entropy:.2f}] LEN={len(s)}: {s[:100]}')
"

(можно поиграться длинной и стпенью)
```


в прошлый раз в выдаче были:
```
"2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4=",
"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
"CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=",
"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
"Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=",
"C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
```

теперь в выдаче - данных значений нет!
это существенно должно усложнить SAST, для чего собственно, и нужна обфускация 

![[Снимок экрана 2026-04-27 в 23.06.40.png]]

------
### шаг 4 ВЫВОД

Этот тест, и его вывод, были очевидны самого начала, но я на практике доказал сам себе, так очередную вещь, что обускация кода реально помогает защищаться от Reverse инжиниринга!

Далее можно использовать какие-либо инструменты для массовой обфускации всего кода (например SwiftShield), также значения, которые закодированы можно разместить в keychain


