# ТЕСТИРОВАНИЕ API В PORT SWIGGER

## ЧТО ЭТО ВООБЩЕ

API это когда бекенд общается с фронтом
REST API самый популярный - там JSON туда-сюда летает
Портосвиггер выкатил тему про то как это говно тестировать на прочность

## ЧЕМ ОТЛИЧАЕТСЯ ОТ ОБЫЧНОГО ВЕБА

Обычный сайт тебе страницы рисует формы кнопки. А API просто данные гоняет. В контексте пентеста это значит:

1. нет этих кнопок интерфейса - только голые запросы
   
2. документация часто либо есть либо нет (и то и другое плохо)
   
3. авторизация через токены JWT или ключи в заголовках
   
4. парамтеров может быть дохера и все они могут быть уязвимы
   

## ГДЕ ИСКАТЬ API В ЦЕЛЕВОЙ СИСТЕМЕ

Самые популярные пути где API обычно обитает:

```
/api  
/v1  
/v2  
/graphql  
/swagger  
/api-docs  
/openapi.json

/swagger/index.html  
/swagger/ui  
/swagger-ui.html  
/swagger.json  
/swagger.yaml  
/api-docs/swagger.json  
/api-docs/swagger.yaml  
/docs  
/documentation  
/api/documentation  
/api/swagger  
/api/swagger-ui  
/api/swagger.json  
/api/swagger.yaml  
/openapi.yaml  
/api/openapi.json  
/api/openapi.yaml  
/v1/swagger.json  
/v2/swagger.json  
/v3/swagger.json

/graphql  
/graphiql  
/graphql/console  
/v1/graphql  
/v2/graphql  
/api/graphql  
/query  
/graphql/query

/api/v1  
/api/v2  
/api/v3  
/api/v4  
/api/v5  
/rest  
/rest/v1  
/rest/v2  
/api/rest  
/service  
/services  
/service/v1  
/services/v2

/admin/api  
/internal/api  
/private/api  
/partner/api  
/partner/v1  
/third-party/api  
/thirdparty/v1  
/backend/api  
/api/admin  
/api/internal  
/api/private

/api/1  
/api/2  
/api/3  
/api/latest  
/api/stable  
/api/beta  
/api/alpha  
/api/dev  
/api/test  
/api/staging

/api/json  
/api/xml  
/api/yaml  
/api/rest/json  
/api/rest/xml  
/api/data  
/api/endpoint  
/api/service  
/api/services  
/api/function  
/api/functions  
/api/method  
/api/methods  
/api/action  
/api/actions

/api/doc  
/api/docs  
/api/documentation  
/api/guide  
/api/reference  
/api/manual  
/api/help  
/api/index.html  
/api/readme  
/api/README

моб

/api/mobile  
/api/app  
/mobile/api  
/app/api  
/api/v1/mobile  
/api/android  
/api/ios  
/client-api  
/mobile-client

вот тут тоже может че-то будет

/robots.txt  
/sitemap.xml  
/.git/  
/backup/  
/phpinfo.php  
/cgi-bin/phpinfo.php  
/debug/  
index.php~  
index.php.bak  
index.php.swp  
index.php.save  
index.php.old  
index.php.orig  
config.php~  
config.php.bak  
.env~  
.env.bak  
.gitignore  
/debug  
/test  
/tests  
/dev  
/develop  
/development  
/stage  
/staging  
/admin/debug  
/api/debug  
/console/  
/web-console/  
/admin/  
/backup/  
/backups/  
/temp/  
/tmp/  
/logs/  
/log/  
/private/  
/hidden/  
/secret/  
/internal/  
/restricted/  
/secure/  
/protected/  
/uploads/  
/files/  
/downloads/  
/docs/  
/documentation/  
/api/docs/  
/swagger/  
/swagger-ui/  
/graphql/console/  
/.env  
/.htaccess  
/.htpasswd  
/WEB-INF/  
/WEB-INF/web.xml  
/META-INF/  
/META-INF/context.xml  
/server-status  
/server-info  
/config.php  
/config.xml  
/config.json  
/configuration.php  
/settings.php  
/wp-config.php  
/app.config  
/application.properties  
/application.yml  
/database.yml  
/error_log  
/error.log  
/access_log  
/access.log  
/debug.log  
/application.log  
/server.log  
/catalina.out
```

Тыкай эти пути через intruder или просто в браузере открывай
🏆Если повезет найдешь сваггер документацию - там вообще всё разложено какие ручки есть какие парамтеры принимают

В современных приложухах часто юзают GraphQL
Если хочешь глянуть на лабы GraphQL то вот тут [[0-theory-GraphQL-API]]
Это такая штука где ты сам выбираешь какие поля хочешь получить. GraphQL пентестить отдельная песня там могут быть интроспективные запросы которые всю схему сольют

## ОСНОВНЫЕ ДЫРЫ В API

> если API не требует авторизацию вообще - это не бага а просто трынздец

### 1 СЛОМАННАЯ АВТОРИЗАЦИЯ

Самое частое
Проверяешь ручку которая должна быть доступна только админу
Пробуешь дернуть её с токеном обычного юзера
Если работает - вот тебе уязвимость

Надо проверять все уровни доступа:

ручки для админа  
ручки для менеджера  
ручки для юзера  
ручки для неавторизованного

Если где-то доступ работает не по роли - пишем в отчет 🏆

### 2 IDOR В API

GET /api/user/1337

Пробуешь подставить другой айди
Если API отдает данные другого юзера - классический IDOR
В API этого добра много потому что разрабы часто думают раз API то никто не будет просто айди перебирать

Надо пробовать не только GET но и PUT POST DELETE
Может быть такое что ты можешь изменить чужие данные или удалить их, короче IDOR одним словом

### 3 МАССОВОЕ ПРИСВОЕНИЕ

Там где есть PUT или PATCH запросы на обновление данных
Отправляем лишние парамтеры типа role: admin. Если API применяет их - трындец системе, но конечно, подобрать эти ручки вот так - просто так - та еще задачка..

```http
POST /api/user/update  
Content-Type: application/json

{  
name: куй,  
email: pidor@mail.ru  
}
```

 добавляешь в запрос role: admin и если роль меняется - ты админ

### 4 НЕОГРАНИЧЕННАЯ ЧАСТОТА ЗАПРОСОВ

API которые не ограничивают рейты позволяют долбить себя до усрачки
Это дает:

брутфорс паролей  
перебор айди  
ддос по-тихому

Проверяется просто: отправляешь кучу запросов подряд
Если банят - норм
Если нет - уязвимость

### 5 НЕПРАВИЛЬНАЯ ОБРАБОТКА HTTP МЕТОДОВ

Иногда работает так:

GET /api/news/1 - отдает новость  
DELETE /api/news/1 - удаляет новость

А если отправить DELETE с токеном обычного юзера? Если работает - то это плохо для системы (злоумышленнику хорошо)

поэтому: пробуем все методы на всех ручках. Часто забывают настроить права на нестандартные методы

### 6 ИНЪЕКЦИИ

SQL инъекции никуда не делись. JSON передаешь а внутри sql код
Бекенд может его скушать если криво написан

Пробуй стандартные пейлоады в поля:

```c
' OR 1=1--  
'; DROP TABLE users--  
' UNION SELECT password FROM users--
```
### 7 УТЕЧКА ДАННЫХ В ОТВЕТАХ

API может отдавать слишком много инфы. Например при запросе юзера приходит его хеш пароля или внутренние айдишники. Или поля которые не должны быть видны типа isAdmin

Ответ сервера надо всегда разглядывать внимательно. Много лишнего добра может быть.

## ИНСТРУМЕНТЫ ДЛЯ ТЕСТИРОВАНИЯ

Бурп сам норм работает. В репитере дергаешь ручки меняешь парамтеры.

Постман для документирования запросов норм

Для GraphQL есть отдельные плагины в бурпе InQL

Для автоматизированного поиска - скрипты на питоне 


==ВАЖНО: если есть сваггер документация это не значит что система безопасна
Часто сваггер показывает одно а по факту работает другое и разрабы забыли про это другое==

#### Короче говоря - все тоже самое - что и большая часть обычных уязмостей классических, но только - тут всего по чуть-чуть (видимо , мне нужно было эту тему решать в самом начале, а не под конец, когда уже почти все лабы решил на портсвиге)