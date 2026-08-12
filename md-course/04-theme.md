# Урок 4. Своя тема: кольори і темний режим

**Мета:** прибрати захардкоджений колір фону, зібрати всі кольори в одному
місці й навчити застосунок перемикатися між світлою та темною темою разом із
системою.

## Навіщо тема

Зараз у `MainActivity.kt` є рядок:

```kotlin
private val BackgroundColor = Color(0xFF0F172A)
```

Поки екран один — нормально. Але щойно екранів стане п'ять, кольори
розповзуться по файлах, і зміна фірмового кольору перетвориться на пошук з
заміною. `MaterialTheme` вирішує це: кольори, шрифти й форми лежать в одному
місці, а компоненти самі їх звідти беруть.

`Button` уже зараз малюється кольором `primary` з теми — просто тема поки
стандартна, фіолетова за замовчуванням Material 3.

## Кольорова схема Material 3

Схема — це не палітра «на око», а набір ролей. Головні:

| Роль | Де застосовується |
|---|---|
| `primary` | головна дія: заливка `Button`, активний перемикач |
| `onPrimary` | те, що лежить **на** `primary` (текст на кнопці) |
| `background` | фон екрана |
| `onBackground` | текст на фоні |
| `surface` | поверхні поверх фону: картки, панелі |
| `onSurface` | текст на поверхнях |
| `error` / `onError` | помилки |

Пара `X` / `onX` — центральна ідея: задаєш фон і колір того, що на ньому, і
компоненти самі підбирають правильний колір тексту. Саме тому в уроці про фон
працював `contentColor`.

## Крок 1. Файл із кольорами

Створи папку `ui/theme` і в ній файл
`…/androidstarter/ui/theme/Color.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui.theme

import androidx.compose.ui.graphics.Color

// Темна палітра
val Navy900 = Color(0xFF0F172A)
val Navy800 = Color(0xFF1E293B)
val Amber300 = Color(0xFFFFC65C)
val Slate100 = Color(0xFFF1F5F9)

// Світла палітра
val White = Color(0xFFFFFFFF)
val Slate50 = Color(0xFFF8FAFC)
val Amber700 = Color(0xFFB45309)
val Slate900 = Color(0xFF0F172A)
```

## Крок 2. Файл із темою

Створи `…/androidstarter/ui/theme/Theme.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui.theme

import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable

private val DarkColors = darkColorScheme(
    primary = Amber300,
    onPrimary = Navy900,
    background = Navy900,
    onBackground = Slate100,
    surface = Navy800,
    onSurface = Slate100,
)

private val LightColors = lightColorScheme(
    primary = Amber700,
    onPrimary = White,
    background = Slate50,
    onBackground = Slate900,
    surface = White,
    onSurface = Slate900,
)

@Composable
fun AndroidStarterTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    MaterialTheme(
        colorScheme = if (darkTheme) DarkColors else LightColors,
        content = content,
    )
}
```

Що тут важливо:

- `isSystemInDarkTheme()` читає системне налаштування телефону. Параметр із
  таким значенням за замовчуванням дозволяє за потреби нав'язати тему вручну —
  це знадобиться в уроці 8.
- `content: @Composable () -> Unit` — функція приймає інший UI як аргумент.
  Завдяки правилу Kotlin «остання лямбда виноситься за дужки» виклик виглядає
  як блок: `AndroidStarterTheme { … }`.
- Ролі, які не вказані явно (`secondary`, `outline`, …), беруться з
  дефолтної схеми Material 3.

## Крок 3. Підключити тему й прибрати захардкоджений колір

У `MainActivity.kt`:

**3.1.** Видали рядки з `BackgroundColor` і коментар над ним.

**3.2.** Заміни тіло `onCreate` на:

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            AndroidStarterTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background,
                ) {
                    HelloScreen()
                }
            }
        }
    }
```

**3.3.** Імпорти — додай:

```kotlin
import androidx.activity.enableEdgeToEdge
import com.webuxmotion.androidstarter.ui.theme.AndroidStarterTheme
```

і прибери той, що більше не потрібен:

```kotlin
import androidx.compose.ui.graphics.Color
```

Зверни увагу: `contentColor` у `Surface` більше не задається вручну. Він
обчислюється автоматично як `onBackground` — це і є користь від пар `X`/`onX`.

## Крок 4. Врахувати edge-to-edge

`enableEdgeToEdge()` дозволяє застосунку малювати під системними панелями —
смуга статусу стає прозорою і фон екрана йде під неї (з Android 15 це поведінка
за замовчуванням, тож краще звикати одразу).

Плата за це — вміст може заїхати під годинник угорі та під навігаційні кнопки
внизу. Виправляється одним модифікатором у `HelloScreen`:

```kotlin
    Column(
        modifier = Modifier
            .fillMaxSize()
            .safeDrawingPadding()      // відступи від системних панелей
            .padding(24.dp),
```

```kotlin
import androidx.compose.foundation.layout.safeDrawingPadding
```

У наступному уроці цю роботу візьме на себе `Scaffold`, але корисно раз
побачити механізм напряму.

## Крок 5. Перевірити обидві теми

```sh
./gradlew installDebug
```

Тепер перемкни тему в системі: шторка → «Темна тема», або
`Налаштування` → `Дисплей` → `Темна тема`. Застосунок має перемалюватись сам,
без перезапуску.

Перемкнути з терміналу, не чіпаючи телефон:

```sh
adb shell "cmd uimode night yes"   # темна
adb shell "cmd uimode night no"    # світла
```

У прев'ю (`@Preview`) друга тема дивиться так:

```kotlin
@Preview(showBackground = true)
@Preview(showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
private fun HelloScreenPreview() {
    AndroidStarterTheme {
        HelloScreen()
    }
}
```

```kotlin
import android.content.res.Configuration
```

Дві анотації над однією функцією дають дві панелі прев'ю поруч.

## Крок 6. Зафіксувати

```sh
git add -A
git commit -m "Урок 4: власна тема і темний режим"
```

## Що змінилось у git

```
app/src/main/java/.../ui/theme/Color.kt      (новий)
app/src/main/java/.../ui/theme/Theme.kt      (новий)
app/src/main/java/.../MainActivity.kt        (змінений)
```

**Перші файли, що з'явилися не в корені пакета.** Структура папок у Kotlin має
збігатися з `package`: файл у `ui/theme/` мусить починатися з
`package com.webuxmotion.androidstarter.ui.theme`. Помилка `Package directive
does not match file location` — саме про це.

Подивись на коміт як на ціле:

```sh
git show --stat HEAD
```

Два нових файли, один змінений — і жодних змін у gradle. Тема не потребує
залежностей: `darkColorScheme` і `lightColorScheme` уже є в `material3`.

## Вправа

1. Підбери свою палітру (зручно на [m3.material.io/theme-builder](https://m3.material.io/theme-builder))
   і заміни кольори в `Color.kt`. Зверни увагу, що жодного іншого файлу
   чіпати не доведеться.
2. Додай у `Theme.kt` свою типографіку: `MaterialTheme(typography = Typography(
   headlineMedium = TextStyle(...)))`.
3. Тимчасово виклич `AndroidStarterTheme(darkTheme = true)` у `MainActivity` і
   переконайся, що системне налаштування більше не впливає. Потім поверни як було.

## Далі

[Урок 5. Списки: `LazyColumn` і `data class`](05-lists.md)
