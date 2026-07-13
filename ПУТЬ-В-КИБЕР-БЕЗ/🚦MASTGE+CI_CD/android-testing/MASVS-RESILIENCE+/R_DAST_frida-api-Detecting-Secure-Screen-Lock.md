# MASTG-TEST-0249: Runtime Use of Secure Screen Lock Detection APIs

тест провожу на приложении 👉 [[0_MeetWay]]

Этот тест проверяет, **использует ли приложение API для проверки блокировки экрана в реальном времени**, во время своей работы

Он является **динамическим дополнением** к статическому тесту `MASTG-TEST-0247`
[[R_SAST-api-Detecting-Secure-Screen-Lock]]

предыдущий тест - [[R_SAST-api-Detecting-Secure-Screen-Lock]] - лишь находил в коде вызовы методов отвечающих за блокировку экрана телефона = `KeyguardManager.isDeviceSecure()` и `BiometricManager.canAuthenticate()`, а данный этот тест подтверждает, что эти вызовы действительно _срабатывают_ при взаимодействии пользователя с приложением

то есть нужно динамически проверить вызовы функций:
`KeyguardManager.isDeviceSecure()` и `BiometricManager.canAuthenticate()`

буду выполнять тест через frida

на пк и тестовом андроиде 14 установлена фрида и стоит мое тестовое приложение - вот это   -   [[0_MeetWay]] 

--------

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

вижу PID своего приложение

17576   MeetWay
```


------

теперь можно глянуть какие модули / библиотеки используются в приложении

открываю косоль фриды 
```sh
frida -U -n MeetWay

смотрю библиотеки
Process.enumerateModules().forEach(function(m){ console.log(m.name); });


```

вижу этот
список имён всех библиотек и DEX-файлов, которые твоё приложение загрузило в память
```q
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -n MeetWay
     ____
    / _  |   Frida 17.9.1 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to KB2003 (id=6b62e119)

[KB2003::MeetWay ]-> Process.enumerateModules().forEach(function(m){ console.log
(m.name); });
app_process64
linker64
libandroid_runtime.so
libbase.so
libbinder.so
libcutils.so
libhidlbase.so
liblog.so
libnativeloader.so
libsigchain.so
libutils.so
libwilhelm.so
libc++.so
libc.so
libm.so
libdl.so
libharfbuzz_ng.so
libminikin.so
libz.so
android.media.audio.common.types-V2-cpp.so
audioclient-types-aidl-cpp.so
audioflinger-aidl-cpp.so
audiopolicy-types-aidl-cpp.so
spatializer-aidl-cpp.so
av-types-aidl-cpp.so
android.hardware.camera.device@3.2.so
libandroid_net.so
libbattery.so
libnetdutils.so
libmemtrack.so
libandroidfw.so
libappfuse.so
libcrypto.so
libdebuggerd_client.so
libbinder_ndk.so
libui.so
libgraphicsenv.so
libgui.so
libhwui.so
libmediandk.so
libpermission.so
libPlatformProperties.so
libsensor.so
libinput.so
libicu.so
libcamera_client.so
libcamera_metadata.so
libprocinfo.so
libsqlite.so
libEGL.so
libGLESv1_CM.so
libGLESv2.so
libGLESv3.so
libincfs.so
libdataloader.so
libvulkan.so
libETC1.so
libjpeg.so
libhardware.so
libhardware_legacy.so
libselinux.so
libmedia.so
libmedia_helper.so
libmediametrics.so
libmeminfo.so
libaudioclient.so
libaudioclient_aidl_conversion.so
libaudiofoundation.so
libaudiopolicy.so
libusbhost.so
libpdfium.so
libimg_utils.so
libnetd_client.so
libprocessgroup.so
libnativebridge_lazy.so
libnativeloader_lazy.so
libmemunreachable.so
libvintf.so
libnativedisplay.so
libnativewindow.so
libdl_android.so
libtimeinstate.so
server_configurable_flags.so
liboplusplugin.so
libimage_io.so
libultrahdr.so
libutilscallstack.so
libvndksupport.so
libnativebridge.so
libbase.so
libc++.so
libunwindstack.so
framework-permission-aidl-cpp.so
libmedia_codeclist.so
libaudiomanager.so
libdatasource.so
libstagefright.so
libstagefright_foundation.so
libstagefright_http_support.so
effect-aidl-cpp.so
shared-file-region-aidl-cpp.so
android.hardware.camera.common@1.0.so
android.hardware.graphics.common@1.0.so
android.hardware.memtrack@1.0.so
android.hardware.memtrack-V1-ndk.so
android.hardware.graphics.common-V4-ndk.so
android.hardware.graphics.allocator-V2-ndk.so
android.hardware.graphics.allocator@2.0.so
android.hardware.graphics.allocator@3.0.so
android.hardware.graphics.allocator@4.0.so
android.hardware.graphics.common@1.2.so
android.hardware.graphics.mapper@2.0.so
android.hardware.graphics.mapper@2.1.so
android.hardware.graphics.mapper@3.0.so
android.hardware.graphics.mapper@4.0.so
libgralloctypes.so
libsync.so
android.hardware.graphics.bufferqueue@1.0.so
android.hardware.graphics.bufferqueue@2.0.so
android.hardware.graphics.common@1.1.so
android.hidl.token@1.0-utils.so
liboplus_display_trace.so
libpng.so
libdng_sdk.so
libpiex.so
libexpat.so
android.hardware.graphics.composer3-V2-ndk.so
libheif.so
libprotobuf-cpp-lite.so
libft2.so
libandroid_runtime_lazy.so
libmediadrm.so
libmedia_omx.so
libmedia_jni_utils.so
libmediandk_utils.so
android.hardware.drm-V1-ndk.so
libicuuc.so
libicui18n.so
libc++.so
lib-platform-compat-native-api.so
libandroidicu.so
android.hardware.configstore@1.0.so
android.hardware.configstore-utils.so
libSurfaceFlingerProp.so
libunwindstack.so
libziparchive.so
android.system.suspend-V1-ndk.so
libpcre2.so
libpackagelistparser.so
liboplusvideoboostclient.so
mediametricsservice-aidl-cpp.so
audiopolicy-aidl-cpp.so
capture_state_listener-aidl-cpp.so
libaudio_aidl_conversion_common_cpp.so
libaudioutils.so
libmediautils.so
libnblog.so
libshmemcompat.so
packagemanager_aidl-cpp.so
libcgrouprc.so
libtinyxml2.so
libbpf_bcc.so
libbpf_minimal.so
liboplusloadframe.so
libjpegencoder.so
libjpegdecoder.so
liblzma.so
android.hardware.media.omx@1.0.so
android.hidl.memory@1.0.so
libstagefright_framecapture_utils.so
libcodec2.so
libcodec2_vndk.so
libmedia_omx_client.so
libsfplugin_ccodec.so
libsfplugin_ccodec_utils.so
libstagefright_codecbase.so
libstagefright_omx_utils.so
libhidlallocatorutils.so
libhidlmemory.so
android.hidl.allocator@1.0.so
android.hardware.cas.native@1.0.so
android.hardware.drm@1.0.so
liboplusmmdebug.so
android.hardware.common-V2-ndk.so
android.hardware.media@1.0.so
android.hidl.token@1.0.so
libmediadrmmetrics_lite.so
android.hardware.drm@1.1.so
android.hardware.drm@1.2.so
android.hardware.drm@1.3.so
android.hardware.drm@1.4.so
libbase.so
android.hardware.configstore@1.1.so
liblzma.so
libspeexresampler.so
libshmemutil.so
android.hardware.common.fmq-V1-ndk.so
android.hardware.media.bufferpool@2.0.so
android.hardware.media.bufferpool2-V1-ndk.so
libdmabufheap.so
libfmq.so
libion.so
libstagefright_bufferpool@2.0.1.so
libstagefright_aidl_bufferpool2.so
android.hardware.media.c2@1.0.so
libcodec2_client.so
libstagefright_bufferqueue_helper.so
libstagefright_omx.so
libstagefright_surface_utils.so
libstagefright_xmlparser.so
android.hidl.memory.token@1.0.so
android.hardware.cas@1.0.so
android.hidl.safe_union@1.0.so
android.hardware.media.c2@1.1.so
android.hardware.media.c2@1.2.so
libcodec2_hidl_client@1.0.so
libcodec2_hidl_client@1.1.so
libcodec2_hidl_client@1.2.so
libclang_rt.ubsan_standalone-aarch64-android.so
libeglextimpl.so
libguiextimpl.so
libaudioclientextimpl.so
vendor.oplus.hardware.performance-V2-ndk.so
libiatlasservice.so
libimmlistservice.so
liboplusaudioDump.so
libbsproxy.so
libvibrator.so
libmmlistparser.so
liboplus-uah-client.so
vendor.oplus.hardware.urcc-V1-ndk.so
libvulkanextimpl.so
liboplusavenhancements.so
libavenhancements.so
liboplusstagefright.so
libmediaplayerservice.so
liboplusmediaplayerservice.so
liboplusutils.so
liboplussfplugin_ccodec.so
libstagefright_httplive.so
libnbaio.so
libatlasservice.so
liboplusmultimediaconfig.so
libactivitymanager_aidl.so
libdrmframework.so
libpowermanager.so
liboplusvideoboostservice.so
liboplus_multimedia_kernel_event.so
libolc.so
libdrmframeworkcommon.so
android.hardware.power@1.0.so
android.hardware.power@1.1.so
android.hardware.power@1.2.so
android.hardware.power@1.3.so
android.hardware.power-V4-cpp.so
libskjpegencoderextimpl.so
liboplus_imageprocessing.so
libsensorextimpl.so
libinputextensionsimpl.so
libstatslog.so
libstatssocket.so
libstatspull.so
libnativehelper.so
libart.so
libartpalette.so
liblz4.so
libartbase.so
libdexfile.so
libprofile.so
heapprofd_client_api.so
libartpalette-system.so
libtombstoned_client.so
boot.oat
boot-framework-adservices.oat
libadbconnection.so
libadbconnection_client.so
libprocessgroup.so
libbase.so
libc++.so
libperfetto_hprof.so
libandroid.so
libxml2.so
android.hardware.power-V4-ndk.so
libaaudio.so
libaaudio_internal.so
aaudio-aidl-cpp.so
liboplusaudiopcmdump.so
libamidi.so
libcamera2ndk.so
libjnigraphics.so
libOpenMAXAL.so
libOpenSLES.so
libRS.so
android.hardware.renderscript@1.0.so
libstdc++.so
libwebviewchromium_plat_support.so
libicu_jni.so
libjavacore.so
libandroidio.so
libexpat.so
libopenjdk.so
libopenjdkjvm.so
libonetrace_jni.so
libonetrace_utils.so
libqti-at.so
libqti-perfd-client_system.so
vendor.qti.hardware.perf@2.2.so
vendor.qti.hardware.perf@2.3.so
vendor.qti.hardware.perf2-V1-ndk.so
vendor.qti.hardware.perf@2.0.so
vendor.qti.hardware.perf@2.1.so
libstats_jni.so
libphoenix_jni.so
libphoenix_native.so
libjavacrypto.so
libcrypto.so
libssl.so
libc++.so
libmedia_jni.so
libmediadrmmetrics_consumer.so
libmtp.so
libsonivox.so
android.hardware.tv.tuner-V2-ndk.so
libmediadrmmetrics_full.so
libasyncio.so
libprotobuf-cpp-full.so
libsoundpool.so
libaudioeffect_jni.so
librtp_jni.so
libstagefright_amrnb_common.so
librs_jni.so
liboplusextzawgyi.so
android.hardware.graphics.mapper@3.0-impl-qti-display.so
libutils.so
libcutils.so
libhardware.so
libhidlbase.so
libqdMetaData.so
libgrallocutils.so
libgralloccore.so
vendor.qti.hardware.display.mapper@3.0.so
vendor.qti.hardware.display.mapperextensions@1.0.so
android.hardware.graphics.mapper@2.0.so
android.hardware.graphics.mapper@2.1.so
vendor.qti.hardware.display.mapperextensions@1.1.so
android.hardware.graphics.mapper@3.0.so
libc++.so
libprocessgroup.so
libbase.so
libgralloc.qti.so
libgralloctypes.so
android.hardware.graphics.common@1.2.so
android.hardware.graphics.mapper@4.0.so
libion.so
android.hardware.graphics.common@1.0.so
android.hardware.graphics.common@1.1.so
android.hardware.graphics.common-V1-ndk_platform.so
android.hardware.common-V1-ndk_platform.so
android.hardware.graphics.mapper@4.0-impl-qti-display.so
vendor.qti.hardware.display.mapper@4.0.so
libEGL_adreno.so
libadreno_utils.so
libgsl.so
libz.so
libGLESv2_adreno.so
libllvm-glnext.so
libGLESv1_CM_adreno.so
eglSubDriverAndroid.so
vendor.qti.hardware.display.mapper@2.0.so
libcompiler_rt.so
libqti_performance.so
vendor.qti.hardware.iop@2.0.so
libwebviewchromium_loader.so
libhttpengine.so
libcrypto_httpengine.so
libskewknob_system.so
libframework-connectivity-tiramisu-jni.so
libsigprotector_jni.so
libframework-connectivity-jni.so
libSchedAssistJni.so
libSchedAssistExtImpl.so
liboplushwui_jni.so
liboplusgui_jni.so
libhwuiextimpl.so
libostatslog.so
libdolphin.so
gralloc.kona.so
[KB2003::MeetWay ]->
```

Нет ни одной библиотеки, которая бы явно указывала на использование коммерческого обфускатора или защитного пакета (типа DexGuard, libDexHelper, libprotect). также нет кастомных   библиотек

Этот список говорит о том, что приложение использует стандартные системные компоненты и не загружает какие-то скрытые или шифрованные модули извне

----

теперь сделаю скрипт для проверки 

пишу скрипт для фриды для проверки вызовов функций

```js
Java.perform(function () {
    console.log("[*] Скрипт запущен. Ждем вызовов...");

    // Перехват KeyguardManager.isDeviceSecure()
    try {
        var KeyguardManager = Java.use("android.app.KeyguardManager");
        KeyguardManager.isDeviceSecure.implementation = function () {
            var result = this.isDeviceSecure();
            console.log("[+] KeyguardManager.isDeviceSecure() вызван. Результат: " + result);
            // Выводим стек вызовов, чтобы понять, откуда пришел вызов
            console.log("[+] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return result;
        };
        console.log("[+] Хук на KeyguardManager.isDeviceSecure() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке KeyguardManager: " + e);
    }

    // Перехват BiometricManager.canAuthenticate()
    try {
        var BiometricManager = Java.use("androidx.biometric.BiometricManager");
        BiometricManager.canAuthenticate.overload('int').implementation = function (authenticators) {
            var result = this.canAuthenticate(authenticators);
            console.log("[+] BiometricManager.canAuthenticate() вызван с параметром: " + authenticators + ". Результат: " + result);
            console.log("[+] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return result;
        };
        console.log("[+] Хук на BiometricManager.canAuthenticate() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке BiometricManager: " + e);
    }

    // Также перехватим системный BiometricManager, если используется он
    try {
        var SystemBiometricManager = Java.use("android.hardware.biometrics.BiometricManager");
        SystemBiometricManager.canAuthenticate.overload('int').implementation = function (authenticators) {
            var result = this.canAuthenticate(authenticators);
            console.log("[+] SystemBiometricManager.canAuthenticate() вызван с параметром: " + authenticators + ". Результат: " + result);
            return result;
        };
        console.log("[+] Хук на SystemBiometricManager.canAuthenticate() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке SystemBiometricManager: " + e);
    }

    console.log("[*] Все хуки установлены. теперь я буду тыкать приложение везде");
});
```

запускаю

```q
frida -U -n MeetWay -l /Users/evgeniy/Desktop/frida-android-scripts/hook_screen_lock.js
```

скрипт - не обнаружил вызовы методов :

KeyguardManager.isDeviceSecure()
BiometricManager.canAuthenticate()

------

теперь нужно сделать тоже самое - но с запуском приложения через фрида

закрыл приложение

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/frida-android-scripts/hook_screen_lock.js
```

и снова ничего нет, методы не вызываются!!

-----------

## вывод

данный тест можно считать проваленным, так как не используются динамически методы блокировки экрана приложения, то есть , если в приложении есть конфидециальная информация, то при отсутствии блокировки телефона - злоумышленник сможет легко открыть и телефон и приложение.


### Оценка рисков

Приложение позволяет пользователю получить доступ к чувствительным данным и функциям, не требуя наличия активной системной блокировки экрана. Это создает прямую угрозу безопасности в следующих сценариях:

Потеря или кража устройства: злоумышленник может получить полный доступ к данным приложения, обойдя любые механизмы защиты, так как приложение не запрашивает подтверждение личности

Несанкционированный доступ: любой человек, получивший физический доступ к разблокированному устройству, может беспрепятственно открыть и использовать приложение

------------
### Рекомендации

В целях повышения безопасности приложения и защиты пользовательских данных настоятельно рекомендуется внедрить следующие меры:

Добавить обязательную проверку наличия системной блокировки экрана через KeyguardManager.isDeviceSecure() при запуске приложения и перед выполнением критических действий

В случае отсутствия блокировки экрана приложение должно блокировать доступ к основному функционалу и направлять пользователя в системные настройки для ее установки

Рассмотреть возможность внедрения биометрической аутентификации через BiometricPrompt как дополнительного фактора защиты для доступа к чувствительным данным