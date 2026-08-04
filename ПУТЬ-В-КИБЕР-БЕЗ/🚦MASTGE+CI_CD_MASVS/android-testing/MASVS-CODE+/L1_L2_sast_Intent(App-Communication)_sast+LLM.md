# MASTG-TEST-0372: Implicit Intents Used for Internal App Communication
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0372/

поиск интентов

✅✅✅   ОБА ТЕСТА СРАЗУ
# MASTG-TEST-0374: References to Implicit Intents Carrying Sensitive Extras
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0374/

то какие данные используются в интентах

------


 **Использует ли приложение неявные (implicit) интенты для внутренней коммуникации**. Это может позволить другому приложению перехватить (hijack) этот интент и получить данные или выполнить действие от имени твоего приложения

------------------------------

Интенты - это сообщения, которые компоненты Android (Activity, Service, BroadcastReceiver) используют для общения друг с другом. Интенты бывают двух типов:

- **Явные (explicit):** Указывают конкретный компонент (например, `new Intent(this, MainActivity.class)`). Они безопасны, так как всегда доставляются только в указанное место
   
- **Неявные (implicit):** Не указывают конкретный компонент, а только действие (`ACTION_VIEW`, `ACTION_SEND`). Android сам выбирает, какое приложение может обработать этот интент
   
Если приложение использует неявный интент для отправки данных внутри себя (например, для открытия своей же Activity), то другое приложение может создать такое же `intent-filter` и перехватить это сообщение, получив доступ к данным или выполнив действие от имени твоего приложения

------------------------------

интерпритация 


- **  
    Тест пройден:** Все интенты для внутренней коммуникации явные (указывают конкретный компонент).
    
- **Тест провален:** Найден неявный интент, используемый для внутренней коммуникации (без указания конкретного компонента или пакета).

-----


чек лист поиска

```q
new Intent( – поиск создания интентов. 

Intent() – поиск конструктора без параметров. 

setAction( – метод, который задает действие для интента (признак неявного интента). 

startActivity – отправка интента для запуска Activity. 

startActivityForResult – отправка интента с ожиданием результата. 

startService – отправка интента для запуска сервиса. 

bindService – отправка интента для привязки к сервису. 

sendBroadcast – отправка широковещательного интента. 

launch (в контексте ActivityResultLauncher)
```

##  Результат анализа 

### Explicit Intents (явные - безопасные)

Все переходы между Activity внутри приложения - **явные**, с указанием конкретного класса:

| Файл | Intent | Назначение | Безопасность |
|---|---|---|---|
| `YandexAuthWebViewActivity.java:244` | `Intent(this, MainActivity.class)` | Переход после авторизации | ✅ Explicit |
| `YandexAuthWebViewActivity.java:266` | `Intent(this, MainActivity.class)` | Закрытие WebView | ✅ Explicit |
| `MainActivity.java:379` | `Intent(this, AuthRedirectActivity.class)` | Запуск редиректа | ✅ Explicit |
| `AuthRedirectActivity.java:152` | `Intent(this, MainActivity.class)` | Возврат в Main | ✅ Explicit |
| `NotificationHelper.java:43` | `Intent(context, MainActivity.class)` | PendingIntent нотификации | ✅ Explicit |
| `MeetWayMessagingService.java:142-153` | `Intent(this, MainActivity/NotificationActionReceiver)` | Нотификации | ✅ Explicit |
| `MeetWayMessagingService.java:310` | `Intent(context, MainActivity.class)` | Reply Intent | ✅ Explicit |
| `MeetWayApp.java:125` | `Intent(this, MainActivity.class)` | Старт приложения | ✅ Explicit |

### Implicit Intents для внешних действий (допустимо)

| Файл | Intent | Назначение | Безопасность |
|---|---|---|---|
| `SettingScreenKt.java:68` | `ACTION_SEND` + `createChooser()` | Экспорт данных – пользователь сам выбирает приёмник | ⚠️ Нормально |
| `RegistrationScreenKt.java:1135` | `ACTION_VIEW` + URL | Открытие ссылки в браузере | ⚠️ Нормально |
| `LentaMainViewScreenKt.java:3277` | `APPLICATION_DETAILS_SETTINGS` | Открытие настроек приложения | ⚠️ Нормально |

### Проблемное место: `sendAppBroadcast()`

**Файл:** `MainActivity.java:1169-1182`

```java
public final void sendAppBroadcast(String action, String chatId, String otherUserId) {
    Intent intent = new Intent(action);           // ← implicit! нет компонента
    intent.putExtra("chatId", chatId);            // ← данные
    intent.putExtra("otherUserID", otherUserId);  // ← данные
    applicationContext.sendBroadcast(intent);     // ← broadcast без permission
}
```

**Принимается ли кем-то?** Нет - в приложении **нет ни одного `registerReceiver()`** ни в манифесте, ни динамически. Broadcast'ы уходят в пустоту

**Какие данные передаются:**

| Action                                        | Данные                  |
| --------------------------------------------- | ----------------------- |
| `com.evgeniy.meetway.APP_ACTIVE`              | –                       |
| `com.evgeniy.meetway.APP_DID_BECOME_ACTIVE`   | –                       |
| `com.evgeniy.meetway.PUSH_MESSAGE`            | `chatId`                |
| `com.evgeniy.meetway.UPDATE_CHAT_FROM_PUSH`   | `chatId`, `otherUserID` |
| `com.evgeniy.meetway.OPEN_CHAT`               | `chatId`, `otherUserID` |
| `com.evgeniy.meetway.NAVIGATE_TO_WELCOME`     | –                       |
| `com.evgeniy.meetway.NAVIGATE_TO_FIRST_SCRIN` | –                       |

### BroadcastReceiver

| Компонент | exported | Безопасность |
|---|---|---|
| `NotificationActionReceiver` (нотификации) | `false` ✅ | Надёжно – только внутренние вызовы с explicit Intent |

###  Итоговая таблица

| Паттерн | Explicit | Implicit (внешний) | Implicit (внутренний) | Риск |
|---|---|---|---|---|
| Activity → Activity | ✅ Все | ❌ Нет | ❌ Нет | ✅ Низкий |
| Intent → Browser/Share | – | ✅ 3 шт | – | ⚠️ Приемлемо |
| `sendBroadcast()` | `NotificationActionReceiver` | – | ⚠️ `sendAppBroadcast()` | ⚠️ Средний |

### Замечание

`sendAppBroadcast()` использует неявные интенты для внутренней коммуникации – это **нарушение best practices** MASTG-TEST-0372. Однако приёмников для этих broadcast'ов в приложении нет (dead code или незавершённая реализация)

Данные `chatId` и `otherUserID` не являются критически чувствительными (chatId = ID диалога, otherUserID = ID собеседника)

---

##  Вывод: WARNING 

тест не пройден!!
но по сути, данные  chatId и otherUserID - это открытые безопасные данные!
тест технически не пройден, но фактически - ничего страшного в этом нет!
просто - в будущем нужно будет это аккуратно исправить , чтобы на будущее никто не накосячил, скопировав данный метод , но с другими данными уже!

**Рекомендация:**
- Заменить `new Intent(action)` → `new Intent(action).setPackage(getPackageName())`
- Или использовать `LocalBroadcastManager`
- Или удалить мёртвый код, если broadcast'ы нигде не принимаются

**Фактический риск:** низкий (нет реального приёмника, данные низкой чувствительности)

