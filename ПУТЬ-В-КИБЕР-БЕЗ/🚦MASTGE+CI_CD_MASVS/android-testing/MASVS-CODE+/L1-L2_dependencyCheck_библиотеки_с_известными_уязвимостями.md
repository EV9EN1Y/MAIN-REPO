# MASTG-TEST-0272: Identify Dependencies with Known Vulnerabilities in the Android Project
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0272/

**есть ли в проекте сторонние библиотеки с известными уязвимостями**

-----

открываю исходный код и иду в файл  в папке -  
/Users/evgeniy/Documents/android-meetway/app/build.gradle.kts

нужно добавить в него настройки плагина для сканера
прямо в конец файла  build.gradle.kts

```kotlin

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.google.services)
    alias(libs.plugins.dependency.check)
}

android {
    namespace = "com.evgeniy.meetway"
    compileSdk {
        version = release(36)
    }

    defaultConfig {
        applicationId = "com.evgeniy.meetway"
        minSdk = 33
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures {
        compose = true
        buildConfig = true
    }
}

dependencies {
    // === AndroidX Core ===
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.activity.compose)

    // === Compose (BOM-managed) ===
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.graphics)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.compose.material3)
    implementation(libs.androidx.compose.material.icons.extended)

    // === Lifecycle + ViewModel ===
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.lifecycle.runtime.compose)

    // === Navigation ===
    implementation(libs.androidx.navigation.compose)

    // === Yandex SDK (раскомментировать после настройки Yandex Maven репозитория) ===
    // implementation("com.yandex.android:auth:3.1.0")

    // === Firebase ===
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.auth)
    implementation(libs.firebase.firestore)
    implementation(libs.firebase.messaging)

    // === Networking ===
    implementation(libs.retrofit)
    implementation(libs.retrofit.converter.gson)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging.interceptor)
    implementation(libs.gson)

    // === Media (ExoPlayer + CameraX) ===
    implementation(libs.media3.exoplayer)
    implementation(libs.media3.ui)
    implementation(libs.camerax.core)
    implementation(libs.camerax.camera2)
    implementation(libs.camerax.lifecycle)
    implementation(libs.camerax.view)

    // === Browser (Chrome Custom Tabs) ===
    implementation(libs.androidx.browser)

    // === Image Loading ===
    implementation(libs.coil.compose)

    // === Security ===
    implementation(libs.androidx.security.crypto)

    // === Biometric Auth ===
    implementation(libs.androidx.biometric)

    // === Coroutines ===
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.kotlinx.coroutines.android)

    // === Testing ===
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
    debugImplementation(libs.androidx.compose.ui.tooling)
    debugImplementation(libs.androidx.compose.ui.test.manifest)
}

dependencyCheck {
    formats = listOf("HTML", "XML", "JSON")
    failOnError = false
    autoUpdate = false
    skipConfigurations = listOf("debug", "androidTestDebug", "test")
    nvd {
        delay = 30000
        maxRetryCount = 0
    }
}


```

далее - в терминале запустить сканер

```q
cd /Users/evgeniy/Documents/android-meetway
  ./gradlew dependencyCheckAnalyz

```

При первом запуске ./gradlew dependencyCheckAnalyze плагин качает CVE-данные (бд)
   с NVD и Складывает их в H2-базу (Java-embedded database) по пути:
   ```
     ~/.gradle/dependency-check-data/11.0/odc.mv.db
   ```
   
результаты работы сканера

3 уязы он нашел

```q
Starting a Gradle Daemon, 1 stopped Daemon could not be reused, use --status for details

> Task :app:dependencyCheckAnalyze
Verifying dependencies for project app
Checking for updates and analyzing dependencies for vulnerabilities
----------------------------------------------------
.NET Assembly Analyzer could not be initialized and at least one 'exe' or 'dll' was scanned. The 'dotnet' executable could not be found on the path; either disable the Assembly Analyzer or add the path to dotnet core in the configuration.
The dotnet 8.0 core runtime or SDK is required to analyze assemblies


----------------------------------------------------
Sonatype OSS Index Analyzer disabled due to missing credentials. Authentication with token is now required, and OSS Index is migrating to Sonatype Guide. See https://dependency-check.github.io/DependencyCheck/analyzers/oss-index-analyzer.html for more information on authentication with Sonatype Guide OSS Index.

Generating report for project app


❌☠️❌  Found 3 vulnerabilities in project app ❌☠️❌


One or more dependencies were identified with known vulnerabilities in app:

browser-1.10.0.aar (pkg:maven/androidx.browser/browser@1.10.0, cpe:2.3:a:android:android_browser:1.10.0:*:*:*:*:*:*:*) : CVE-2008-7298
classes.jar (pkg:maven/androidx.browser/browser@1.10.0, 

cpe:2.3:a:android:android_browser:1.10.0:*:*:*:*:*:*:*) : CVE-2008-7298
httpclient-4.5.6.jar (pkg:maven/org.apache.httpcomponents/httpclient@4.5.6, 

cpe:2.3:a:apache:httpclient:4.5.6:*:*:*:*:*:*:*) : CVE-2020-13956


See the dependency-check report for more details.



[Incubating] Problems report is available at: file:///Users/evgeniy/Documents/android-meetway/build/reports/problems/problems-report.html

Deprecated Gradle features were used in this build, making it incompatible with Gradle 10.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

For more on this, please refer to https://docs.gradle.org/9.1.0/userguide/command_line_interface.html#sec:command_line_warnings in the Gradle documentation.

BUILD SUCCESSFUL in 22s
1 actionable task: 1 executed
```

-----

**Рекомендации:**

1. Обновить `androidx.browser` до последней версии (сейчас актуальная 1.8.0 или выше, проверь актуальность)
   
2. Обновить `org.apache.httpcomponents:httpclient` до версии 4.5.13 или выше, где эта уязвимость исправлена
   
3. Либо заменить `httpclient` на более современный `OkHttp`, который у тебя уже используется

--------


**Вывод:** Тест **ПРОВАЛЕН**. Приложение содержит сторонние библиотеки с известными уязвимостями. Рекомендуется обновить их до патченных версий

