# MASTG-TEST-0337: References to Object Deserialization of Untrusted Data
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0337/

----

тест - проверяет, **не десериализует ли приложение данные из ненадежных источников** (например, из `Intent`'ов других приложений, из сетевых ответов или файлов) без проверки их типа или содержимого

Object Serialization (Java-сериализация) --- это механизм, который позволяет сохранять состояние объекта в байтовый поток и восстанавливать его обратно. Если приложение принимает откуда-то сериализованные данные и десериализует их без проверки, злоумышленник может подсунуть специально сформированный объект, который при десериализации выполнит вредоносный код или изменит поведение приложения. Это классическая уязвимость, известная как **небезопасная десериализация**

---

чек для поиска

1. `ObjectInputStream` – класс для десериализации
    
2. `readObject` – метод десериализации
    
3. `Serializable` – интерфейс, делающий класс сериализуемым
    
4. `Parcelable` – Android-специфичный механизм (более безопасный, но стоит проверить)
    

**Методы получения данных из Intent и Bundle:**

5. `getSerializableExtra` – получение сериализованных данных из `Intent`
    
6. `getSerializable` – получение сериализованных данных из `Bundle`
    
7. `getParcelableExtra` – получение Parcelable данных из `Intent`
    
8. `getParcelable` – получение Parcelable данных из `Bundle`
    

**Дополнительные сигнатуры:**

9. `readResolve` – метод, который может быть вызван при десериализации
    
10. `readExternal` – метод для десериализации в `Externalizable`

Если данные приходят из `Intent`, `Bundle`, сетевого ответа или файла, который может быть изменен извне – это потенциальная уязвимость.

Если ты находишь `getSerializableExtra` или `getSerializable`, это тоже повод проверить, откуда приходят данные и проверяются ли они перед использованием


------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```


```q
ObjectInputStream  - 50+ совпадений
readObject   - 50+ совпадений 
Serializable   - 50+ совпадений 
Parcelable  - 50+ совпадений 

getSerializableExtra - 6 совпадений 
getSerializable - 16 совпадений 
getParcelableExtra - 41 совпадений 
getParcelable  - 50+ совпадений 

readResolve   50+ совпадений 
readExternal 19 совпадений


Если данные приходят из `Intent`, `Bundle`, сетевого ответа или файла, который может быть изменен извне – это потенциальная уязвимость.

Если находишь `getSerializableExtra` или `getSerializable`, это тоже повод проверить, откуда приходят данные и проверяются ли они перед использованием

```

## запускаю анализ находок через LLM OPEN CLAW




### Где найдено (app-код MeetWay vs библиотеки)

| Поисковый паттерн         | В app-коде MeetWay | В библиотеках                   | Риск        |
| ------------------------- | ------------------ | ------------------------------- | ----------- |
| `ObjectInputStream`       | 0                  | 50+ (libs)                      | ❌ Нет в app |
| `readObject`              | 0                  | 50+ (libs)                      | ❌ Нет в app |
| `readExternal`            | 0                  | 19 (libs)                       | ❌ Нет в app |
| `readResolve`             | 0                  | 50+ (libs)                      | ❌ Нет в app |
| `Serializable` implements | 0                  | 50+ (libs)                      | ❌ Нет в app |
| `getSerializableExtra`    | **0**              | 6 (androidx.core, GMS)          | ❌ Нет в app |
| `getSerializable`         | **0**              | 16 (androidx.core, media3, GMS) | ❌ Нет в app |
| `getParcelableExtra`      | **0**              | 41 (libs)                       | ❌ Нет в app |
| `getParcelable`           | **0**              | 50+ (libs)                      | ❌ Нет в app |
| `Parcelable` implements   | 0                  | 50+ (libs)                      | ❌ Нет в app |

### Что использует MeetWay вместо этого

Приложение использует **только безопасное парсинг JSON**:

| Файл | Что использует | Для чего |
|---|---|---|
| `FeedViewModel.java` | `Gson.fromJson()`, `JsonParser` | Парсинг ответов ленты |
| `AuthViewModel.java` | `JsonParser.parseString()` | Парсинг JWT payload |
| `CloudFunctionService.java` | `Gson`, `JSONObject`, `JSONArray` | Парсинг ответов Yandex Cloud Functions |
| `AuthRedirectActivity.java` | `JSONObject` | Парсинг Yandex ID из редиректа |

JSON-парсинг через `Gson` и `org.json` – **не является уязвимостью десериализации**. Злоумышленник не может подсунуть произвольный Java-объект – только JSON-структуры.

###  Итог

Все 50+ вхождений `ObjectInputStream`, `readObject`, `getSerializable` и т.д. - **исключительно в библиотеках** (androidx.core, media3, GMS).

App-код MeetWay **нигде** не использует:
- Java Object Serialization (`ObjectInputStream` / `readObject`)
- `getSerializableExtra()` / `getSerializable()`
- `Externalizable` (`readExternal`)
- Собственные `Parcelable`-классы

---

##  Вывод: тест ПРОЙДЕН

Уязвимость небезопасной десериализации не подтверждена.
Приложение не десериализует Java-объекты из ненадежных источников.
Тест MASTG-TEST-0337 пройден






