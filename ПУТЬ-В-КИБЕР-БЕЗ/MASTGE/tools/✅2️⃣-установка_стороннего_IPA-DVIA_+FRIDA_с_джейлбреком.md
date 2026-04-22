установка_стороннего_IPA_с_джейлбреком и установка FRIDA

айфон уже с джейл брейком: и уже стоит Sileo
открываю Sileo и скачиваю **Filza** (позволяет видеть все внутр файлы телефона и устанавливать левые программы)


--------
#### СОДЕРЖАНИЕ

```c

|Джейлбрейк palera1n на iOS 16.7.14|✅

|Установка Sileo и Filza|

|Установка Frida 16.4.3 (правильная архитектура iphoneos-arm)|✅

|Установка AppSync Unified (через lukezgd.github.io/repo)|✅

|Установка DVIA-v2 через Filza|✅

|Установка NewTerm2 через репозиторий Chariz|✅

|Выравнивание версий Frida на Mac и iPhone (16.4.3)|✅

|Запуск sudo frida-server -l 0.0.0.0 на телефоне|✅

|Подключение с Mac через frida-ps -H 192.168.0.107|✅
```

-------

айфон 8 уже с джейлбрейком 👉 здесь можно подробнее узнать
[[✅1️⃣-джейл-брейк_Palera1n_iPhone8_ios16.7.14_macOS]]

### задача - поставить на айфон левое приложение DVIA-v2 для обучения по тестам  MASTGE, и установить Frida на айфон

--------

### 🟣 нужно скачать файл приложения

через сафари качаю в файлы, бинарник приложения

https://github.com/prateek147/DVIA-v2/releases/download/v2.0/DVIA-v2-swift.ipa

![[DVIA-v2-swift.ipa]]

скачал , приложение лежит  в файлах айфона

или вот это (рабочее)
https://github.com/prateek147/DVIA-v2/raw/refs/heads/master/DVIA-v2.ipa
также через браузер прям

-------

### 🟣 нужно скачать  filza
https://tigisoftware.com/cydia/     тут качаю filza через sileo
устанавливаю его там же в sileo
filza установлена - ярлык появился

-------

### 🟣 нужно скачать Frida

на мак
```c
# Если pipx не установлен
brew install pipx
pipx ensurepath

# Затем установите Frida
pipx install frida-tools

pip3 install --user frida-tools

frida --version
```


на телефон

через sileo **`https://build.frida.re`** - и там же через поиск установить ее

короче, и так и так можно, главное, чтобы версии совпадали на маке и на телефоне

https://github.com/frida/frida/releases/download/16.4.3/frida_16.4.3_iphoneos-arm.deb  это фриду качал на макбук и перекинул на айфон в файлы

![[frida_16.4.3_iphoneos-arm.deb]]

через sileo нашел файл  фриды и установил его 
и на выходе вижу

(успешно установилась фрида)
```c
скачал и нажал установить 
iPhone#
dpkg -i "/var/mobile/Containers/Shared/App
D715ECC4A15B/File Provider Storage/frida_16.4.3_ip honeos-arm.deb" ;
Selecting previously unselected package re.frida.s erver.
(Reading database ... 6103 files and directories c urrently installed.)
Preparing to unpack •../frida_16.4.3_iphoneos-arm. deb
•••
Unpacking re.frida.server (16.4.3)...
Setting up re.frida.server (16.4.3)
```

и тут же - действия - перезапуск! (Respring)

телефон погас - и запустился

--------

так как был перезапуск -то желательно перезапустить джейлбрейк
`palera1n -f`

перезапустил джейл-брейк

----

### 🟣 нужно скачать AppSync Unified
(это чтобы была возможность устанавливать любые приложения на телефон , в том числе и DVIA)

поэтому нужно через sileo скачать это - AppSync Unified 
вот здесь:
https://lukezgd.github.io/repo

и тут же  в sileo - установил AppSync Unified
в конце установки - есть кнопка - перезапуск телефона автоматом
(но в телефон еще и завис наполовину)

теперь , после включения телефона, нужно снова вернуть джейлбрейк
palera1n -f

джейлбрейк установлен!

----
### 🟣 теперь можно установить  DVIA

 иду в filza и найдя файл .ipa приложения - пробую установить DVIA

и, вуаля!!!!!  DVIA-v2 установлен!!!
появился ярлык на обоях!


---------
### 🟣 сразу установил терминал NewTerm2 на айфон

через sileo
https://repo.chariz.com/

далее в поиске там же устанавливаю его
и перезапускаю палерейн

-----

итого:

джейл брейк - установлен так: palera1n -f
скачан Sileo внутри palera1n 
установил Filza  через Sileo
фрида установлена (скачал файл на комп и перекинул на айфон)
установил AppSync Unified через Sileo
и в конце установил еще и DVIA-v2, который скачал по ссылке в сафари

-------

### 🟣 пробую запустить фрида-сервер

сверка версий фриды на телефоне и компе
на телефоне `frida-server --version`
у меня 16.4.3

на маке `frida --version`
16.4.3

версии на компе и айфоне должны совпадать
иначе будут проблемы  с подключением

=====================================

через терминал на айфоне 

`sudo frida-server -l 0.0.0.0 `
Пароль: `ваш пароль`

и после ввода пароля - терминал завис, типо это так и должно быть
и его нужно просто свернуть, но не закрывать

теперь на маке пробую подключиться
`frida-ps -H 192.168.1.107` 

---

 и в ответе получаю список всех процессов айфона
## ФРИДА УСПЕШНО ПОДКЛЮЧЕНА

```c
MacBook-Pro ~ % frida-ps -H 192.168.0.107
PID  Name
---  ------------------------------------------------------------
444
      NewTerm
186
      PosterBoard
300
      Safari
253
      Spotlight
355
      Календарь
305
      Настройки
111      ACCHWComponentAuthService
417      AMPIDService
424      ASPCarryLog
218      AccessibilityUIServer
467      AccountSubscriber
368      AccountSubscriber
463      AegirPoster
271      AppPredictionIntentsHelperService
294      AppSSODaemon
302      AppStore
 90      AppleCredentialManagerDaemon
299      AssetCacheLocatorService
189      BlueTool
164      CAReportingService
318      CMFSyncAgent
333      CacheDeleteAppContainerCaches
490      CacheDeleteDaily
339      CacheDeleteExtension
347      CalendarFocusConfigurationExtension
184      CalendarWidgetExtension
469      CategoriesService
384      CategoriesService
296      Ciconia
128      CloudKeychainProxy
221      CollectionsPoster
200      CollectionsPoster
102      CommCenter
145      CommCenterMobileHelper
489      CommCenterRootHelper
273      ContextService
319      CoreThreadCommissionerServiced
473      DASDelegateService
462      DPSubmissionService
389      DayStreamProcessorService
206      EmojiPosterExtension
201      ExtragalacticPoster
443      FamilyControlsAgent
303      Files
306      Filza
298      GSSCred
182      GeneralMapsWidget
199      GradientPosterExtension
304      Happ
310      HeuristicInterpreter
390      HistoricalAnalyzerService
240      IMDPersistenceAgent
466      InteractiveLegacyProfilesSubscriber
367      InteractiveLegacyProfilesSubscriber
344      KonaSynthesizer
468      LegacyProfilesSubscriber
369      LegacyProfilesSubscriber
503      Loader
458      LocalStorageFileProvider
171      LockScreenPeopleWidget_iOSExtension
476      MTLAssetUpgraderD
314      MTLCompilerService
295      MTLCompilerService
284      MTLCompilerService
275      MTLCompilerService
274      MTLCompilerService
217      MTLCompilerService
216      MTLCompilerService
180      MTLCompilerService
136      MTLCompilerService
135      MTLCompilerService
345      MacinTalkAUSP
349      MailShortcutsExtension
 73      ManagedSettingsAgent
465      ManagementTestSubscriber
366      ManagementTestSubscriber
348      MessagesActionExtension
337      MobileBackupCacheDeleteService
108      MobileGestaltHelper
323      MobileNotes
 89      OTACrashCopier
342      OTATaskingAgent
464      PasscodeSettingsSubscriber
365      PasscodeSettingsSubscriber
483      PerfPowerTelemetryClientRegistrationService
370      PerfPowerTelemetryClientRegistrationService
482      PerfPowerTelemetryReaderService
214      PhotosPosterProvider
190      PhotosReliveWidget
143      PowerUIAgent
207      PridePosterExtension
247      ProtectedCloudKeySyncing
460      RemoteManagementAgent
350      SafariBookmarksSyncAgent
126      ScreenTimeAgent
197      ScreenTimeWidgetExtension
504      Sileo
343      SiriTTSSynthesizerAU
 37      SpringBoard
244      StatusKitAgent
243      ThreeBarsXPCService
280      UARPUpdaterServiceAFU
282      UARPUpdaterServiceHID
283      UARPUpdaterServiceLegacyAudio
281      UARPUpdaterServiceUSBPD
210      UnityPosterExtension
276      UsageTrackingAgent
 33      UserEventAgent
313      UserFontManager
204      WeatherPoster
174      WeatherWidget
261      WiFiCloudAssetsXPCService
140      WiFiCloudAssetsXPCService
 51      WirelessRadioManagerd
289      accessoryd
 47      accessoryupdaterd
110      accountsd
272      adid
125      adprivacyd
226      afcd
 88      aggregated
132      akd
 57      amfid
100      amsaccountsd
254      amsengagementd
172      analyticsd
205      announced
142      apfs_iosd
124      appleaccountd
165      applecamerad
179      appstored
141      apsd
 69      askpermissiond
195      assetsd
 39      assistantd
 50      atc
139      audioclocksyncd
115      awdd
 42      axassetsd
 68      backboardd
420      backgroundassets.user
481      batteryintelligenced
 52      biomed
379      biomesyncd
131      biometrickitd
293      bird
 99      bluetoothd
 82      bluetoothuserd
315      bookassetd
419      budd
416      businessservicesd
123      calaccessd
112      callservicesd
238      captiveagent
159      carkitd
248      cdpd
106      cfprefsd
130      chronod
235      ckdiscretionaryd
144      cloudd
 78      cloudpaird
485      cloudphotod
245      com.apple.CallKit.CallDirectoryMaintenance
371      com.apple.DictionaryServiceHelper
107      com.apple.DriverKit-AppleBCMWLAN
242      com.apple.FaceTime.FTConversationService
267      com.apple.MapKit.SnapshotService
265      com.apple.MobileInstallationHelperService
328      com.apple.MobileSoftwareUpdate.CleanupPreparePathService
391      com.apple.SiriTTSService.TrialProxy
435      com.apple.StreamingUnzipService
442      com.apple.VideoSubscriberAccount.DeveloperService
351      com.apple.WebKit.WebContent
352      com.apple.accessibility.mediaaccessibilityd
183      com.apple.mobilenotes.WidgetExtension
334      com.apple.quicklook.ThumbnailsAgent
316      com.apple.sbd
237      com.apple.siri.embeddedspeech
203      com.apple.siri.embeddedspeech
224      companion_proxy
 46      configd
234      contactsd
 77      containermanagerd
 67      contextstored
220      coreauthd
163      coreduetd
178      coreidvd
122      corespeechd
338      coresymbolicationd
258      countryd
202      ctkd
 81      dasd
418      dataaccessd
478      deferredmediad
185      deleted
329      deleted_helper
196      destinationd
354      diagnosticextensionsd
 87      distnoted
121      dmd
158      donotdisturbd
394      dprivacyd
 80      driverkitd
401      druid
160      duetexpertd
346      extensionkitservice
198      extensionkitservice
170      extensionkitservice
101      fairplayd.H2
325      fairplaydeviceidentityd
232      familycircled
 56      familynotificationd
297      filecoordinationd
279      fileproviderd
307      financed
169      findmydeviced
438      fitcored
322      fitnesscoachingd
227      fmfd
168      fmflocatord
193      followupd
181      fontservicesd
500      frida-server
 94      frida-server
103      fseventsd
437      geocorrectiond
162      geod
268      gpsd
453      griddatad
388      healthappd
 45      healthd
208      homed
188      iconservicesagent
 64      identityservicesd
 76      imagent
119      ind
118      installcoordinationd
252      installd
233      intelligenceplatformd
177      itunescloudd
241      itunesstored
335      kbd
 55      keybagd
176      languageassetd
  1      launchd
320      linkd
129      liveactivitiesd
154      localizationswitcherd
 75      locationd
 86      lockdownd
 34      logd
374      logd_helper
445      login
137      lsd
138      mDNSResponder
317      maild
127      mapspushd
277      medialibraryd
 41      mediaremoted
 38      mediaserverd
455      metrickitd
471      microstackshot
155      misagent
 44      misd
385      mlruntimed
308      mmaintenanced
229      mobile_assertion_agent
246      mobile_installation_proxy
147      mobileactivationd
133      mobileassetd
161      mobilerepaird
 60      mobiletimerd
 79      nanoprefsyncd
109      nanoregistryd
321      nanoregistrylaunchd
262      nanotimekitcompaniond
 93      navd
356      ndoagent
236      nearbyd
134      nehelper
285      nesessionmanager
146      networkserviceproxy
 63      nfcd
230      notification_proxy
105      notifyd
156      nsurlsessiond
364      online-auth-agent
223      osanalyticshelper
222      ospredictiond
270      parsecd
 85      passd
380      passwordbreachd
291      pasted
 92      peakpowermanagerd
 36      peopled
150      pfd
287      photoanalysisd
470      pipelined
157      pkd
 49      powerd
309      privacyaccountingd
421      proactiveeventtrackerd
117      profiled
441      progressd
263      promotedcontentd
288      ptpd
 96      rapportd
 91      remindd
 54      remoted
363      remotemanagementd
120      remotepairingdeviced
331      replayd
459      revisiond
 40      routined
311      rtcreportingd
 35      runningboardd
 95      safetyalertsd
278      searchd
219      searchpartyd
148      securityd
 59      seld
336      sensorkitd
114      seserviced
 72      sharingd
113      siriactionsd
212      siriinferenced
211      siriknowledged
434      sirittsd
 83      sleepd
256      sociallayerd
 61      softwareupdated
269      sosd
372      splashboardd
499      sudo
498      sudo
257      suggestd
260      swcd
153      symptomsd
266      symptomsd-diag
151      syncdefaultsd
488      sysdiagnose
 48      tccd
 66      thermalmonitord
 71      timed
457      tipsd
191      touchsetupd
340      translationd
225      transparencyd
209      triald
152      trustd
215      useractivityd
 98      usermanagerd
341      videosubscriptionsd
251      vmd
239      voiced
 62      watchdogd
173      watchlistd
 65      wcd
330      weatherd
194      webbookmarksd
167      wifianalyticsd
 53      wifid
187      wifip2pd
250      wifivelocityd
446      zsh
MacBook-Pro ~ %
```
