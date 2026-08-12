# Урок 2. Компонування: `Column`, `Row`, `@Preview`

**Мета:** розкласти кілька елементів на екрані й навчитися дивитись результат
без збірки на телефоні.

## Три базові контейнери

| Контейнер | Складає вміст | Головні параметри |
|---|---|---|
| `Column` | згори вниз | `verticalArrangement`, `horizontalAlignment` |
| `Row` | зліва направо | `horizontalArrangement`, `verticalAlignment` |
| `Box` | шарами, один поверх одного | `contentAlignment` |

Легко переплутати: **arrangement** — це розподіл елементів **вздовж** головної
осі контейнера (`Center`, `SpaceBetween`, `spacedBy(8.dp)`), а **alignment** —
вирівнювання **впоперек** неї.

## Крок 1. Замінити `Box` на `Column`

У `…/androidstarter/MainActivity.kt` заміни `HelloScreen` на:

```kotlin
@Composable
fun HelloScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Text(
            text = "Android Starter",
            style = MaterialTheme.typography.headlineMedium,
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = "Урок 2: компонування",
            style = MaterialTheme.typography.bodyLarge,
        )
        Spacer(modifier = Modifier.height(24.dp))
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            Text(text = "Kotlin", style = MaterialTheme.typography.labelLarge)
            Text(text = "•", style = MaterialTheme.typography.labelLarge)
            Text(text = "Compose", style = MaterialTheme.typography.labelLarge)
        }
    }
}
```

Нові імпорти:

```kotlin
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
```

`Box` більше не використовується — Android Studio підсвітить його імпорт сірим.
Можеш прибрати рядок `import androidx.compose.foundation.layout.Box`, але
компіляції він не заважає.

## Крок 2. Два способи робити проміжки

У коді вище є обидва — порівняй:

```kotlin
Spacer(modifier = Modifier.height(8.dp))          // ручний проміжок між двома елементами
Arrangement.spacedBy(12.dp)                       // однаковий проміжок між усіма дітьми
```

`spacedBy` майже завжди краще: додаси елемент — проміжок з'явиться сам, і не
буде «зайвого» відступу в кінці. `Spacer` доречний, коли проміжки різні, як тут.

## Крок 3. `weight` — поділити вільний простір

Спробуй тимчасово замінити `Row` на такий і подивись, що станеться:

```kotlin
Row(modifier = Modifier.fillMaxWidth()) {
    Text("зліва", modifier = Modifier.weight(1f))
    Text("справа")
}
```

`weight(1f)` каже: «займи весь простір, що лишився після інших елементів». Це
той самий механізм, що `flex: 1` у CSS. Потрібен імпорт
`androidx.compose.foundation.layout.fillMaxWidth`.

## Крок 4. Прев'ю прямо в IDE

Щоб не збирати APK заради кожного зсуву на 4dp, Compose вміє малювати екран у
самій Android Studio.

**4.1.** Додай два записи в `gradle/libs.versions.toml`, у секцію `[libraries]`:

```toml
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
```

Версії тут немає навмисно: ці бібліотеки входять до Compose BOM
(`platform(libs.androidx.compose.bom)` у модулі), і BOM сам добирає узгоджені
версії для всього сімейства Compose.

**4.2.** Підключи їх у `app/build.gradle.kts`, у блок `dependencies`:

```kotlin
    implementation(libs.androidx.compose.ui.tooling.preview)
    debugImplementation(libs.androidx.compose.ui.tooling)
```

`debugImplementation` означає «тільки в debug-збірці». Інструменти прев'ю важать
чимало, і в релізному APK вони не потрібні.

Зверни увагу на перетворення імені: у TOML запис зветься
`androidx-compose-ui-tooling`, а в Kotlin до нього звертаються як
`libs.androidx.compose.ui.tooling` — дефіси стають крапками.

**4.3.** Додай у кінець `MainActivity.kt`:

```kotlin
@Preview(showBackground = true)
@Composable
private fun HelloScreenPreview() {
    MaterialTheme {
        HelloScreen()
    }
}
```

Імпорт: `import androidx.compose.ui.tooling.preview.Preview`

**4.4.** Синхронізуй проєкт (Android Studio сама запропонує «Sync Now» після
зміни gradle-файлів) і натисни `Split` у правому верхньому куті редактора —
з'явиться панель із намальованим екраном.

Прев'ю-функцію роблять `private` і не викликають з застосунку: вона існує тільки
для IDE.

## Крок 5. Перевірити й зафіксувати

```sh
./gradlew installDebug
git add -A
git commit -m "Урок 2: Column/Row і прев'ю"
```

## Що змінилось у git

```
app/build.gradle.kts                        # +2 залежності
gradle/libs.versions.toml                   # +2 записи в каталог
app/src/main/java/.../MainActivity.kt       # верстка + @Preview
```

**Три файли — і це шаблон, який повторюватиметься весь курс.** Щоразу, коли
з'являється нова бібліотека, змінюються рівно два gradle-файли: каталог версій
(що існує) і модуль (що підключено). Тримати версії в одному місці — саме те,
заради чого потрібен `libs.versions.toml`.

## Вправа

1. Заміни `Arrangement.Center` на `Arrangement.SpaceBetween` і подивись, як
   елементи розповзуться по краях екрана.
2. Зроби `Row`, у якому один елемент притиснутий ліворуч, а другий — праворуч,
   без жодного `Spacer` фіксованої ширини (підказка: `Spacer(Modifier.weight(1f))`).
3. Додай другий `@Preview` з іншим розміром екрана:
   `@Preview(widthDp = 320, heightDp = 480)`.

## Далі

[Урок 3. Стан і взаємодія: кнопка-лічильник](03-state.md)
