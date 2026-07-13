# MASTG-TEST-0350: Runtime Use of Broken Symmetric Encryption Modes
https://mas.owasp.org/MASTG/tests/android/MASVS-CRYPTO/MASTG-TEST-0350/

Тест проверяет в рантайме, не использует ли приложение небезопасные режимы шифрования (например, **ECB**) для защиты чувствительных данных. Даже если в коде есть вызов `Cipher.getInstance("AES/ECB/PKCS5Padding")`, но он никогда не вызывается для реальных данных, это не проблема. Тест же проверяет, вызывается ли он в реальных сценариях

Тест динамический и использует Frida для перехвата вызовов `Cipher.getInstance()` и `Cipher.init()` в реальном времени


Тест будет провален, если обнаружено использование режима ECB для шифрования данных в реальном времени

---------


фрида везде стоит уже

```q
кидаю shell телефона

adb shell
su

запускаю сервер фриды на телефоне

cd /data/local/tmp
chmod 755 frida-server-17.9.1-android-arm64
nohup ./frida-server-17.9.1-android-arm64 &


НА МАКЕ В НОВОМ ТЕМИНАЛЕ 

// кидаю порт
adb forward tcp:27042 tcp:27042

смотрю запущенные процессы

frida-ps -U

```



```q
cat > /Users/evgeniy/Desktop/hook_cipher_modes.js << 'EOF'
// hook_cipher_modes.js - Перехват Cipher.getInstance и Cipher.init
Java.perform(function() {
    console.log("[*] === Cipher Mode Hook Script Started ===");

    // 1. Перехват Cipher.getInstance() — все варианты
    try {
        var Cipher = Java.use("javax.crypto.Cipher");

        // Cipher.getInstance(String transformation)
        Cipher.getInstance.overload('java.lang.String').implementation = function(transformation) {
            console.log("[+] Cipher.getInstance(String) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            // Проверяем, не используется ли небезопасный режим ECB
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation);
        };
        console.log("[+] Hooked: Cipher.getInstance(String)");

        // Cipher.getInstance(String transformation, String provider)
        Cipher.getInstance.overload('java.lang.String', 'java.lang.String').implementation = function(transformation, provider) {
            console.log("[+] Cipher.getInstance(String, String) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Provider: " + provider);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation, provider);
        };
        console.log("[+] Hooked: Cipher.getInstance(String, String)");

        // Cipher.getInstance(String transformation, Provider provider)
        Cipher.getInstance.overload('java.lang.String', 'java.security.Provider').implementation = function(transformation, provider) {
            console.log("[+] Cipher.getInstance(String, Provider) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Provider: " + provider.getName());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation, provider);
        };
        console.log("[+] Hooked: Cipher.getInstance(String, Provider)");

    } catch(e) {
        console.log("[-] Error hooking Cipher.getInstance: " + e);
    }

    // 2. Перехват Cipher.init() чтобы видеть, когда шифр используется
    try {
        var Cipher = Java.use("javax.crypto.Cipher");
        Cipher.init.overload('int', 'java.security.Key').implementation = function(opmode, key) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Key) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Key algorithm: " + key.getAlgorithm());
            // Получаем текущий алгоритм и режим у шифра (если он уже инициализирован)
            var cipher = this;
            // Попытка получить параметры шифра (если доступны)
            try {
                var params = cipher.getParameters();
                console.log("[+] Cipher parameters: " + params);
            } catch(e) {}
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key);
        };
        console.log("[+] Hooked: Cipher.init(Key)");

        Cipher.init.overload('int', 'java.security.Key', 'java.security.spec.AlgorithmParameterSpec').implementation = function(opmode, key, params) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Key, params) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Key algorithm: " + key.getAlgorithm());
            console.log("[+] Params: " + params);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key, params);
        };
        console.log("[+] Hooked: Cipher.init(Key, params)");
    } catch(e) {
        console.log("[-] Error hooking Cipher.init: " + e);
    }

    console.log("[*] === All Cipher hooks are set. Waiting for crypto operations... ===");
});
EOF
```


запуск скрита

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_cipher_modes.js
```




результаты работы скрипта:


```q
:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:640)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.encrypt(InsecureNonceAesGcmJce.java:93)
	at com.google.crypto.tink.subtle.AesGcmJce.encrypt(AesGcmJce.java:52)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.encrypt(AeadWrapper.java:72)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKeyValuePair(EncryptedSharedPreferences.java:641)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putEncryptedObject(EncryptedSharedPreferences.java:388)
	at androidx.security.crypto.EncryptedSharedPreferences$Editor.putString(EncryptedSharedPreferences.java:262)
	at com.evgeniy.meetway.data.local.SecureStorage.saveMyId(SecureStorage.kt:196)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.checkSavedSession(AuthViewModel.kt:73)
	at com.evgeniy.meetway.viewmodel.AuthViewModel.<init>(AuthViewModel.kt:57)
	at java.lang.reflect.Constructor.newInstance0(Native Method)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:343)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:295)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.android.kt:265)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:142)
	at androidx.lifecycle.SavedStateViewModelFactory.create(SavedStateViewModelFactory.android.kt:112)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl_androidKt.createViewModel(ViewModelProviderImpl.android.kt:35)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel(ViewModelProviderImpl.kt:59)
	at androidx.lifecycle.viewmodel.internal.ViewModelProviderImpl.getViewModel$lifecycle_viewmodel$default(ViewModelProviderImpl.kt:43)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.android.kt:90)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.get(ViewModel.kt:172)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.get(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt__ViewModelKt.viewModel(ViewModel.kt:106)
	at androidx.lifecycle.viewmodel.compose.ViewModelKt.viewModel(dex-id-f0643ac487072ca9d4489da5517f132e06a9895f:1)
	at com.evgeniy.meetway.ui.screens.auth.WelcomeScreenKt.WelcomeScreen(WelcomeScreen.kt:178)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$0(NavGraph.kt:42)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:212)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph(NavGraph.kt:36)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8$lambda$7(MainActivity.kt:147)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda1.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:126)
	at androidx.compose.material3.SurfaceKt$Surface$1.invoke(Surface.kt:108)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.SurfaceKt.Surface-T9BRK9s(Surface.kt:105)
	at com.evgeniy.meetway.MainActivity.MainContent$lambda$8(MainActivity.kt:146)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda2.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.material3.TextKt.ProvideTextStyle(Text.kt:349)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:69)
	at androidx.compose.material3.MaterialThemeKt$MaterialTheme$1.invoke(MaterialTheme.kt:68)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.material3.MaterialThemeKt.MaterialTheme(MaterialTheme.kt:60)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme$lambda$0(Theme.kt:42)
	at com.evgeniy.meetway.ui.theme.ThemeKt$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at com.evgeniy.meetway.ui.theme.ThemeKt.MeetWayTheme(Theme.kt:41)
	at com.evgeniy.meetway.MainActivity.MainContent(MainActivity.kt:145)
	at com.evgeniy.meetway.MainActivity.onCreate$lambda$0(MainActivity.kt:79)
	at com.evgeniy.meetway.MainActivity$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.ui.platform.ComposeView.Content(ComposeView.android.kt:431)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.ui.platform.AbstractComposeView$ensureCompositionCreated$1.invoke(ComposeView.android.kt:250)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.CompositionLocalsKt.ProvideCommonCompositionLocals(CompositionLocals.kt:216)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:145)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt$ProvideAndroidCompositionLocals$3.invoke(AndroidCompositionLocals.android.kt:144)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.ui.platform.AndroidCompositionLocals_androidKt.ProvideAndroidCompositionLocals(AndroidCompositionLocals.android.kt:133)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:140)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1$3.invoke(Wrapper.android.kt:139)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:390)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:139)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.internal.Expect_jvmKt.invokeComposable(Expect.jvm.kt:24)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3843)
	at androidx.compose.runtime.ComposerImpl.composeContent--ZbOJvo$runtime(Composer.kt:3747)
	at androidx.compose.runtime.CompositionImpl.composeContent(Composition.kt:832)
	at androidx.compose.runtime.Recomposer.composeInitial$runtime(Recomposer.kt:1234)
	at androidx.compose.runtime.CompositionImpl.composeInitial(Composition.kt:672)
	at androidx.compose.runtime.CompositionImpl.setContent(Composition.kt:639)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:123)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.setOnViewTreeOwnersAvailable(AndroidComposeView.android.kt:1990)
	at androidx.compose.ui.platform.WrappedComposition.setContent(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.WrappedComposition.onStateChanged(Wrapper.android.kt:168)
	at androidx.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.jvm.kt:316)
	at androidx.lifecycle.LifecycleRegistry.addObserver(LifecycleRegistry.jvm.kt:193)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:121)
	at androidx.compose.ui.platform.WrappedComposition$setContent$1.invoke(Wrapper.android.kt:114)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2077)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:223)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getJwt(SecureStorage.kt:56)
	at com.evgeniy.meetway.service.CloudFunctionService.getJwtForBackground(CloudFunctionService.kt:73)
	at com.evgeniy.meetway.service.CloudFunctionService.getUserData-gIAlu-s(CloudFunctionService.kt:417)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel$loadUserData$1.invokeSuspend(ProfileViewModel.kt:165)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ProfileViewModel.loadUserData(ProfileViewModel.kt:162)
	at com.evgeniy.meetway.ui.screens.profile.FirstScrinScreenKt$FirstScrinScreen$1$1.invokeSuspend(FirstScrinScreen.kt:225)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$21(NavGraph.kt:142)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda15.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:80)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.internal.DispatchedContinuationKt.resumeCancellableWith(DispatchedContinuation.kt:359)
	at kotlinx.coroutines.intrinsics.CancellableKt.startCoroutineCancellable(Cancellable.kt:26)
	at kotlinx.coroutines.CoroutineStart.invoke(CoroutineStart.kt:358)
	at kotlinx.coroutines.AbstractCoroutine.start(AbstractCoroutine.kt:124)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch(Builders.common.kt:52)
	at kotlinx.coroutines.BuildersKt.launch(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.launch$default(Builders.common.kt:43)
	at kotlinx.coroutines.BuildersKt.launch$default(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChats(ChatViewModel.kt:78)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ListAllMyChatsScreen$1$1.invokeSuspend(ListAllMyChatsScreen.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performTrampolineDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse(ChatViewModel.kt:158)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.parseChatsResponse$default(ChatViewModel.kt:131)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChats$1.invokeSuspend(ChatViewModel.kt:107)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Key algorithm: AES
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher parameters: null
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Provider: AndroidOpenSSL
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/GCM/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:49)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:45)
	at java.lang.ThreadLocal.setInitialValue(ThreadLocal.java:225)
	at java.lang.ThreadLocal.get(ThreadLocal.java:194)
	at java.lang.ThreadLocal.get(ThreadLocal.java:172)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/GCM/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:49)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:45)
	at java.lang.ThreadLocal.setInitialValue(ThreadLocal.java:225)
	at java.lang.ThreadLocal.get(ThreadLocal.java:194)
	at java.lang.ThreadLocal.get(ThreadLocal.java:172)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/GCM/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:49)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:45)
	at java.lang.ThreadLocal.setInitialValue(ThreadLocal.java:225)
	at java.lang.ThreadLocal.get(ThreadLocal.java:194)
	at java.lang.ThreadLocal.get(ThreadLocal.java:172)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/GCM/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:49)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce$1.initialValue(InsecureNonceAesGcmJce.java:45)
	at java.lang.ThreadLocal.setInitialValue(ThreadLocal.java:225)
	at java.lang.ThreadLocal.get(ThreadLocal.java:194)
	at java.lang.ThreadLocal.get(ThreadLocal.java:172)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getPassphrase(SecureStorage.kt:102)
	at com.evgeniy.meetway.util.EncryptionUtil.generateChatKey(EncryptionUtil.kt:58)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:163)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$ChatRow$2$1$decrypted$1.invokeSuspend(ListAllMyChatsScreen.kt:294)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.loadChatPreview(ChatViewModel.kt:414)
	at com.evgeniy.meetway.viewmodel.ChatViewModel.access$loadChatPreview(ChatViewModel.kt:22)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadChatPreview$1.invokeSuspend(ChatViewModel.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen(ListAllMyChatsScreen.kt:92)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt.ListAllMyChatsScreen$lambda$30(ListAllMyChatsScreen.kt:6)
	at com.evgeniy.meetway.ui.screens.chat.ListAllMyChatsScreenKt$$ExternalSyntheticLambda5.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.navigation.NavGraphKt.MeetWayNavGraph$lambda$32$lambda$31$lambda$20(NavGraph.kt:135)
	at com.evgeniy.meetway.navigation.NavGraphKt$$ExternalSyntheticLambda14.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:712)
	at androidx.navigation.compose.NavHostKt$NavHost$32$1.invoke(NavHost.kt:711)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider(SaveableStateHolder.kt:82)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.SaveableStateProvider(NavBackStackEntryProvider.kt:69)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.access$SaveableStateProvider(NavBackStackEntryProvider.kt:1)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:56)
	at androidx.navigation.compose.NavBackStackEntryProviderKt$LocalOwnersProvider$1.invoke(NavBackStackEntryProvider.kt:55)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.runtime.CompositionLocalKt.CompositionLocalProvider(CompositionLocal.kt:370)
	at androidx.navigation.compose.NavBackStackEntryProviderKt.LocalOwnersProvider(NavBackStackEntryProvider.kt:51)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:711)
	at androidx.navigation.compose.NavHostKt$NavHost$32.invoke(NavHost.kt:691)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:142)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:863)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:835)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.navigation.compose.NavHostKt.NavHost(NavHost.kt:663)
	at androidx.navigation.compose.NavHostKt.NavHost$lambda$80(NavHost.kt:25)
	at androidx.navigation.compose.NavHostKt$$ExternalSyntheticLambda3.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1.invoke(AnimatedContent.kt:818)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:121)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedContentKt.AnimatedContent(AnimatedContent.kt:873)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:23)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$9.invoke(AnimatedContent.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$loadMessages$1.invokeSuspend(ChatViewModel.kt:210)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String) called
[+] Transformation: AES/GCM/NoPadding
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:164)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.evgeniy.meetway.util.EncryptionUtil.decrypt(EncryptionUtil.kt:167)
	at com.evgeniy.meetway.viewmodel.ChatViewModel$decryptMessages$2.invokeSuspend(ChatViewModel.kt:497)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:832)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/CTR/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:113)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:87)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:95)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.getInstance(String, Provider) called
[+] Transformation: AES/ECB/NoPadding
[+] Provider: AndroidOpenSSL
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.getInstance(Native Method)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:47)
	at com.google.crypto.tink.subtle.EngineWrapper$TCipher.getInstance(EngineWrapper.java:40)
	at com.google.crypto.tink.subtle.EngineFactory$AndroidPolicy.getInstance(EngineFactory.java:139)
	at com.google.crypto.tink.subtle.EngineFactory.getInstance(EngineFactory.java:203)
	at com.google.crypto.tink.subtle.PrfAesCmac.instance(PrfAesCmac.java:51)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:68)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[!] WARNING: ECB mode detected in transformation: AES/ECB/NoPadding
[+] Cipher.init(Key) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Cipher parameters: null
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.PrfAesCmac.compute(PrfAesCmac.java:69)
	at com.google.crypto.tink.subtle.AesSiv.s2v(AesSiv.java:103)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:114)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: ENCRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.subtle.AesSiv.encryptDeterministically(AesSiv.java:119)
	at com.google.crypto.tink.daead.DeterministicAeadWrapper$WrappedDeterministicAead.encryptDeterministically(DeterministicAeadWrapper.java:78)
	at androidx.security.crypto.EncryptedSharedPreferences.encryptKey(EncryptedSharedPreferences.java:604)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:539)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Cipher.init(Key, params) called
[+] Mode: DECRYPT_MODE
[+] Key algorithm: AES
[+] Params: [object Object]
[+] Backtrace:
java.lang.Exception
	at javax.crypto.Cipher.init(Native Method)
	at com.google.crypto.tink.aead.internal.InsecureNonceAesGcmJce.decrypt(InsecureNonceAesGcmJce.java:136)
	at com.google.crypto.tink.subtle.AesGcmJce.decrypt(AesGcmJce.java:63)
	at com.google.crypto.tink.aead.AeadWrapper$WrappedAead.decrypt(AeadWrapper.java:91)
	at androidx.security.crypto.EncryptedSharedPreferences.getDecryptedObject(EncryptedSharedPreferences.java:546)
	at androidx.security.crypto.EncryptedSharedPreferences.getString(EncryptedSharedPreferences.java:422)
	at com.evgeniy.meetway.data.local.SecureStorage.getMyId(SecureStorage.kt:200)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen(ChatsScreen.kt:1403)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt.ChatsScreen$lambda$306(ChatsScreen.kt:12)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$$ExternalSyntheticLambda32.invoke(D8$$SyntheticClass:0)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipToGroupEnd(Composer.kt:3289)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.animation.AnimatedContentKt$AnimatedContent$6$1$5.invoke(AnimatedContent.kt:853)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:130)
	at androidx.compose.runtime.internal.ComposableLambdaImpl.invoke(ComposableLambda.kt:51)
	at androidx.compose.animation.AnimatedVisibilityKt.AnimatedEnterExitImpl(AnimatedVisibility.kt:752)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:27)
	at androidx.compose.animation.AnimatedVisibilityKt$AnimatedEnterExitImpl$4.invoke(AnimatedVisibility.kt:10)
	at androidx.compose.runtime.RecomposeScopeImpl.compose(RecomposeScopeImpl.kt:196)
	at androidx.compose.runtime.ComposerImpl.recomposeToGroupEnd(Composer.kt:2895)
	at androidx.compose.runtime.ComposerImpl.skipCurrentGroup(Composer.kt:3231)
	at androidx.compose.runtime.ComposerImpl.doCompose-aFTiNEg(Composer.kt:3855)
	at androidx.compose.runtime.ComposerImpl.recompose-aFTiNEg$runtime(Composer.kt:3779)
	at androidx.compose.runtime.CompositionImpl.recompose(Composition.kt:1075)
	at androidx.compose.runtime.Recomposer.performRecompose(Recomposer.kt:1364)
	at androidx.compose.runtime.Recomposer.access$performRecompose(Recomposer.kt:156)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2.invokeSuspend$lambda$22(Recomposer.kt:627)
	at androidx.compose.runtime.Recomposer$runRecomposeAndApplyChanges$2$$ExternalSyntheticLambda0.invoke(D8$$SyntheticClass:0)
	at androidx.compose.ui.platform.AndroidUiFrameClock$withFrameNanos$2$callback$1.doFrame(AndroidUiFrameClock.android.kt:39)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performFrameDispatch(AndroidUiDispatcher.android.kt:108)
	at androidx.compose.ui.platform.AndroidUiDispatcher.access$performFrameDispatch(AndroidUiDispatcher.android.kt:41)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.doFrame(AndroidUiDispatcher.android.kt:69)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1542)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:984)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

Process terminated
[KB2003::com.evgeniy.meetway ]->

Thank you for using Frida!
evgeniy@Evgeniys-MacBook-Pro-2 ~ %
```