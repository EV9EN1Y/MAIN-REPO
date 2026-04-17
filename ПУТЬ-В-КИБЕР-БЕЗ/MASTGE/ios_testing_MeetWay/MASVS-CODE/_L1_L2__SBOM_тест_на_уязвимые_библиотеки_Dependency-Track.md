# MASTG-TEST-0275: 
зависимости с известными уязвимостями в спецификации приложения

тест проверяет наличие зависимостей с известными уязвимостями в приложениях для iOS с помощью спецификации программного обеспечения (Software Bill of Materials, SBOM). 

SBOM должен быть в формате CycloneDX - стандарте для описания компонентов и зависимостей программного обеспечения

типо, разработчик может мне дать уже полностью готовый файл, и я сразу его проверяю, например через Dependency-Track

---------

файла у меня нет, но исходя из предыдущего теста, у меня есть полный список всех зависимостей

```

PODS:
  - absl (1.20240722.0)
  - AWSCore (2.41.0)
  - AWSS3 (2.41.0)
  - FBLPromises (2.4.0)
  - Firebase/Core (11.15.0)
  - FirebaseFirestore (11.15.0)
  - GoogleDataTransport (10.1.0)
  - GoogleUtilities (8.1.0)
  - gRPC (1.69.0)
  - leveldb (1.22.6)
  - nanopb (3.30910.0)
  - openssl_grpc (0.0.37)
  - YandexLoginSDK (3.0.2)

DEPENDENCIES:
  - absl
  - AWSCore
  - AWSS3
  - FBLPromises
  - Firebase/Core
  - FirebaseFirestore
  - GoogleDataTransport
  - GoogleUtilities
  - gRPC
  - leveldb
  - nanopb
  - openssl_grpc
  - YandexLoginSDK
```







--------


БУДУ ВЫПОЛНЯТЬ ПРОВЕРКУ ЧЕРЕЗ  
 
### OWASP Dependency-Check (CLI)

	1) ставлю прогу эту

```
brew install dependency-check
```

	1) также понадобиться ключ международ базы
https://nvd.nist.gov/developers/request-an-api-key

```q
dependency-check --scan ТУТ ФАЙЛ 🟣  путь --enableExperimental --format HTML --out ./scan-report --nvdApiKey ЗДЭСЯ_🟣_КЛЮЧ

```

так как файла со всеми зависимостями у меня нет
то создам файл из того, что я нарыл в прошлом тесте
в нужный формат

```json
cat > ~/Desktop/package.json << 'EOF'
{
  "name": "ios-app-dependencies",
  "version": "1.0.0",
  "description": "iOS dependencies for security scan",
  "dependencies": {
    "absl": "1.20240722.0",
    "aws-sdk": "2.41.0",
    "fbl-promises": "2.4.0",
    "firebase": "11.15.0",
    "google-data-transport": "10.1.0",
    "google-utilities": "8.1.0",
    "grpc": "1.69.0",
    "leveldb": "1.22.6",
    "nanopb": "3.30910.0",
    "openssl": "0.0.37",
    "yandex-login-sdk": "3.0.2"
  }
}
EOF
```

теперь можно запускать сканер по данному файлу

```q
dependency-check --scan ~/Desktop/package.json \
  --enableExperimental \
  --format HTML \
  --out ~/Desktop/scan-report \
  --nvdApiKey 9c680c05-ddee-426d-b7e3-14d7be35d1c6
```
первый раз нужно дождатсья, пока  все подгрузится
процесс не быстрый
![[Снимок экрана 2026-04-16 в 15.54.41.png]]

после сканера - посмотрим, чем там получилось в итоге
```q
open ~/Desktop/scan-report/dependency-check-report.html
```

результаты сканера:

⭕️  

----------------------


БУДУ ВЫПОЛНЯТЬ ПРОВЕРКУ ЧЕРЕЗ  
## ==Dependency-Track

# 👉❌  пишу постфакт, Dependency-Track - блокирует сигнатуры из РФ
так, что не подходит

попросил нейронку, она переупаковала в нужный json формат, и сохранил на рабочий стол этот файл!

```json
cat > ~/Desktop/sbom_ios_app.json << 'EOF'
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "version": 1,
  "metadata": {
    "timestamp": "2026-04-16T10:00:00Z",
    "tools": [
      {
        "name": "manual-generation",
        "vendor": "analyst",
        "version": "1.0"
      }
    ],
    "component": {
      "type": "application",
      "name": "ios-app-dependencies",
      "version": "1.0.0",
      "bom-ref": "app-1.0.0"
    }
  },
  "components": [
    {
      "type": "library",
      "name": "absl",
      "version": "1.20240722.0",
      "purl": "pkg:cocoapods/absl@1.20240722.0",
      "bom-ref": "absl-1.20240722.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "AWSCore",
      "version": "2.41.0",
      "purl": "pkg:cocoapods/AWSCore@2.41.0",
      "bom-ref": "AWSCore-2.41.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "AWSS3",
      "version": "2.41.0",
      "purl": "pkg:cocoapods/AWSS3@2.41.0",
      "bom-ref": "AWSS3-2.41.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "FBLPromises",
      "version": "2.4.0",
      "purl": "pkg:cocoapods/FBLPromises@2.4.0",
      "bom-ref": "FBLPromises-2.4.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "Firebase",
      "version": "11.15.0",
      "purl": "pkg:cocoapods/Firebase@11.15.0",
      "bom-ref": "Firebase-11.15.0",
      "group": "Firebase/Core"
    },
    {
      "type": "library",
      "name": "FirebaseFirestore",
      "version": "11.15.0",
      "purl": "pkg:cocoapods/FirebaseFirestore@11.15.0",
      "bom-ref": "FirebaseFirestore-11.15.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "GoogleDataTransport",
      "version": "10.1.0",
      "purl": "pkg:cocoapods/GoogleDataTransport@10.1.0",
      "bom-ref": "GoogleDataTransport-10.1.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "GoogleUtilities",
      "version": "8.1.0",
      "purl": "pkg:cocoapods/GoogleUtilities@8.1.0",
      "bom-ref": "GoogleUtilities-8.1.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "gRPC",
      "version": "1.69.0",
      "purl": "pkg:cocoapods/gRPC@1.69.0",
      "bom-ref": "gRPC-1.69.0",
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "leveldb",
      "version": "1.22.6",
      "purl": "pkg:cocoapods/leveldb@1.22.6",
      "bom-ref": "leveldb-1.22.6",
      "licenses": [
        {
          "license": {
            "id": "BSD-3-Clause"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "nanopb",
      "version": "3.30910.0",
      "purl": "pkg:cocoapods/nanopb@3.30910.0",
      "bom-ref": "nanopb-3.30910.0",
      "licenses": [
        {
          "license": {
            "id": "Zlib"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "openssl_grpc",
      "version": "0.0.37",
      "purl": "pkg:cocoapods/openssl_grpc@0.0.37",
      "bom-ref": "openssl_grpc-0.0.37",
      "licenses": [
        {
          "license": {
            "id": "OpenSSL"
          }
        }
      ]
    },
    {
      "type": "library",
      "name": "YandexLoginSDK",
      "version": "3.0.2",
      "purl": "pkg:cocoapods/YandexLoginSDK@3.0.2",
      "bom-ref": "YandexLoginSDK-3.0.2",
      "licenses": [
        {
          "license": {
            "name": "Proprietary"
          }
        }
      ]
    }
  ],
  "dependencies": [
    {
      "ref": "app-1.0.0",
      "dependsOn": [
        "absl-1.20240722.0",
        "AWSCore-2.41.0",
        "AWSS3-2.41.0",
        "FBLPromises-2.4.0",
        "Firebase-11.15.0",
        "FirebaseFirestore-11.15.0",
        "GoogleDataTransport-10.1.0",
        "GoogleUtilities-8.1.0",
        "gRPC-1.69.0",
        "leveldb-1.22.6",
        "nanopb-3.30910.0",
        "openssl_grpc-0.0.37",
        "YandexLoginSDK-3.0.2"
      ]
    },
    {
      "ref": "Firebase-11.15.0",
      "dependsOn": [
        "FirebaseFirestore-11.15.0"
      ]
    },
    {
      "ref": "gRPC-1.69.0",
      "dependsOn": [
        "absl-1.20240722.0",
        "openssl_grpc-0.0.37"
      ]
    },
    {
      "ref": "AWSS3-2.41.0",
      "dependsOn": [
        "AWSCore-2.41.0"
      ]
    }
  ]
}
EOF
```

-----

устанавливаю   Dependency-Track
```bash
mkdir $HOME/dependency-track && cd $HOME/dependency-track
curl -LO https://dependencytrack.org/docker-compose.yml
```

докер запущен
врубаю анализатор

```bash
docker compose up -d
```

после запуска - перехожу в веб интерфейс
http://localhost:8080
`admin` / `admin`   по умолчанию
нужно сменить пароль

создаю новый проект приложения

загружаю тот длинный json файл свой сюда - Components - Upload BOM

![[Снимок экрана 2026-04-16 в 14.03.13.png]]


-----

также нужно получить ключ от международной базы
https://nvd.nist.gov/developers/request-an-api-key

и далее в настроках админ - ресурсы уязвимостей
подрубаем все, и вводим апи ключ от международ базы  nvd.nist.gov !

![[Снимок экрана 2026-04-16 в 14.00.40.png]]

и он должен начать подкачивать все бд уязвимостей

на вкладке 1. Home Vulnerabilities  начнут появляться списски уязвимостей

логи можно глянуть так
```bash
docker compose logs apiserver | grep -i nvd
```
и когда будет выведено 
INFO [NvdMirrorTask] Mirroring completed successfully
INFO [NvdMirrorTask] Processed X vulnerabilities

значит все загрузилось!
итого минут 20 нужно

------

также нужно зарегаться здесь https://guide.sonatype.com
и получить апи ключ, который потом всунуть вот сюда: 
1. Administration
2. sonatype oss index

sonatype_pat_yEYGx2cIRsIUTcVwVh4zUKbrGIBwlr7WvJz21wlZP7x_naFR

![[Снимок экрана 2026-04-16 в 15.32.58.png]]

все настроил, но упс,,, я забанен, так как в России нахожуссь
{
  "error": "Banned"
   ну и пошли они нафиг!





