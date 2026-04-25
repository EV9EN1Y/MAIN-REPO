Здесь я настроил SLL PInning для своего приложения на Swift, ios 17.6

вот ОБЗОР самого приложения : [[0_MeetWay]]

чтобы тренироваться во обходе Sll pinning с помощью, frida например

---

### метод в AppDelegate

в AppDelegate добавил основную функцию инициализации  SSL Pinning

задача:

Настройка TrustKit для обычных HTTPS запросов

Добавил домены  (kTSKPinnedDomains) которые буду защищать (при обнаружении поддельных сертификатов)

Также здесь находятся хеши сертификатов - публичные ключи, если сервер возвращает не этот хеш, соединение разрывается

реализованы стандартные правила проверок:
`kTSKEnforcePinning: true ` - Блокировать соединение при несовпадении
`kTSKIncludeSubdomains: true`  - Защищать все поддомены
`kTSKSwizzleNetworkDelegates: true` -  авто перехват
`GRPCInterceptor.injectIntoYandexSDK() ` -  перехватывает gRPC вызовы

```
запрос к functions.yandexcloud.net
         ↓
    [TrustKit перехватывает]
         ↓
    "Какой сертификат прислал сервер?"
         ↓
    извлекает хеш публичного ключа: "abc123..."
         ↓
    сравнивает с разрешенными: ["Y5SL...", "C5+lp..."]
         ↓
    ┌──────────────┬──────────────┐
    ↓              ↓              ↓
СОВПАЛ         НЕ СОВПАЛ      НЕТ СЕРТИФИКАТА
    ↓              ↓              ↓
ПРОПУСКАЮ      БЛОКИРУЮ       БЛОКИРУЮ
                ↓
       алерт (если есть делегат)
```
```swift
 **private** **func** setupTrustKitWithGRPC() {

            // 1. Настраиваем TrustKit для обычных HTTPS запросов

            **let** trustKitConfig: [String: **Any**] = [

                kTSKSwizzleNetworkDelegates: **true**,

                kTSKPinnedDomains: [

                    "api.yandex.ru": [

                        kTSKPublicKeyHashes: [

                            "2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4=",

                            "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="

                        ],

                        kTSKIncludeSubdomains: **true**,

                        kTSKEnforcePinning: **true**

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

                    ],

                    "login.yandex.ru": [

                        kTSKPublicKeyHashes: [

                            "kyHvcBfJA5The/fNkQH0q/IVw/uk5bfphH41bQmDATo=",

                            "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="

                        ],

                        kTSKIncludeSubdomains: **false**,

                        kTSKEnforcePinning: **true**

                    ],

                    "firestore.googleapis.com": [

                        kTSKPublicKeyHashes: [

                            "+vLEyBQERqRwpgiGwEi7Dx6jujKTEdoJzr4CSmYXCz0="

                        ],

                        kTSKIncludeSubdomains: **true**,

                        kTSKEnforcePinning: **false**

                    ]

                ]

            ]

            TrustKit.initSharedInstance(withConfiguration: trustKitConfig)

            print("✅ TrustKit сконфигурирован для HTTPS запросов")

            // 2. Дополнительно перехватываем gRPC вызовы Yandex SDK

            GRPCInterceptor.injectIntoYandexSDK()

        }
```


--------

##  класс GRPCInterceptor
это кастомное решение для защиты gRPC запросов, которые TrustKit не может перехватить автоматически!!!

TrustKit работает только с **HTTPS** запросами через `URLSession`. Но Yandex SDK использует **gRPC** (который работает поверх HTTP/2, но с своей реализацией соединений). Поэтому вы создали свой перехватчик

методы:
`injectIntoYandexSDK  и  swizzled_request` - Поиск и Swizzling методов Yandex SDK

> **Swizzling** - это техника в iOS, которая позволяет заменить реализацию существующего метода на свою собственную в runtime (во время выполнения программы), то есть перехватить фунцию и что-либо сделать 


- Yandex SDK может использовать разные имена классов в разных версиях
- Swizzling позволяет "внедриться" в работу SDK без изменения его кода
- Вы заменяете оригинальные методы на свои

`verifyPinning`  - синхронно проверяет сертификат, потому что, хоть и - gRPC методы обычно асинхронные, но проверка должна произойти ДО того, как запрос уйдет на сервер, семафор позволяет дождаться результата проверки

`getHashFromTrust` - парсит сертификаты сами, извлекает хеш

схема:
```
Yandex SDK хочет сделать gRPC запрос
         ↓
Вызывает оригинальный метод (например, connectToHost)
         ↓
    [SWIZZLING]
         ↓
Вызывается ваш swizzled_request вместо оригинального
         ↓
    verifyPinning(for: host)
         ↓
    Создает фейковый HTTPS запрос к тому же хосту
         ↓
    Получает сертификат и вычисляет хеш
         ↓
    Сравнивает с ожидаемыми хешами
         ↓
    ┌─────────────┬─────────────┐
    ↓             ↓             ↓
Совпал       Не совпал     Ошибка
    ↓             ↓             ↓
Пропускаем    БЛОКИРУЕМ    БЛОКИРУЕМ
    ↓             ↓
Вызываем      Показываем
оригинальный  алерт ❌
метод ✅
```

```swift
**import** Foundation

**import** UIKit

**import** CommonCrypto

**import** TrustKit

**import** Security

  

  

// **MARK: - gRPC Interceptor для Yandex SDK**

**class** GRPCInterceptor {

    **static** **func** injectIntoYandexSDK() {

        print("🔧 Начинаем перехват gRPC вызовов Yandex SDK...")

        // Пытаемся найти классы YandexLoginSDK

        **let** possibleClasses = [

            "YandexLoginSDK.YandexLoginService",

            "YandexLoginSDK.NetworkService",

            "YandexLoginSDK.TokenService",

            "YandexLoginSDK.AuthService",

            "YMKTransport",

            "YMKNetworkManager",

            "GRPCHost",

            "GRPCChannel",

            "GRPCCall"

        ]

        **var** foundAny = **false**

        **for** className **in** possibleClasses {

            // Пробуем получить класс по имени

            **guard** **let** targetClass = NSClassFromString(className) ?? findClassByName(className) **else** {

                **continue**

            }

            print("🔍 Найден класс: \(targetClass)")

            // Пробуем разные варианты методов для swizzling

            **let** selectorVariants = [

                "makeRequest:completion:",

                "sendRequest:completion:",

                "performRequest:completion:",

                "callWithRequest:completion:",

                "execute:completion:",

                "startWithHost:port:",

                "connectToHost:port:",

                "createChannelWithHost:port:"

            ]

            **for** selectorString **in** selectorVariants {

                **let** selector = NSSelectorFromString(selectorString)

                **let** swizzledSelector = **#selector**(swizzled_request(host:port:completion:))

                **if** **let** originalMethod = class_getInstanceMethod(targetClass, selector) {

                    **let** swizzledMethod = class_getInstanceMethod(GRPCInterceptor.**self**, swizzledSelector)!

                    method_exchangeImplementations(originalMethod, swizzledMethod)

                    print("✅ Swizzled \(className).\(selectorString)")

                    foundAny = **true**

                    **break**

                }

            }

        }

        **if** !foundAny {

            print("⚠️ Не найдены классы Yandex SDK для swizzling")

            print("   Попробуем альтернативный подход - перехват через URLProtocol...")

            setupURLProtocolInterceptor()

        }

    }

    // Вспомогательная функция для поиска класса

    **private** **static** **func** findClassByName(_ name: String) -> AnyClass? {

        // Пробуем разные варианты имени класса

        **let** variants = [

            name,

            name.replacingOccurrences(of: "YandexLoginSDK.", with: ""),

            "_TtC15YandexLoginSDK" + name.components(separatedBy: ".").last!,

            name.components(separatedBy: ".").last!

        ]

        **for** variant **in** variants {

            **if** **let** found = NSClassFromString(variant) {

                **return** found

            }

        }

        **return** **nil**

    }

    **@objc** **static** **func** swizzled_request(host: String, port: Int, completion: **@escaping** (**Any**?, Error?) -> Void) {

        print("🎯 Перехвачен gRPC запрос к: \(host):\(port)")

        // Проверяем pinning для этого хоста

        **if** !verifyPinning(for: host) {

            print("❌ SSL Pinning failed для \(host) - запрос БЛОКИРУЕТСЯ")

            // АЛЕРТ

                **if** **let** appDelegate = UIApplication.shared.delegate **as**? AppDelegate {

                    appDelegate.showSSLPinningAlert(host: host)

                }

            **let** error = NSError(domain: "SSLPinningError",

                               code: -1200,

                               userInfo: [NSLocalizedDescriptionKey: "SSL Pinning failed for \(host)"])

            completion(**nil**, error)

            **return**

        }

        print("✅ SSL Pinning успешно пройден для \(host)")

        // ВАЖНО: Здесь нужно вызвать оригинальную реализацию

        // Для этого сохраняем оригинальный IMP при swizzling

        // Упрощённый вариант - вызываем через perform

        **let** selector = **#selector**(swizzled_request(host:port:completion:))

        // Находим оригинальную реализацию через метод класса

        **if** **let** originalMethod = class_getClassMethod(GRPCInterceptor.**self**, selector) {

            // Получаем IMP оригинального метода (до swizzling)

            // В реальности нужно сохранять оригинальный IMP при сваззлинге

            print("⚠️ Вызов оригинального метода для \(host)")

        }

    }

    **private** **static** **func** verifyPinning(for host: String) -> Bool {

        **let** pinnedHashes: [String: [String]] = [

            "functions.yandexcloud.net": [

                "Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=",

                "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="

            ],

            "storage.yandexcloud.net": [

                "CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=",

                "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="

            ]

        ]

        **guard** **let** expectedHashes = pinnedHashes[host] **else** {

            **return** **true**

        }

        // Синхронное получение хеша сертификата

        **guard** **let** actualHash = getCertificateHashSync(for: host) **else** {

            print("⚠️ Не удалось получить сертификат для \(host)")

            **return** **false**

        }

        **let** isValid = expectedHashes.contains(actualHash)

        **if** !isValid {

            print("❌ Хеш не совпадает для \(host)")

            print("   Ожидался: \(expectedHashes.joined(separator: " или "))")

            print("   Получен: \(actualHash)")

        }

        **return** isValid

    }

  

    // Синхронное получение хеша сертификата

    **private** **static** **func** getCertificateHashSync(for host: String) -> String? {

        **let** url = URL(string: "https://\(host)")!

        **var** resultHash: String?

        **let** semaphore = DispatchSemaphore(value: 0)

        **let** session = URLSession(configuration: .ephemeral)

        **let** task = session.dataTask(with: url) { _, response, error **in**

            **defer** { semaphore.signal() }

            **guard** error == **nil** **else** {

                print("⚠️ Ошибка соединения: \(error!.localizedDescription)")

                **return**

            }

            // Исправленный способ получения serverTrust

            **guard** **let** httpsResponse = response **as**? HTTPURLResponse,

                  **let** serverTrust = httpsResponse.value(forKey: "serverTrust") **else** {

                print("⚠️ Не удалось получить serverTrust для \(host)")

                **return**

            }

            // serverTrust уже имеет тип SecTrust, не нужно кастить

            resultHash = getHashFromTrust(serverTrust **as**! SecTrust)

        }

        task.resume()

        _ = semaphore.wait(timeout: .now() + 10)

        session.finishTasksAndInvalidate()

        **return** resultHash

    }

    **private** **static** **func** getActualCertificateHash(for host: String) -> String? {

        **let** url = URL(string: "https://\(host)")!

        **let** session = URLSession(configuration: .ephemeral)

        **let** semaphore = DispatchSemaphore(value: 0)

        **var** resultHash: String?

        **let** task = session.dataTask(with: url) { _, response, error **in**

            **defer** { semaphore.signal() }

            **if** **let** error = error {

                print("⚠️ Ошибка получения сертификата для \(host): \(error)")

                **return**

            }

            // Исправленный способ - без приведения к SecTrust

            **guard** **let** httpsResponse = response **as**? HTTPURLResponse,

                  **let** serverTrustValue = httpsResponse.value(forKey: "serverTrust") **else** {

                print("⚠️ Не удалось получить serverTrust для \(host)")

                **return**

            }

            // serverTrustValue имеет тип Any, но в реальности это SecTrust

            **let** serverTrust = serverTrustValue **as**! SecTrust

            resultHash = getHashFromTrust(serverTrust)

        }

        task.resume()

        _ = semaphore.wait(timeout: .now() + 5)

        session.finishTasksAndInvalidate()

        **return** resultHash

    }

  

    // Вспомогательный метод для получения хеша из SecTrust

    **private** **static** **func** getHashFromTrust(_ trust: SecTrust) -> String? {

        // Сначала оцениваем trust

        **var** error: CFError?

        **let** evaluationSucceeded = SecTrustEvaluateWithError(trust, &error)

        **guard** evaluationSucceeded **else** {

            print("⚠️ Trust evaluation failed: \(error?.localizedDescription ?? "unknown error")")

            **return** **nil**

        }

        // Получаем сертификат

        **guard** **let** certificate = SecTrustGetCertificateAtIndex(trust, 0) **else** {

            print("⚠️ No certificate found")

            **return** **nil**

        }

        // Получаем публичный ключ

        **let** publicKey = SecCertificateCopyKey(certificate)

        **guard** **let** publicKey = publicKey **else** {

            print("⚠️ No public key found")

            **return** **nil**

        }

        // Получаем данные публичного ключа

        **var** keyCopyError: Unmanaged<CFError>?

        **guard** **let** keyData = SecKeyCopyExternalRepresentation(publicKey, &keyCopyError) **as** Data? **else** {

            print("⚠️ Failed to copy public key: \(keyCopyError?.takeRetainedValue().localizedDescription ?? "unknown")")

            **return** **nil**

        }

        // Вычисляем SHA256

        **var** hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))

        keyData.withUnsafeBytes {

            _ = CC_SHA256($0.baseAddress, CC_LONG(keyData.count), &hash)

        }

        **return** Data(hash).base64EncodedString()

    }

    // Альтернативный метод через URLProtocol

    **private** **static** **func** setupURLProtocolInterceptor() {

        // Регистрируем кастомный URLProtocol для перехвата

        URLProtocol.registerClass(GRPCURLProtocol.**self**)

        print("✅ URLProtocol зарегистрирован для перехвата")

    }

}
```


-----------
##  класс GRPCURLProtocol
GRPCURLProtocol использует механизм `URLProtocol` для перехвата сетевых запросов на более низком уровне

```
1. Yandex SDK создает запрос к functions.yandexcloud.net
         ↓
2. URLSession проверяет: "А есть ли URLProtocol для этого запроса?"
         ↓
3. Вызывается GRPCURLProtocol.canInit(with: request) → true
         ↓
4. Создается экземпляр GRPCURLProtocol
         ↓
5. Вызывается startLoading()
         ↓
6. Запускается реальный сетевой запрос
         ↓
7. Сервер отвечает и начинается TLS handshake
         ↓
8. Вызывается urlSession( didReceive challenge: )
         ↓
9. Извлекаем сертификат и проверяем хеш
         ↓
    ┌─────────────┬─────────────┐
    ↓             ↓             ↓
Совпал        Не совпал     Ошибка
    ↓             ↓
Пропускаем    ПОКАЗЫВАЕМ
запрос        АЛЕРТ! ❌
    ↓
    И БЛОКИРУЕМ
```

```swift
**import** Foundation

**import** UIKit

**import** CommonCrypto

**import** TrustKit

**import** Security

  

// **MARK: - URLProtocol для перехвата gRPC**

**class** GRPCURLProtocol: URLProtocol, URLSessionDelegate {

    **private** **var** dataTask: URLSessionDataTask?

    **override** **class** **func** canInit(with request: URLRequest) -> Bool {

        **guard** **let** host = request.url?.host **else** { **return** **false** }

        **let** targetHosts = ["functions.yandexcloud.net", "storage.yandexcloud.net", "api.yandex.ru"]

        **return** targetHosts.contains { host.hasSuffix($0) }

    }

    **override** **class** **func** canonicalRequest(for request: URLRequest) -> URLRequest {

        **return** request

    }

    **override** **func** startLoading() {

        **let** config = URLSessionConfiguration.default

        **let** session = URLSession(configuration: config, delegate: **self**, delegateQueue: **nil**)

        dataTask = session.dataTask(with: request) { [**weak** **self**] data, response, error **in**

            **if** **let** error = error {

                **self**?.client?.urlProtocol(**self**!, didFailWithError: error)

            } **else** {

                **if** **let** response = response {

                    **self**?.client?.urlProtocol(**self**!, didReceive: response, cacheStoragePolicy: .notAllowed)

                }

                **if** **let** data = data {

                    **self**?.client?.urlProtocol(**self**!, didLoad: data)

                }

                **self**?.client?.urlProtocolDidFinishLoading(**self**!)

            }

        }

        dataTask?.resume()

    }

    **override** **func** stopLoading() {

        dataTask?.cancel()

    }

    // URLSessionDelegate для pinning

    **func** urlSession(_ session: URLSession,

                    didReceive challenge: URLAuthenticationChallenge,

                    completionHandler: **@escaping** (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {

        **guard** challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,

              **let** serverTrust = challenge.protectionSpace.serverTrust **else** {

            completionHandler(.cancelAuthenticationChallenge, **nil**)

            **return**

        }

        **let** host = challenge.protectionSpace.host

        // Используем TrustKit для проверки

        **if** TrustKit.sharedInstance().pinningValidator.handle(challenge, completionHandler: completionHandler) == **true** {

            **return**

        }

        // Fallback проверка

        **let** actualHash = getCertificateHash(from: serverTrust)

        **let** expectedHashes = [

            "Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=",

            "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="

        ]

        **if** expectedHashes.contains(actualHash) {

            completionHandler(.useCredential, URLCredential(trust: serverTrust))

        } **else** {

            //  АЛЕРТ

               **if** **let** appDelegate = UIApplication.shared.delegate **as**? AppDelegate {

                   appDelegate.showSSLPinningAlert(host: host)

               }

            print("❌ Certificate pinning failed for \(host)")

            completionHandler(.cancelAuthenticationChallenge, **nil**)

        }

    }

    **private** **func** getCertificateHash(from trust: SecTrust) -> String {

        **guard** **let** certificate = SecTrustGetCertificateAtIndex(trust, 0),

              **let** publicKey = SecCertificateCopyKey(certificate) **else** {

            **return** ""

        }

        **var** error: Unmanaged<CFError>?

        **guard** **let** keyData = SecKeyCopyExternalRepresentation(publicKey, &error) **as** Data? **else** {

            **return** ""

        }

        **var** hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))

        keyData.withUnsafeBytes {

            _ = CC_SHA256($0.baseAddress, CC_LONG(keyData.count), &hash)

        }

        **return** Data(hash).base64EncodedString()

    }

}
```


