# MASTG-TEST-0311: небезопасное случайное использование API

смысл в том, чтобы найти в коде "самописные" или устаревшие способы 
генерирования рандомных чисел!

также обычных рандом функций 
`rand()`, `random()`, `srand()`
`erand48`, `lrand48`, `mrand48`
`arc4random`



проблема их в том, что например для функции rand() - зная предыдущее значение, можно вычислить следующее!

безопасные это
✅ `SecRandomCopyBytes`
✅ `CryptoKit`

---------

тестировать можно очень просто

через радар2 например

```q
# 1. Импорты небезопасных рандомов (C library)
is | grep -E "rand|random|srand|rand_r"

# 2. Импорты семейства *rand48
is | grep -E "drand48|erand48|lrand48|mrand48|nrand48|jrand48|srand48"

# 3. Импорты безопасных рандомов Apple
is | grep -E "SecRandomCopyBytes|SecRandom"

# 4. Поиск вызовов rand в коде (ассемблерные паттерны)
/ rand
/ random
/ srand

# 5. Поиск вызовов *rand48
/ drand48
/ lrand48

# 6. Поиск безопасных вызовов
/ SecRandomCopyBytes

# 7. Поиск CryptoKit (там генерация ключей безопасная)
/ CryptoKit

# 8. Поиск arc4random (в iOS считается безопасным, но проверим)
/ arc4random

# 9. Поиск GameKit (может использоваться для случайностей в играх)
/ GKRandomSource
```

-----

тест буду проводить на приложении - [[0_MeetWay]]


результаты выдачи радара по вышеуказанным командам 

```q
не найдено устаревших или слабых рандом функций

также - найдено 3 совпадения - `SecRandomCopyBytes
и это очень хорошо, это безопасно!


а что касается вот этого найденного -  `sSb6randomSbyFZ`

вот, я его проверил 

[0x00004000]> 0x009215f0

[0x009215f0]> axt

sym.MeetWay.AudioVisualizerView.updateAnimationValues._58B9898145204CA89E0CDD64611B323B.with_...F_ 0x659a30 [CALL:--x] bl sym.imp.random_...byFZ_

[0x009215f0]> pdf

            ; CALL XREF from func.006597d0 @ 0x659a30(x) ; sym.MeetWay.AudioVisualizerView.updateAnimationValues._58B9898145204CA89E0CDD64611B323B.with_...F_
┌ 12: sym.imp.random_...byFZ_ ();
│           0x009215f0      900a00b0       adrp x16, reloc.SwiftUI.Button.action.label_...yXEtcfC_ ; 0xa72000 ; random(...byFZ)
│           0x009215f4      102e44f9       ldr x16, [x16, 0x858]       ; [0xa72858:4]=0x4cc "jc_methname" ; reloc.random_...byFZ_
└           0x009215f8      00021fd6       br x16
[0x009215f0]>

это - системный вызов  `AudioVisualizerView.updateAnimationValues
анимация отображания И так далее итп .. фигня в общем, к безопасности не имеет отношения

```

---

#### тест пройден

**Результаты:**
- Небезопасные `srand`, `drand48`, `lrand48` — не обнаружены
- Обнаружен Swift `Bool.random()` — используется в некриптографическом контексте
- Обнаружен `SecRandomCopyBytes` — безопасный криптографический генератор
- Обнаружен CryptoKit — безопасная криптография

==Вывод: Небезопасные генераторы случайных чисел не используются в критических контекстах==




