# Урок 11. Підтвердження видалення і сповіщення

**Мета:** захистити користувача від випадкового видалення діалогом
підтвердження, а після видалення показати два різні типи повідомлення —
**Snackbar** усередині застосунку і **системне сповіщення** у шторці.

Це перша фіча «після релізу»: застосунок уже випущений, і далі його розвивають
маленькими кроками. Наприкінці уроку піднімемо `versionCode` — саме так виглядає
звичайний цикл оновлення.

## Три способи щось повідомити — і коли який

| Спосіб | Де видно | Для чого |
|---|---|---|
| **Snackbar** | внизу екрана, кілька секунд | результат дії, поки користувач у застосунку |
| **Сповіщення** | шторка системи, лишається | подія, важлива й тоді, коли застосунок закритий |
| **Toast** | сіра плашка поверх усього | застарілий підхід, у Compose майже не потрібен |

Для видалення нотатки в реальному застосунку вистачило б Snackbar (та ще й із
кнопкою «Скасувати»). Системне сповіщення тут — навчальний привід розібрати
канали, дозволи та `NotificationCompat`, бо саме вони потрібні для нагадувань,
завантажень і фонової роботи.

## Крок 1. Нові тексти

Додай у `app/src/main/res/values/strings.xml`:

```xml
    <string name="delete_dialog_title">Delete note?</string>
    <string name="delete_dialog_text">“%1$s” will be removed from the list.</string>
    <string name="action_cancel">Cancel</string>
    <string name="snackbar_deleted">Note deleted</string>
    <string name="notification_deleted_title">Note deleted</string>
    <string name="notification_channel_name">Note actions</string>
    <string name="notification_channel_description">Confirmations of actions on notes</string>
    <string name="action_ok">OK</string>
```

І в `app/src/main/res/values-uk/strings.xml`:

```xml
    <string name="delete_dialog_title">Видалити нотатку?</string>
    <string name="delete_dialog_text">«%1$s» зникне зі списку.</string>
    <string name="action_cancel">Скасувати</string>
    <string name="snackbar_deleted">Нотатку видалено</string>
    <string name="notification_deleted_title">Нотатку видалено</string>
    <string name="notification_channel_name">Дії з нотатками</string>
    <string name="notification_channel_description">Підтвердження дій із нотатками</string>
    <string name="action_ok">Гаразд</string>
```

## Крок 2. Діалог підтвердження

Уся зміна — всередині `NotesScreen.kt`, жодних нових файлів і залежностей.

**2.1.** На початку функції `NotesScreen`, поруч із `draft`, додай стан діалога:

```kotlin
    var noteToDelete by remember { mutableStateOf<Note?>(null) }
```

`Note?` замість `Boolean` — одна змінна відповідає одразу на два питання: «чи
показувати діалог» (не `null`) і «яку саме нотатку видаляємо». Два окремі стани
завжди ризикують розсинхронитись; один — ні.

Тут `remember`, а не `rememberSaveable`: `Note` не вміє зберігатись у `Bundle`.
Практичного значення це майже не має — при повороті телефона діалог просто
закриється, і користувач натисне ще раз.

**2.2.** Кнопка «Видалити» на картці більше не видаляє одразу, а лише відкриває
діалог. У виклику `NoteCard` заміни:

```kotlin
                                onDelete = { noteToDelete = note },
```

**2.3.** У самому кінці тіла `Scaffold` (після блоку `when { … }`, всередині
`Column`) додай сам діалог:

```kotlin
            noteToDelete?.let { note ->
                AlertDialog(
                    onDismissRequest = { noteToDelete = null },
                    title = { Text(stringResource(R.string.delete_dialog_title)) },
                    text = { Text(stringResource(R.string.delete_dialog_text, note.title)) },
                    confirmButton = {
                        TextButton(
                            onClick = {
                                onDelete(note)
                                noteToDelete = null
                            },
                        ) {
                            Text(stringResource(R.string.action_delete))
                        }
                    },
                    dismissButton = {
                        TextButton(onClick = { noteToDelete = null }) {
                            Text(stringResource(R.string.action_cancel))
                        }
                    },
                )
            }
```

**2.4.** Імпорти:

```kotlin
import androidx.compose.material3.AlertDialog
import androidx.compose.runtime.remember
```

Що тут важливо:

- **Діалог — це не «вікно», яке відкривають.** Це частина опису екрана, яка
  існує, поки `noteToDelete != null`. Знову той самий принцип: UI є функцією від
  стану.
- **`onDismissRequest`** спрацьовує при натисканні повз діалог або на «Назад».
  Обнулити стан там **обов'язково**, інакше діалог не закриється.
- **`?.let { note -> … }`** одночасно перевіряє на `null` і дає нам `note` як
  не-nullable значення всередині блоку.

Перевір уже зараз:

```sh
./gradlew installDebug
```

Натискання «Видалити» показує діалог; «Скасувати» лишає нотатку на місці.

## Крок 3. Snackbar після видалення

`Scaffold` уміє показувати Snackbar сам — треба лише дати йому «господаря»
(`SnackbarHost`) і попросити показати текст.

**3.1.** Поруч зі станом діалога:

```kotlin
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
```

**3.2.** Додай параметр у `Scaffold`, поруч із `topBar`:

```kotlin
        snackbarHost = { SnackbarHost(snackbarHostState) },
```

**3.3.** У `confirmButton` діалога, після `onDelete(note)`:

```kotlin
                                scope.launch {
                                    snackbarHostState.showSnackbar(
                                        message = deletedMessage,
                                        actionLabel = okLabel,
                                        duration = SnackbarDuration.Long,
                                    )
                                }
```

**`actionLabel`** — та сама кнопка «OK» на плашці. Натискання на неї закриває
Snackbar одразу, не чекаючи таймера. `SnackbarDuration.Long` — приблизно 10
секунд (є ще `Short` ≈ 4 секунди і `Indefinite` — висіти, поки не натиснуть).

Рядки треба прочитати **до** лямбди — усередині `scope.launch` ми вже поза
`@Composable`-контекстом, і `stringResource` там викликати не можна. Додай поруч
із рештою станів:

```kotlin
    val deletedMessage = stringResource(R.string.snackbar_deleted)
    val okLabel = stringResource(R.string.action_ok)
```

**3.4.** Імпорти:

```kotlin
import androidx.compose.material3.SnackbarDuration
import androidx.compose.material3.SnackbarHost
import androidx.compose.material3.SnackbarHostState
import androidx.compose.runtime.rememberCoroutineScope
import kotlinx.coroutines.launch
```

**3.5.** Знати про запас: `showSnackbar` повертає результат, і за ним видно,
чи натиснув користувач кнопку:

```kotlin
    val result = snackbarHostState.showSnackbar(…)
    if (result == SnackbarResult.ActionPerformed) {
        // натиснули кнопку
    } else {
        // плашка зникла сама або її змахнули
    }
```

Зараз нам байдуже — кнопка просто закриває плашку. Але саме на цьому будують
«Скасувати» замість підтвердження (див. вправу 1 наприкінці уроку).

Якщо хочеться замість напису «OK» хрестик — є готовий варіант:
`withDismissAction = true` у `showSnackbar`. Тоді `actionLabel` не потрібен.

**Чому `scope.launch`.** `showSnackbar` — `suspend`-функція: вона чекає, поки
Snackbar сховається (і повертає, чи натиснули кнопку дії). Викликати її можна
лише з корутини. `rememberCoroutineScope()` дає область, прив'язану до життя
цього екрана: зникне екран — незавершений виклик скасується.

Це той самий механізм, що `viewModelScope` в уроці 8, але для UI-подій.

## Крок 4. Системне сповіщення

Тут з'являється три поняття Android, яких у Compose немає: **канал**, **дозвіл**
і **`NotificationCompat`**.

**4.1. Іконка.** Створи `app/src/main/res/drawable/ic_notification.xml`:

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24"
    android:tint="#FFFFFF">
    <path
        android:fillColor="#FFFFFF"
        android:pathData="M6,4 h8 l4,4 v12 a1,1 0 0,1 -1,1 h-11 a1,1 0 0,1 -1,-1 v-15 a1,1 0 0,1 1,-1 z" />
</vector>
```

Маленька іконка сповіщення показується як **силует**: система бере лише
прозорість, а всі кольори замінює своїм. Тому малюють суцільну білу фігуру на
прозорому фоні — кольорова картинка перетвориться на білу пляму.

**4.2. Дозвіл** у `AndroidManifest.xml`, поруч із `INTERNET`:

```xml
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

**4.3.** Створи `…/androidstarter/notifications/Notifier.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.notifications

import android.Manifest
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.core.content.ContextCompat
import com.webuxmotion.androidstarter.R

object Notifier {

    private const val CHANNEL_ID = "note_actions"
    private var nextNotificationId = 1

    fun ensureChannel(context: Context) {
        val channel = NotificationChannel(
            CHANNEL_ID,
            context.getString(R.string.notification_channel_name),
            NotificationManager.IMPORTANCE_DEFAULT,
        ).apply {
            description = context.getString(R.string.notification_channel_description)
        }
        NotificationManagerCompat.from(context).createNotificationChannel(channel)
    }

    fun hasPermission(context: Context): Boolean {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) return true
        return ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.POST_NOTIFICATIONS,
        ) == PackageManager.PERMISSION_GRANTED
    }

    fun showNoteDeleted(context: Context, noteTitle: String) {
        if (!hasPermission(context)) return

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(context.getString(R.string.notification_deleted_title))
            .setContentText(noteTitle)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(nextNotificationId++, notification)
    }
}
```

Розбір:

- **Канал (`NotificationChannel`)** обов'язковий з Android 8.0. Це «категорія»
  сповіщень, налаштуваннями якої керує **користувач**: він може вимкнути звук
  саме для «Дій з нотатками», не чіпаючи решту. Канал створюють щоразу при
  старті — повторний виклик нічого не ламає, система просто ігнорує його, якщо
  канал уже є. Важливо: **параметри вже створеного каналу з коду не змінити**
  (крім назви й опису) — доведеться робити канал із новим `id`.
- **Дозвіл `POST_NOTIFICATIONS`** з'явився в Android 13 (API 33, `TIRAMISU`).
  На старіших системах його не існує, тому `hasPermission` там завжди повертає
  `true`. Це типова форма «сумісності»: перевіряємо `Build.VERSION.SDK_INT` і
  поводимось по-різному. Наш `minSdk = 26`, тож обидві гілки реальні.
- **`NotificationCompat`** із `androidx.core` замість системного
  `Notification.Builder` — саме він ховає відмінності між версіями Android.
  Нова залежність не потрібна: `androidx.core:core-ktx` у проєкті з першого дня.
- **`nextNotificationId++`** — різні `id` дають кілька сповіщень поруч. Якби
  `id` був однаковий, кожне нове **замінювало** б попереднє (так роблять
  прогрес-бари завантаження).

**4.4. Запитати дозвіл і створити канал.** У `NotesScreen.kt`, поруч із рештою
станів:

```kotlin
    val context = LocalContext.current
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission(),
    ) { /* результат нам не потрібен: без дозволу просто не показуємо сповіщення */ }

    LaunchedEffect(Unit) {
        Notifier.ensureChannel(context)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU &&
            !Notifier.hasPermission(context)
        ) {
            permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
```

**4.5.** У `confirmButton` діалога, після `onDelete(note)`:

```kotlin
                                Notifier.showNoteDeleted(context, note.title)
```

**4.6.** Імпорти:

```kotlin
import android.Manifest
import android.os.Build
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.ui.platform.LocalContext
import com.webuxmotion.androidstarter.notifications.Notifier
```

Два нових інструменти Compose:

- **`LaunchedEffect(Unit)`** виконує блок **один раз** при появі екрана (і ще
  раз, якщо зміниться ключ у дужках). Це правильне місце для дій, які не можна
  робити в тілі `@Composable`: створити канал, запитати дозвіл, запустити
  завантаження. Без нього діалог дозволу вискакував би на кожну рекомпозицію.
- **`rememberLauncherForActivityResult`** — сучасний спосіб попросити систему
  про щось і дочекатись відповіді: дозвіл, вибір фото, зйомка. Старі
  `onRequestPermissionsResult` і `startActivityForResult` більше не потрібні.

Заготовка `NotesScreen` після всіх правок виглядає так (лише порядок, без тіла):

```kotlin
fun NotesScreen(...) {
    var draft by rememberSaveable { … }
    var noteToDelete by remember { mutableStateOf<Note?>(null) }
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    val deletedMessage = stringResource(R.string.snackbar_deleted)
    val okLabel = stringResource(R.string.action_ok)
    val context = LocalContext.current
    val permissionLauncher = rememberLauncherForActivityResult(…) { }

    LaunchedEffect(Unit) { … }         // канал + запит дозволу

    Scaffold(topBar = { … }, snackbarHost = { … }) { innerPadding ->
        Column(…) {
            Row(…) { /* поле вводу + Додати */ }
            when { … }                 // завантаження / помилка / список
            noteToDelete?.let { … }    // діалог; у confirmButton — видалення,
                                       // сповіщення і Snackbar
        }
    }
}
```

## Крок 5. Кнопка «OK» на сповіщенні

Сповіщення зараз висить у шторці, поки його не змахнути. Додамо кнопку, яка
закриває його одним натисканням.

Тут з'являється принципово нова річ: **код має виконатись тоді, коли твій
застосунок може бути вже закритий**. Лямбду в шторку не передаси — тому Android
використовує `PendingIntent` («дозвіл системі виконати дію від твого імені
пізніше») і `BroadcastReceiver` («приймач», що цю дію виконує).

Текст кнопки — той самий `action_ok`, який ми вже додали в кроці 1.

**5.1. Приймач.** Створи
`…/androidstarter/notifications/NotificationDismissReceiver.kt` —
**повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.notifications

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationManagerCompat

class NotificationDismissReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        val notificationId = intent.getIntExtra(Notifier.EXTRA_NOTIFICATION_ID, -1)
        if (notificationId != -1) {
            NotificationManagerCompat.from(context).cancel(notificationId)
        }
    }
}
```

`BroadcastReceiver` — компонент, який система вміє запускати сама, навіть якщо
процес застосунку вбитий. Тому в `onReceive` не можна робити нічого довгого:
кілька секунд роботи — і система вб'є процес. Для тривалих справ звідси
запускають `WorkManager`.

**5.2. Оголоси приймач у маніфесті.** У `AndroidManifest.xml`, усередині
`<application>`, після `<activity>`:

```xml
        <receiver
            android:name=".notifications.NotificationDismissReceiver"
            android:exported="false" />
```

Компонент, якого немає в маніфесті, для системи не існує — жодної помилки
компіляції не буде, кнопка просто мовчатиме. `android:exported="false"` означає
«запускати може тільки мій застосунок»: чужим програмам нема чого гасити наші
сповіщення.

**5.3. Додай кнопку в сповіщення.** Заміни в `Notifier.kt` функцію
`showNoteDeleted` і додай константу:

```kotlin
    const val EXTRA_NOTIFICATION_ID = "notification_id"

    fun showNoteDeleted(context: Context, noteTitle: String) {
        if (!hasPermission(context)) return

        val notificationId = nextNotificationId++

        val dismissIntent = Intent(context, NotificationDismissReceiver::class.java)
            .putExtra(EXTRA_NOTIFICATION_ID, notificationId)

        val dismissPendingIntent = PendingIntent.getBroadcast(
            context,
            notificationId,
            dismissIntent,
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
        )

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(context.getString(R.string.notification_deleted_title))
            .setContentText(noteTitle)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)
            .addAction(
                R.drawable.ic_notification,
                context.getString(R.string.action_ok),
                dismissPendingIntent,
            )
            .build()

        NotificationManagerCompat.from(context).notify(notificationId, notification)
    }
```

Нові імпорти в `Notifier.kt`:

```kotlin
import android.app.PendingIntent
import android.content.Intent
```

Зверни увагу: `nextNotificationId++` тепер зчитується **один раз** у змінну.
Раніше він стояв прямо у виклику `notify(...)`, а тепер той самий номер потрібен
у трьох місцях — інакше кнопка гасила б не те сповіщення.

**5.4. Три речі, на яких тут спотикаються:**

- **`FLAG_IMMUTABLE` обов'язковий** з Android 12 (API 31). Без одного з прапорців
  `IMMUTABLE`/`MUTABLE` застосунок впаде з `IllegalArgumentException` прямо в
  момент показу сповіщення. «Immutable» означає, що отримувач не може дописати
  щось у наш `Intent` — це саме те, що треба.
- **`requestCode` має бути унікальним** (у нас це `notificationId`). Система
  вважає два `PendingIntent` однаковими, якщо збігаються отримувач і
  `requestCode`, і **не дивиться на extras**. Передаси всюди `0` — і кнопка на
  третьому сповіщенні закриє перше. `FLAG_UPDATE_CURRENT` додатково каже
  «онови дані, якщо такий уже існує».
- **Іконка в `addAction`** із Android 7 не показується — лишилась у сигнатурі
  заради сумісності зі старими системами й годинниками Wear. Передати туди
  `0` теж можна.

Так само влаштована й будь-яка інша кнопка на сповіщенні: «Відповісти»,
«Відкласти», «Зупинити». Різниця лише в тому, що виконує приймач.

**5.5. Звірка.** `Notifier.kt` за урок змінювався двічі, тому ось його
**фінальний вміст** — можна просто порівняти зі своїм:

```kotlin
package com.webuxmotion.androidstarter.notifications

import android.Manifest
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.core.content.ContextCompat
import com.webuxmotion.androidstarter.R

object Notifier {

    const val EXTRA_NOTIFICATION_ID = "notification_id"

    private const val CHANNEL_ID = "note_actions"
    private var nextNotificationId = 1

    fun ensureChannel(context: Context) {
        val channel = NotificationChannel(
            CHANNEL_ID,
            context.getString(R.string.notification_channel_name),
            NotificationManager.IMPORTANCE_DEFAULT,
        ).apply {
            description = context.getString(R.string.notification_channel_description)
        }
        NotificationManagerCompat.from(context).createNotificationChannel(channel)
    }

    fun hasPermission(context: Context): Boolean {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) return true
        return ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.POST_NOTIFICATIONS,
        ) == PackageManager.PERMISSION_GRANTED
    }

    fun showNoteDeleted(context: Context, noteTitle: String) {
        if (!hasPermission(context)) return

        val notificationId = nextNotificationId++

        val dismissIntent = Intent(context, NotificationDismissReceiver::class.java)
            .putExtra(EXTRA_NOTIFICATION_ID, notificationId)

        val dismissPendingIntent = PendingIntent.getBroadcast(
            context,
            notificationId,
            dismissIntent,
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
        )

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(context.getString(R.string.notification_deleted_title))
            .setContentText(noteTitle)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)
            .addAction(
                R.drawable.ic_notification,
                context.getString(R.string.action_ok),
                dismissPendingIntent,
            )
            .build()

        NotificationManagerCompat.from(context).notify(notificationId, notification)
    }
}
```

І `confirmButton` діалога в підсумку виглядає так:

```kotlin
                    confirmButton = {
                        TextButton(
                            onClick = {
                                onDelete(note)
                                noteToDelete = null
                                Notifier.showNoteDeleted(context, note.title)
                                scope.launch {
                                    snackbarHostState.showSnackbar(
                                        message = deletedMessage,
                                        actionLabel = okLabel,
                                        duration = SnackbarDuration.Long,
                                    )
                                }
                            },
                        ) {
                            Text(stringResource(R.string.action_delete))
                        }
                    },
```

Порядок тут не випадковий: спершу видаляємо, потім закриваємо діалог, і лише
після цього показуємо повідомлення. Snackbar запускається останнім, бо
`scope.launch` не блокує — решта рядків виконалась би однаково, але читати
зверху вниз так зрозуміліше.

## Крок 6. Перевірити

```sh
./gradlew installDebug
```

Сценарій перевірки:

1. Натисни «Видалити» на картці → з'явився діалог із назвою нотатки.
2. «Скасувати» → нотатка на місці, нічого не сталося.
3. «Видалити» → нотатка зникла, внизу спливла плашка з кнопкою «OK».
   Натисни «OK» → плашка зникає одразу, не чекаючи таймера.
4. Опусти шторку → там **окреме** сповіщення «Нотатку видалено» з назвою і
   своєю кнопкою «OK». Це вже інший механізм: плашка живе всередині екрана,
   сповіщення — у системі.
5. Натисни на кнопку → сповіщення зникає.
6. Видали ще дві нотатки й закрий сповіщення **середнє** — решта дві мають
   лишитись на місці. Якщо зникає не те, шукай помилку в `requestCode`.

Перевірити, що приймач узагалі зареєстрований:

```sh
adb shell dumpsys package com.webuxmotion.androidstarter | grep -i receiver -A3
```

Подивитись, що система думає про твої сповіщення:

```sh
adb shell dumpsys notification --noredact | grep -A3 androidstarter | head -20
```

На Android 13+ при першому запуску вискочить запит дозволу. Щоб пройти цей шлях
ще раз, скинь дозволи:

```sh
adb shell pm reset-permissions
```

## Крок 7. Підняти версію і зафіксувати

Це оновлення вже випущеного застосунку, тож у `app/build.gradle.kts`:

```kotlin
        versionCode = 3
        versionName = "1.1"
```

```sh
git add -A
git commit -m "Урок 11: підтвердження видалення і сповіщення"
```

## Що змінилось у git

```
app/src/main/AndroidManifest.xml                    # +дозвіл, +<receiver>
app/src/main/res/drawable/ic_notification.xml       (новий)
app/src/main/res/values/strings.xml                 # +8 рядків
app/src/main/res/values-uk/strings.xml              # +8 рядків
app/src/main/java/.../notifications/Notifier.kt                  (новий)
app/src/main/java/.../notifications/NotificationDismissReceiver.kt (новий)
app/src/main/java/.../ui/NotesScreen.kt             (змінений)
app/build.gradle.kts                                # версія
```

**Жодної нової залежності** — і це показово. Діалоги вже є в `material3`,
Snackbar — у `Scaffold`, сповіщення — в `androidx.core`, запит дозволу — в
`activity-compose`. Перш ніж додавати бібліотеку, варто перевірити, чи потрібне
вже не лежить у наявних.

Зате знову змінився **маніфест**, причому двічі й з різних причин: дозвіл
(право турбувати користувача) і `<receiver>` (новий компонент, який система має
вміти запускати сама). Маніфест — це «договір із системою»: усе, що застосунок
просить у неї або пропонує їй, описане там.

`NotesViewModel` не змінився взагалі: логіка видалення була готова, ми лише
додали підтвердження перед нею й повідомлення після.

## Вправа

1. **Скасування замість підтвердження.** Прибери діалог і зроби «м'який»
   варіант: видаляти одразу, а в Snackbar давати кнопку повернення.
   `showSnackbar` повертає результат:

   ```kotlin
   val result = snackbarHostState.showSnackbar(
       message = deletedMessage,
       actionLabel = undoLabel,
       duration = SnackbarDuration.Short,
   )
   if (result == SnackbarResult.ActionPerformed) { /* повернути нотатку */ }
   ```

   Для цього знадобиться метод `restoreNote(note)` у `NotesViewModel`. Саме так
   роблять у Gmail — і це чесніше до користувача, ніж зайве питання.
2. **Клік по сповіщенню.** Зараз воно нікуди не веде. Додай `PendingIntent`, що
   відкриває застосунок:

   ```kotlin
   val intent = Intent(context, MainActivity::class.java)
   val pending = PendingIntent.getActivity(
       context, 0, intent, PendingIntent.FLAG_IMMUTABLE,
   )
   // .setContentIntent(pending)
   ```
3. **Діалог для інших дій.** Винеси `ConfirmDialog(title, text, onConfirm,
   onDismiss)` в окремий файл і перевикористай для кнопки «Очистити все».
4. **Стан діалога вище.** Перенеси `noteToDelete` у `NotesUiState` і
   `NotesViewModel`. Подумай, що це дає (стан переживе поворот) і чим коштує
   (`ViewModel` тепер знає про UI-подробиці).

## Далі

Курс завершено. Що з цього виросло і куди рухатись — у підсумку
[уроку 10](10-release.md#підсумок-курсу).
