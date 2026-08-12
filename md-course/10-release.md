# Урок 10. Реліз: іконка, назва, підписаний APK

**Мета:** довести застосунок до вигляду, у якому його не соромно комусь
віддати: власна іконка, тексти в ресурсах, версія, зменшений і підписаний APK.

## Крок 1. Тексти — у ресурси

Досі всі написи були зашиті в коді. Для релізу так не роблять: тексти в
ресурсах можна перекласти, не чіпаючи жодного рядка Kotlin.

`app/src/main/res/values/strings.xml` — **повний вміст**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Android Starter</string>
    <string name="notes_title">Notes (%1$d)</string>
    <string name="action_refresh">Refresh</string>
    <string name="action_theme">Theme</string>
    <string name="action_add">Add</string>
    <string name="new_note_label">New note</string>
    <string name="action_delete">Delete</string>
    <string name="action_retry">Try again</string>
    <string name="error_prefix">Error: %1$s</string>
    <string name="back">Back</string>
</resources>
```

Створи `app/src/main/res/values-uk/strings.xml` — **повний вміст**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Нотатки</string>
    <string name="notes_title">Нотатки (%1$d)</string>
    <string name="action_refresh">Оновити</string>
    <string name="action_theme">Тема</string>
    <string name="action_add">Додати</string>
    <string name="new_note_label">Нова нотатка</string>
    <string name="action_delete">Видалити</string>
    <string name="action_retry">Спробувати ще раз</string>
    <string name="error_prefix">Помилка: %1$s</string>
    <string name="back">Назад</string>
</resources>
```

Суфікс `-uk` у назві папки — **кваліфікатор ресурсів**. Система сама обирає
папку під мову телефона, а `values/` лишається запасним варіантом. Так само
працюють `values-night/` (темна тема), `drawable-hdpi/` (щільність екрана),
`layout-sw600dp/` (планшети).

Тепер підстав їх у коді. У `NotesScreen.kt`:

```kotlin
import androidx.compose.ui.res.stringResource
import com.webuxmotion.androidstarter.R

// було: Text("Нотатки (${notes.size})")
Text(stringResource(R.string.notes_title, notes.size))

// було: Text("Оновити")
Text(stringResource(R.string.action_refresh))
```

`%1$d` у рядку — підстановка першого аргументу як числа (`%1$s` — як рядка).
`R` — згенерований клас із посиланнями на всі ресурси; він з'являється після
першої збірки.

Заміни решту написів так само. Якщо Android Studio не бачить `R` — `Build` →
`Clean Project`, потім `Rebuild`.

## Крок 2. Іконка застосунку

Зараз у застосунку системна іконка-заглушка. Зробимо адаптивну — таку, що
підлаштовується під форму іконок у лаунчері (коло, квадрат, «крапля»).

**2.1.** `app/src/main/res/values/ic_launcher_background.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="ic_launcher_background">#0F172A</color>
</resources>
```

**2.2.** `app/src/main/res/drawable/ic_launcher_foreground.xml`:

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">

    <!-- Аркуш нотатки; усе важливе тримаємо в центральній зоні 72dp. -->
    <path
        android:fillColor="#FFC65C"
        android:pathData="M36,30 h30 l12,12 v36 a4,4 0 0,1 -4,4 h-38 a4,4 0 0,1 -4,-4 v-44 a4,4 0 0,1 4,-4 z" />
    <path
        android:fillColor="#0F172A"
        android:pathData="M43,52 h22 v4 h-22 z M43,62 h22 v4 h-22 z" />
</vector>
```

**2.3.** `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
```

**2.4.** Той самий вміст скопіюй у `mipmap-anydpi-v26/ic_launcher_round.xml`.

**2.5.** У `AndroidManifest.xml`, у тег `<application>`:

```xml
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
```

Про адаптивні іконки варто знати два факти. Перший: система обрізає їх до
потрібної форми, тому важливе має бути в центральному колі діаметром 72dp зі
108dp — краї можуть зникнути. Другий: формат працює з Android 8.0 (API 26), а в
нас `minSdk = 26`, тому PNG-версії під старі системи не потрібні взагалі.

Найпростіший шлях зробити красиву іконку — `File` → `New` → `Image Asset` в
Android Studio: там є попередній перегляд усіх форм.

## Крок 3. Версія застосунку

`app/build.gradle.kts`, блок `defaultConfig`:

```kotlin
        versionCode = 2
        versionName = "1.0"
```

- **`versionCode`** — ціле число для системи. При оновленні мусить **зростати**,
  інакше Android і Google Play відмовляться ставити APK поверх наявного.
- **`versionName`** — рядок для людей: «1.0», «2.3.1-beta». Системі байдуже.

## Крок 4. Ключ для підпису

Android не встановить непідписаний APK. Debug-збірки підписуються автоматично
службовим ключем, релізні — треба своїм.

```sh
mkdir -p ~/keystores
keytool -genkeypair -v \
  -keystore ~/keystores/android-starter.jks \
  -alias android-starter \
  -keyalg RSA -keysize 2048 -validity 10000
```

`keytool` спитає пароль і кілька полів (ім'я, організація — можна лишити
порожніми). `-validity 10000` — приблизно 27 років.

**Цей файл і пароль не можна втратити.** Оновлення застосунку в Google Play
приймаються тільки з тим самим підписом; загубиш ключ — публікувати оновлення
доведеться як новий застосунок, з нуля.

**І цей файл ніколи не потрапляє в git.** Тому паролі кладемо в окремий файл
поза репозиторієм.

Створи `keystore.properties` у корені проєкту:

```properties
storeFile=/Users/webuxmotion/keystores/android-starter.jks
storePassword=твій_пароль
keyAlias=android-starter
keyPassword=твій_пароль
```

Одразу додай у `.gitignore`:

```
keystore.properties
*.jks
*.keystore
```

Перевір, що git його не бачить (має бути порожньо):

```sh
git status --porcelain | grep keystore
```

## Крок 5. Налаштувати релізну збірку

`app/build.gradle.kts`. **Угорі файлу**, поруч із наявним `import`:

```kotlin
import java.util.Properties
```

**Перед блоком `android { }`**:

```kotlin
val keystorePropertiesFile = rootProject.file("keystore.properties")
val keystoreProperties = Properties().apply {
    if (keystorePropertiesFile.exists()) {
        load(keystorePropertiesFile.inputStream())
    }
}
```

**Усередині `android { }`**, після `defaultConfig`:

```kotlin
    signingConfigs {
        create("release") {
            if (keystorePropertiesFile.exists()) {
                storeFile = file(keystoreProperties.getProperty("storeFile"))
                storePassword = keystoreProperties.getProperty("storePassword")
                keyAlias = keystoreProperties.getProperty("keyAlias")
                keyPassword = keystoreProperties.getProperty("keyPassword")
            }
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro",
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }
```

Перевірка `if (keystorePropertiesFile.exists())` потрібна, щоб проєкт
збирався (у debug) навіть у того, хто клонував репозиторій і не має ключа.

Створи порожній `app/proguard-rules.pro` — файл згадується в конфігурації, тому
має існувати:

```
# Правила R8 для релізної збірки.
# Тут описують класи, які не можна перейменовувати чи викидати.
```

## Крок 6. Що робить `isMinifyEnabled`

R8 (наступник ProGuard) робить три речі: викидає невикористаний код,
скорочує імена класів і методів до `a`, `b`, `c`, та вбудовує дрібні методи.
`isShrinkResources` додатково викидає невикористані ресурси. APK худне в рази.

Плата — код, який шукає класи **за іменем у рантаймі**, ламається: імена ж
змінились. Найчастіші жертви — бібліотеки серіалізації JSON. Наш Gson читає
поля `PostDto` через рефлексію, тому додай у `app/proguard-rules.pro`:

```
# Gson читає поля DTO через рефлексію — імена й типи мають лишитись.
-keepattributes Signature
-keep class com.webuxmotion.androidstarter.data.remote.** { *; }
```

Це рівно та причина, чому релізну збірку **треба перевіряти на пристрої**:
debug працює, а реліз падає — типова історія, і майже завжди через R8.

## Крок 7. Зібрати й поставити реліз

```sh
./gradlew assembleRelease
```

Результат:

```
app/build/outputs/apk/release/app-release.apk
```

Debug-версію треба спершу видалити: підписи різні, і система не дасть поставити
одну поверх іншої.

```sh
adb uninstall com.webuxmotion.androidstarter
adb install app/build/outputs/apk/release/app-release.apk
```

Порівняй розміри — різниця й буде роботою R8:

```sh
ls -lh app/build/outputs/apk/debug/app-debug.apk \
       app/build/outputs/apk/release/app-release.apk
```

Для застосунку з цього курсу виходить приблизно **10 МБ у debug проти 1.4 МБ у
релізі**. Більшу частину debug-ваги дають інструменти прев'ю та відладки, які в
реліз просто не потрапляють.

Переконатись, що APK справді підписаний твоїм ключем:

```sh
~/Library/Android/sdk/build-tools/36.1.0/apksigner verify --print-certs \
  app/build/outputs/apk/release/app-release.apk
```

Перевір на телефоні всі екрани: список вантажиться, деталі відкриваються, тема
перемикається. Якщо щось падає саме в релізі — дивись logcat і додавай правила
в `proguard-rules.pro`.

Для Google Play потрібен інший формат — `.aab`:

```sh
./gradlew bundleRelease   # app/build/outputs/bundle/release/app-release.aab
```

Play із нього сам збирає окремий APK під кожен пристрій. Для встановлення
вручну через `adb` він не годиться — тільки APK.

## Крок 8. Зафіксувати

```sh
git status          # переконайся, що keystore.properties і .jks НЕ у списку
git add -A
git commit -m "Урок 10: іконка, локалізація, релізна збірка"
```

## Що змінилось у git

```
.gitignore                                       # +3 правила
app/build.gradle.kts                             # версія, підпис, minify
app/proguard-rules.pro                           (новий)
app/src/main/AndroidManifest.xml                 # +іконка
app/src/main/res/values/strings.xml              # +тексти
app/src/main/res/values-uk/strings.xml           (новий)
app/src/main/res/values/ic_launcher_background.xml   (новий)
app/src/main/res/drawable/ic_launcher_foreground.xml (новий)
app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml       (новий)
app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml (новий)
app/src/main/java/.../ui/NotesScreen.kt          # stringResource
```

І головне — **чого в списку немає**: `keystore.properties` і `.jks`. Секрети в
історії git не зникають після видалення файлу, тому правило одне — не комітити
їх ніколи.

## Підсумок курсу

Подивись на весь шлях одним поглядом:

```sh
git log --oneline --stat
```

Що з'явилось за десять уроків:

| Шар | Файли |
|---|---|
| Точка входу | `MainActivity` |
| Навігація | `AppNavHost` |
| Екрани | `NotesScreen`, `NoteDetailScreen` |
| Стан і логіка | `NotesViewModel`, `SettingsViewModel` |
| Дані | `Note`, `SettingsRepository`, `NotesApi` |
| Оформлення | `ui/theme/Color.kt`, `ui/theme/Theme.kt` |
| Ресурси | `strings.xml` × 2 мови, іконка |
| Збірка | `libs.versions.toml`, `build.gradle.kts`, підпис, R8 |

Закономірності, які варто винести:

- **Нова бібліотека** → змінюються рівно два gradle-файли.
- **Нова можливість системи** (дозвіл, іконка) → змінюється `AndroidManifest.xml`.
- **Зміна вигляду** → один файл екрана.
- **Зміна того, де живе стан** → екрани не змінюються взагалі, якщо межі
  проведені правильно.

## Куди рухатись далі

1. **Room** — локальна база даних. Нотатки нарешті переживуть перезапуск, а
   `Flow` з бази під'єднається до `ViewModel` так само, як DataStore.
2. **Hilt** — впровадження залежностей. Прибирає ту саму рутину «прокинути
   параметр через п'ять рівнів», яку ти відчув в уроці 8.
3. **Тести** — `NotesViewModel` тестується без Android узагалі, бо він не
   залежить від UI. Це прямий дивіденд від архітектури з уроку 7.
4. **Type-safe navigation** — сучасна заміна рядковим маршрутам у Navigation 2.8+
   (`@Serializable`-об'єкти замість `"note/{noteId}"`).
5. **WorkManager**, сповіщення, віджети — якщо потрібна робота поза екраном.

Найкращий спосіб продовжити — той самий, що й у курсі: маленький крок, збірка,
перевірка на телефоні, коміт.
