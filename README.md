# Android Starter

Мінімальний Android-застосунок: один екран із написом `Hello World`.
Kotlin + Jetpack Compose, `minSdk 26`, `compileSdk 36`.

Основа для поступового додавання функціоналу — кожну фічу зручно комітити окремо
й дивитися `git diff`, щоб бачити, які саме файли вона вимагає.

## Підготовка терміналу

**Java.** `java` у `PATH` на цій машині — це JDK 25, якого Gradle 8.11 не
підтримує: збірка падає з невиразним `What went wrong: 25.0.1`. Тому в
`gradle.properties` прописано шлях до JDK 17, що йде в комплекті з Android
Studio:

```properties
org.gradle.java.home=/Applications/Android Studio.app/Contents/jbr/Contents/Home
```

Завдяки цьому `./gradlew` працює як є — нічого експортувати не треба. Якщо
клонуєш репозиторій на іншу машину, цей рядок доведеться поправити або видалити.

**adb.** Лежить у `~/Library/Android/sdk/platform-tools/` і в `PATH` не входить.
Щоб не писати повний шлях щоразу, додай у `~/.zshrc`:

```sh
export PATH="$HOME/Library/Android/sdk/platform-tools:$PATH"
```

Далі в README вважається, що це зроблено. Якщо ні — виклик виглядатиме так:
`~/Library/Android/sdk/platform-tools/adb devices`

## Зібрати APK

```sh
cd ~/nrf-projects/android-starter
./gradlew assembleDebug
```

Готовий файл:

```
app/build/outputs/apk/debug/app-debug.apk
```

Debug-збірка вже підписана тестовим ключем, тому її можна одразу ставити на
телефон. Release-збірка (`./gradlew assembleRelease`) без налаштованого підпису
не встановиться — вона потрібна тільки для публікації.

Корисне:

```sh
./gradlew clean          # видалити результати попередніх збірок
./gradlew tasks          # список усіх доступних завдань
```

Якщо збірка падає з `What went wrong: 25.0.1` (або іншим номером версії) —
Gradle узяв Java не звідти; перевір рядок `org.gradle.java.home` у
`gradle.properties`.

## Підключити телефон

1. На телефоні: `Налаштування` → `Для розробників` → увімкнути **Налагодження USB**
2. На Xiaomi/Redmi (MIUI) додатково увімкнути **Встановлення через USB**
   — без нього установка падає з `INSTALL_FAILED_USER_RESTRICTED`
3. Підключити кабелем і підтвердити на екрані телефону запит «Дозволити
   налагодження USB з цього комп'ютера?»

Перевірка, що телефон видно:

```sh
adb devices -l
```

Має з'явитися рядок зі станом `device`:

```
97373e340806    device usb:0-1.4 product:olive_eea model:Redmi_8 device:olive
```

Якщо стан `unauthorized` — не підтверджено запит на телефоні.
Якщо список порожній — інший кабель/порт або `adb kill-server && adb start-server`.

## Закинути на телефон

Найкоротший шлях — Gradle сам збере і встановить:

```sh
./gradlew installDebug
```

Те саме вручну, якщо APK уже зібраний (`-r` = перевстановити поверх наявного,
зберігши дані застосунку):

```sh
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Запустити застосунок на телефоні з терміналу:

```sh
adb shell am start -n com.webuxmotion.androidstarter/.MainActivity
```

Видалити:

```sh
adb uninstall com.webuxmotion.androidstarter
```

## Установка без `adb install` (обхід обмежень MIUI)

Якщо MIUI не дає ставити по USB, можна просто скопіювати APK у пам'ять телефону
і встановити вручну через файловий менеджер:

```sh
adb push app/build/outputs/apk/debug/app-debug.apk /sdcard/Download/
```

На телефоні: `Провідник` → `Download` → `app-debug.apk` → дозволити встановлення
з цього джерела.

## Діагностика

Логи застосунку (`Ctrl+C` — вихід):

```sh
adb logcat --pid=$(adb shell pidof com.webuxmotion.androidstarter)
```

Знімок екрана телефону у файл:

```sh
adb exec-out screencap -p > screen.png
```

## Структура проєкту

| Файл | Навіщо |
|---|---|
| `settings.gradle.kts` | назва проєкту, список модулів, репозиторії залежностей |
| `build.gradle.kts` | плагіни, спільні для всіх модулів |
| `gradle/libs.versions.toml` | версії всіх бібліотек в одному місці (version catalog) |
| `gradle.properties` | налаштування самого Gradle |
| `gradlew`, `gradle/wrapper/` | Gradle-обгортка: збірка працює без встановленого Gradle |
| `app/build.gradle.kts` | конфіг модуля: SDK-версії, `applicationId`, залежності |
| `app/src/main/AndroidManifest.xml` | опис застосунку для системи; тут `MainActivity` позначена стартовою |
| `app/src/main/java/.../MainActivity.kt` | єдиний екран застосунку |
| `app/src/main/res/values/strings.xml` | назва застосунку |
| `app/src/main/res/values/themes.xml` | тема без ActionBar |

`local.properties` (шлях до Android SDK) навмисно в `.gitignore` — він
машинозалежний. Android Studio створює його автоматично при відкритті проєкту.
