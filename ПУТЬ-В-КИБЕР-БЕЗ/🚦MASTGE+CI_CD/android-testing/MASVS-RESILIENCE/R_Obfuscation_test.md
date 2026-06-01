# MASTG-TEST-0051: Testing Obfuscation

Тест проверяет, насколько разработчики запутали свой код, чтобы ( злоумышленник ) не мог быстро понять, как работает приложение. Это базовая защита от реверс-инжиниринга - первая линия обороны перед анти-отладкой и проверкой целостности

**Обфускация бывает нескольких уровней**

**Базовый (ProGuard/R8)**     - Переименование классов, методов, полей в `a`, `b`, `c`

**Средний**  -  Шифрование строк - `"https://api.com/login"` → зашифрованная строка, расшифровка в рантайме

**Продвинутый** -  Контроль потока (control flow flattening) - Логика превращается в огромный switch-case, где порядок выполнения неочевиден

**Экстрим (native)** - Syscalls вместо libc, Obfuscator-LLVM - Нативный код с фальшивыми ветвлениями и flattened control flow

---------

пример

```c
// Было (понятно)
public void validateUserCredentials(String username, String password) {
    if (isValidUsername(username) && isCorrectPassword(password)) {
        grantAccess();
    } else {
        denyAccess();
    }
}

// Стало (после обфускации)
public void a(String b, String c) {
    if (d(b) && e(c)) {
        f();
    } else {
        g();
    }
}
```

для банков - обязательная процедура

-----------------























