
### здесь я создам на вирт машине необходимое окружение и полностью с нуля настрою ci-cd для gitlab и gthub + выполню первый пайплан-тест!!!!!

(сразу скажу, что с gitlab - все получилось - но в конце он перестал работать из-за санкций, а вот с github получись все на 100%, поэтому, этап с gitlab можно пропустить, )

тестить буду свое приложение в итоге [[0_MeetWay]]

https://cloud.ru  - буду использовать бесплатный тариф ( Ubuntu 22.04  2 CPU, 4 GB RAM, 30 GB)

#### цель:

-- настройка паплайнов с SAST (и других анализаторов (MobSF, Semgrep) ) в рамках CI/CD на удаленном  self-hosted с раннером (gitlab-runner), с докер контейнером (Docker executor) который будет создаваться каждый раз для выполнения очередной задачи + управлять всем этим будет GitLab actions , + нужно будет настроитьт сервер таким образом - чтобы это было безопасно в рамках корпоративной разработки! 
(Docker executor в не-привилегированном режиме)
(Runner нужно залочить за конкретным проектом, выполнение заданий без тегов будет отключено. Сервер будет защищён файрволом и настройками systemd)


Принцип работы:

- *GitHub/GitLab* - «пульт управления»: хранит код, запускает пайплайны, раздаёт задания
   
- *Self‑hosted runner на сервере* - «исполнитель»: забирает задания, запускает их визолированной среде, возвращает результаты
   
- Ключевой момент: runner не выполняет код напрямую, а создаёт «песочницу» (контейнер) для каждого задания
#### план

1) нужен сервер (использую cloud.ru - для старта норм)
2) буду использовать GitLab actions



---------

## приступаю к реализации

🟢  зарегал машину и оплатил ip на https://cloud.ru для вм
 ( Ubuntu 22.04  2 CPU, 4 GB RAM, 30 GB)

при создании машины получил приват кей
сохранил его 

теперь нужно ему дать права (только владелец, чтение/запись)

```shell
chmod 600 /Users/evgeniy/Desktop/вирт\ МАШИНА/id_rsa-2
```

-----

подрубаюсь к серверу

```shell
ssh -i /Users/evgeniy/Desktop/вирт\ МАШИНА/id_rsa-2 логинМашины@77.777.77.77
```


-------

обновляю пакеты и ставлю докер

```shell
sudo apt update && sudo apt install -y docker.io

----------------------

после установки - перезапуск

sudo reboot
```

затем снова подключаюсь

проверю что докер жив и стоит

```shell
sudo docker --version

у меня так 
Docker version 29.1.3, build 29.1.3-0ubuntu3~22.04.1
```

теперь нужно настроить управление контейнерами, чтобы текущий пользователь мог запускать докер команды без sudo
нужно добавить в группу `docker` текущего пользователя user1 , чтобы управлять контейнерами от своего имени (у меня по умолчанию стоит  user1)

```shell
sudo usermod -aG docker $USER
```

теперь нужно выйти и зайти снова
```shell
exit

ssh -i /Users/evgeniy/Desktop/вирт\ МАШИНА/id_rsa-2 user1@66.666.66.66
```


чтобы понять, работает ли все как нужно

докер запускаю без судо

```shell
docker --version

# работает, ОТЛИЧНО


# --------------------

#проверю, что можно запускать контейнеры

docker run hello-world

```

все отлично

он не нашел готовый образ
Unable to find image 'hello-world:latest' locally

и как следует - скачал его из Docker Hub 
Pulling from library/hello-world

контейнер запустился и выполнил вывод принта и закрылся

<img src="../../../assets/Снимок2026-05-0820.07.54.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />


--------


###### тперь нужно ставить   GitLab Runner 

```shell
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

----------

скрипт добавляет официальный репозиторий GitLab Runner в систему и устанавливает его
После этого можно будет установить сам пакет gitlab-runner
```

теперь нужно установить сам GitLab Runner - это та самая прога которая будет забирать задания из GitLab и выполнять их в Docker-контейнерах

```shell
sudo apt install -y gitlab-runner

встала версия  18.11.2 - ОТЛИЧНО!

```

теперь нужно привязать этот ранер к самому GITlab (всмысле - в браузере зайти и получить токен для CI/CD → Runners)

иду на https://gitlab.com/...
настройки - > CI/CD -> runners -> создать новый раннер

вот так настраиваю
(делаю ранер этот конкретно под проект)

<img src="../../../assets/Снимок2026-05-0820.32.00.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




при создании получаю команду + токен для регистрации этого ранера на своем сервере

```shell
gitlab-runner register  --url https://gitlab.com  --token glrt-GDsUwK-Q1XoVHMxlитутатокенитутатокенитутатокенитутатокен.01.1тутатокен

--------------------------------------------------------------

вижу - ранер зереган успешно!

$ sudo gitlab-runner register --url https://gitlab.com --token glrt-GDsUwK-Q1XoVHMxlитутатокенитутатокенитутатокенитутатокен.01.1тутатокен
Runtime platform                                    arch=amd64 os=linux pid=7351 revision=68229485 version=18.11.2
Running in system-mode.

Enter the GitLab instance URL (for example, https://gitlab.com/):
[https://gitlab.com]:
Verifying runner... is valid                        correlation_id=9f89896dde04e924-DME runner=GDsUwK-Q1 runner_name=vm-f7081a
Enter a name for the runner. This is stored only in the local config.toml file:
[vm-f7081a]:
Enter an executor: shell, custom, instance, docker-windows, ssh, virtualbox, docker, docker-autoscaler, docker+machine, kubernetes, parallels:
docker
Enter the default Docker image (for example, ruby:3.3):
alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
$
```

сейчас нужно понять, работает ли ранер и коннектится ли он с самим GitLab


```shell
sudo gitlab-runner run

--------------

Запускает раннер в режиме отладки (прямо в терминале). Мы увидим, как раннер подключается к GitLab и ждёт задания

Важно: Раннер будет работать, пока открыт этот терминал. Для постоянной работы нужно установить его как сервис, но для теста пока так нормально

```

все отлично - ранер запущен и ждет команды с gitlab
((Предупреждение про `request_concurrency` не критично для тестов))

<img src="../../../assets/Снимок2026-05-0820.39.13.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

(позже нужно будет запустить раненер как постоянный процесс - постоянно как сервис!)

---------

пока терминал висит с работающим раннером

иду в браузер в настройки gitlab - CI/CD - runners -
и вижу = что статус ONLINE

<img src="../../../assets/Снимок2026-05-0820.42.01.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

все отлично!!
раннер в статусе **Online**, с тегами `ios-sast` и `docker`, привязан к моему проекту. Теперь можно писать пайплайны


---------
----------
---------

ставлю GitLab Runner как системный сервис

убиваю процесс - если был запущен Ctrl + C

ставлю как системный сервис (ЭТО ЕСЛИ ВДРУГ НЕ УСТАНОВЛЕН, у меня по умолчанию уже стоял)

```shell
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner

Устанавливает раннер как системный сервис, который автоматически запускается при старте сервера и работает в фоне.


---------

🟢 далее 

sudo systemctl start gitlab-runner


Запускает раннер в фоновом режиме как системный сервис

запустил - вывод пустой + на сайте gitlab  высвечивается статус online!

-----------

теперь обязательно делаю ему автозаупуск

sudo systemctl enable gitlab-runner

--------


и в конце концов - проверю статус раннера

sudo systemctl status gitlab-runner

все отлично, и есть небольшое предупреждение 
WARNING: CONFIGURATION: Long polling issues detected
но оно пока что не критично!
```


----------

## пайплайн


### в самом проекте (ios приложение) - нужно добавить файл .gitlab-ci.yml   

Сейчас раннер настроен на **локальный Docker**, но без указания конкретного образа. При создании пайплайна нужно будет явно указывать Docker-образ. 

в файле .gitlab-ci.yml    ios приложения - пишу тестовый пайплайн
```js 
stages:
  - test

test-job:
  stage: test
  tags:
    - docker
  image: alpine:latest
  script:
    - echo "Hello from runner!"

```

и теперь нужно закомитить!

------

запушил файл этот

и гитлаб просит меня залогиниться у них
(то есть ошибка не в пайплайне - а в верификации)

<img src="../../../assets/Снимок2026-05-1012.04.55.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

и проблема тут не в этом, а в том, что он просит или номер телефона - или номер карты! но при этом - ни РОССИЙСКИЙ ТЕЛЕФОН ни РФ карту не принимает он, поэтому пошел этот GITLAB    НАКУЙ!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

----------

----------

И тут я могу либо переключиться на GitHub Actions
либо на альтернативу от яндекса

-----






## GITLAB - не работает для РОССИИ 


🟢 ❌ 🟢 ❌ 🟢    ❌ 🟢 ❌ 🟢 ❌ 🟢 ❌ 🟢    ❌ 🟢 ❌ 🟢 ❌ 🟢    ❌ 🟢 ❌ 🟢 ❌ 




# далее - перехожу на GitHub Actions


-----

сперва остановлю гит-лаб ранер который у меня на сервере работает  


```shell
подрубаюсь 
ssh -i /Users/evgeniy/Desktop/вирт\ МАШИНА/id_rsa-2 user1@87.242.86.39

sudo systemctl stop gitlab-runner
и автозапуск уберу
sudo systemctl disable gitlab-runner

```

-----


включаю раннер в нужном мне репозитории

<img src="../../../assets/Снимок2026-05-1012.53.52.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />


далее:
линукс
х64

далее выполняю поочередно команды на моем сервере, которые там прописаны (при создании раннера)!


```shell
# создам папочку
mkdir actions-runner && cd actions-runner

#качаю раннер
curl -o actions-runner-linux-x64-2.334.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

# проверю хеш (почему бы и нет)
echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" | shasum -a 256 -c

# распаковка 
tar xzf ./actions-runner-linux-x64-2.334.0.tar.gz



## ЗАПУСК
./config.sh --url https://github.com/EV9EN1Y/meetway --token A6ALHBSPFCZ7JLIBMT3RZOLKABD46

вижу

```

<img src="../../../assets/Снимок2026-05-1013.03.39.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

------

запустим для теста!
```js
./run.sh

ВИЖУ - ВСЕ ОКЕЙ !  РАННЕР ПОДКЛЮЧЕН И ЖДЕТ ЗАДАНИЙ!!!

√ Connected to GitHub

Current runner version: '2.334.0'
2026-05-10 08:04:39Z: Listening for Jobs
```

-----------

запущу теперь его как сервис (постоянно - в фоне)

```shell
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

--------

в корне проекта - создам файл пайплайна
**`Cmd + Shift + .`** - показывать скрытые папки

```q
mkdir -p .github/workflows
nano .github/workflows/test.yml

## с содержимым

name: Test Runner

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Say hello
        run: echo "Hello from Cloud.ru server!"
```

пушу этот файл

и захожу в гитхаб - и вижу там во вкладке  Actions
что тест отработал полностью!

<img src="../../../assets/Снимок2026-05-1013.32.46.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

в паплайне написано у меня  - показать строчку
   run: echo "Hello from Cloud.ru server!"

и в результатах теста я вижу эту строку - значит тест полностью выполнен!!!
ура!


<img src="../../../assets/Снимоr2026-05-1013.33.03.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

------------

### ✅ итого  - получил полностью работающую цепочку CI/CD

## ### Архитектура решения

- **Облачный сервер ([Cloud.ru](https://cloud.ru/))** - выделенная виртуальная машина с Ubuntu 22.04 (2 vCPU, 4 ГБ RAM, 30 ГБ NVMe), на которой развёрнуто окружение для выполнения CI/CD-задач
    
- **Self-hosted runner GitHub Actions** - агент, установленный на сервере, который принимает задания от GitHub, изолирует их в Docker-контейнере и возвращает результат
    
- **Репозиторий GitHub** - хранилище кода iOS-приложения и конфигурации пайплайнов (`.github/workflows/*.yml`)
    
- **Docker** - среда изоляции для каждого задания; контейнер создаётся под задачу, выполняет её и уничтожается

### принцип/суть работы

- GitHub выступает как «пульт управления» - хранит код, содержит описание пайплайна, раздаёт задания через файл .yml
    
- Self-hosted runner на Cloud.ru - «исполнитель» - забирает задания, создаёт Docker-контейнер с указанным окружением, выполняет команды, возвращает статус и логи (основа там на сервере работает агент GitHub actions)
    
- Runner **не выполняет код напрямую** на хосте - каждое задание запускается в изолированном контейнере (`privileged = false`), что соответствует базовым принципам безопасности!!!

--------

следующий шаг - добавлять в пайплайн различные статические анализаторы Semgrep, SwiftLint, MobSF и др

------

### что касается безопасности Self-hosted 

```q
Каждое CI/CD‑задание выполняется в новом Docker-контейнере. Этот контейнер не имеет доступа к ресурсам хоста (сервера), а также не может запускать свои собственные привилегированные операции внутри контейнера

Если один пайплайн скомпрометирован (например, злоумышленник внедрит вредоносный код в репозиторий), это не затронет следующий пайплайн

Контейнер не имеет доступа к файловой системе сервера, не может ставить системные пакеты, менять настройки firewalld или systemd

Нет возможности «вылезти» из контейнерa
`privileged = false` отключает доступ к устройствам хоста, сокету Docker и другим низкоуровневым механизмам
Я ЖЕ  -- использовал **Docker executor** без дополнительных параметров, и  он **и так работает в не-привилегированном режиме по умолчанию**
```

-----------




этот код безопасный

```q
jobs:
  test:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```
Контейнер не может:

- монтировать диски хоста;
- запускать `sudo` внутри (если нет специальной настройки);
- влиять на систему






вот так было бы опасно = полный доступ к хосту
```q
container:
  image: alpine:latest
  options: --privileged
```


при использовании GitHub Actions в режиме self-hosted runner без явного указания `container: options: --privileged` контейнеры запускаются в не-привилегированном режиме. Это изолирует CI/CD-задачи от хостовой системы и предотвращает эскалацию привилегий при выполнении потенциально вредоносного кода

--------------------


пример pipeline_(workflow

```q
name: Мой пайплайн           # Название workflow

on: push                     # Триггер: при пуше

jobs:                        # Список заданий

  test-job:                  # Одно задание
  
    runs-on: self-hosted     # Раннер
    
    steps:                   # Шаги внутри задания
    
      - name: Checkout       # Шаг 1
        uses: actions/checkout@v4
        
      - name: Run script     # Шаг 2
        run: echo "Hello"
```


---------

## усиливаю безопасность 
