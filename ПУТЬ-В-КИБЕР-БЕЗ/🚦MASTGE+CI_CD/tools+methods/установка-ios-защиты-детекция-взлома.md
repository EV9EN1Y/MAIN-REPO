
### устанавливаю в приложение простую джейл-брейк детекцию 


добавлю в приложение  [[0_MeetWay]] методы для детекции джейлбрейка

> Я СПЕЦИАЛЬНО НЕ ОБФУСЦИРУЮ ИМЕНА МЕТОДОВ И ПОИСКОВЫХ СТРОК в целя собственного обучения

```swift
 
 зведочки - это так копируется с xcode 🤨

**import** Foundation

**import** UIKit

  

**struct** JailbreakDetector {

    **static** **func** isJailbroken() -> Bool {

        **let** jailbreakPaths = [

            "/Applications/Cydia.app",

            "/Applications/Sileo.app",

            "/Applications/Zebra.app",

            "/usr/sbin/sshd",

            "/bin/bash",

            "/bin/sh",

            "/etc/apt",

            "/private/var/lib/apt",

            "/private/var/stash",

            "/private/var/tmp/cydia.log"

        ]

        **for** path **in** jailbreakPaths {

            **if** FileManager.default.fileExists(atPath: path) {

                **return** **true**

            }

        }

        // МОЖНо-ЛИ запись в системную папку

        **let** testPath = "/private/jailbreak_test_\(UUID().uuidString)"

        **do** {

            **try** "test".write(toFile: testPath, atomically: **true**, encoding: .utf8)

            **try** FileManager.default.removeItem(atPath: testPath)

            **return** **true** // получилось записать  ? = джейлбрейк

        } **catch** {}

        //  URL-схемы магазинов

        **let** jailbreakSchemes = ["cydia://", "sileo://", "zbra://", "filza://"]

        **for** scheme **in** jailbreakSchemes {

            **if** UIApplication.shared.canOpenURL(URL(string: scheme)!) {

                **return** **true**

            }

        }

        //  наличие fork()

        **typealias** ForkFunc = **@convention**(c) () -> Int32

        **if** **let** forkPtr = dlsym(UnsafeMutableRawPointer(bitPattern: -2), "fork") {

            **let** fork = unsafeBitCast(forkPtr, to: ForkFunc.**self**)

            **let** pid = fork()

            **if** pid >= 0 {

                **if** pid == 0 {

                    exit(0) // дочерний процесс

                } **else** {

                    **var** status: Int32 = 0

                    waitpid(pid, &status, 0)

                    **return** **true** // fork() сработал — джейлбрейк

                }

            }

        }

        **return** **false**

    }

}


------------






в аппделегат  
**** сам вызов функции в didFinishLaunchingWithOptions 

добававляю вызов и проверкой


   if JailbreakDetector.isJailbroken() {
    // Показываем алерт СИНХРОННО (без DispatchQueue)
    let alert = UIAlertController(
        title: "⚠️ Внимание, епта!",
        message: "Обнаружен джейлбрейк. Приложение не может быть запущено.",
        preferredStyle: .alert
    )
    alert.addAction(UIAlertAction(title: "OK", style: .destructive) { _ in
        exit(0)
    })
    
    // Находим rootViewController напрямую
    if let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene,
       let window = windowScene.windows.first {
        window.rootViewController = UIViewController() // Пустой контроллер
        window.makeKeyAndVisible()
        window.rootViewController?.present(alert, animated: true)
    } else {
        // Если сцена не найдена — просто выходим
        exit(0)
    }
    
    return true
}
    
    print("✅ Устройство не взломано, продолжаем")

```


вот здесь можно прочесть - как я эту защиту обошел через LLDB
[[_R_(SAST+DAST)-Jailbreak-Detection-in-Code+Runtime(radare2-Frida-LLDB-patch-Objection)]]

