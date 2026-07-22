https://mas.owasp.org/MASTG/tests/android/MASVS-PRIVACY/MASTG-TEST-0319/
# MASTG-TEST-0319: Runtime Use of SDK APIs Known to Handle Sensitive User Data


после того , как я узнал , какие зависимости стоят в приложении, нужно через фриду хукнуть эти методы, если они есть и проверить, какие данные там передаются!

скрипт для фрида, образец

```q
// hook_sdk.js – перехват Firebase & Yandex SDK
// Запуск: frida -U -l hook_sdk.js -f com.evgeniy.meetway

console.log("[*] SDK Hooking Script Loaded");

// ==========================
// 1. FIREBASE ANALYTICS
// ==========================
var FirebaseAnalytics = Java.use("com.google.firebase.analytics.FirebaseAnalytics");

// Перехват setUserId (установка ID пользователя)
FirebaseAnalytics.setUserId.overload("java.lang.String").implementation = function(userId) {
    console.log("[🔥 Firebase] setUserId() called");
    console.log("[📤] User ID: " + userId);
    console.log("[📎] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
    return this.setUserId(userId);
};

// Перехват setUserProperty (установка свойства пользователя)
FirebaseAnalytics.setUserProperty.overload("java.lang.String", "java.lang.String").implementation = function(name, value) {
    console.log("[🔥 Firebase] setUserProperty() called");
    console.log("[📤] Property Name: " + name);
    console.log("[📤] Property Value: " + value);
    console.log("[📎] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
    return this.setUserProperty(name, value);
};

// Перехват logEvent (логирование событий)
FirebaseAnalytics.logEvent.overload("java.lang.String", "android.os.Bundle").implementation = function(eventName, params) {
    console.log("[🔥 Firebase] logEvent() called");
    console.log("[📤] Event Name: " + eventName);
    if (params != null) {
        var keys = params.keySet().toArray();
        for (var i = 0; i < keys.length; i++) {
            var key = keys[i];
            var value = params.get(key);
            console.log("   - " + key + ": " + value);
        }
    }
    console.log("[📎] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
    return this.logEvent(eventName, params);
};

// ==========================
// 2. YANDEX APPMETRICA
// ==========================
try {
    var YandexMetrica = Java.use("com.yandex.metrica.YandexMetrica");
    
    // Перехват reportEvent (отправка события)
    YandexMetrica.reportEvent.overload("java.lang.String").implementation = function(eventName) {
        console.log("[📊 Yandex] reportEvent() called");
        console.log("[📤] Event: " + eventName);
        console.log("[📎] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
        return this.reportEvent(eventName);
    };
    
    // Перехват reportEvent с параметрами
    YandexMetrica.reportEvent.overload("java.lang.String", "java.util.Map").implementation = function(eventName, params) {
        console.log("[📊 Yandex] reportEvent() with params called");
        console.log("[📤] Event: " + eventName);
        if (params != null) {
            var keys = params.keySet().toArray();
            for (var i = 0; i < keys.length; i++) {
                var key = keys[i];
                var value = params.get(key);
                console.log("   - " + key + ": " + value);
            }
        }
        console.log("[📎] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
        return this.reportEvent(eventName, params);
    };
    
    console.log("[*] YandexMetrica hooked successfully!");
} catch (e) {
    console.log("[⚠️] YandexMetrica not found in this app.");
}

// ==========================
// 3. ДОПОЛНИТЕЛЬНО: Перехват всех методов с PII
// ==========================
// Это перехватывает все вызовы, которые могут содержать email, phone, name
// Работает для любых классов, содержащих такие методы

Java.perform(function() {
    var classes = Java.enumerateLoadedClassesSync();
    var targets = ["email", "phone", "password", "name", "surname", "address"];
    
    for (var i = 0; i < classes.length; i++) {
        var className = classes[i];
        // Ищем классы, которые могут содержать сеттеры для PII
        if (className.toLowerCase().indexOf("user") !== -1 || 
            className.toLowerCase().indexOf("profile") !== -1 ||
            className.toLowerCase().indexOf("auth") !== -1 ||
            className.toLowerCase().indexOf("account") !== -1) {
            // Можно добавить логику, но осторожно, чтобы не загрузить систему
        }
    }
});

console.log("[*] All hooks installed. Ready to intercept PII!");

// Функция для логирования стека вызовов
function logStack() {
    return Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new());
}
```