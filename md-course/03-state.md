# Урок 3. Стан і взаємодія: кнопка-лічильник

**Мета:** зробити екран, який реагує на дотик, і зрозуміти головну ідею
Compose — **UI є функцією від стану**.

## Ключова ідея

У старому Android коді ти б знайшов кнопку за `id`, повісив слухач і вручну
змінив текст іншого елемента. У Compose так не роблять. Замість цього:

1. Є **стан** — звичайна змінна (наприклад, `count = 0`).
2. Є `@Composable`-функція, яка **описує** екран для поточного значення стану.
3. Коли стан змінюється, Compose **сам** викликає функцію повторно й оновлює
   те, що змінилось. Це і є **рекомпозиція**.

Ти ніколи не пишеш «зміни текст». Ти пишеш «текст дорівнює `count`» і окремо
«при натисканні `count` збільшується на 1».

## Крок 1. Лічильник

Заміни `HelloScreen` у `…/androidstarter/MainActivity.kt` на:

```kotlin
@Composable
fun HelloScreen() {
    var count by remember { mutableStateOf(0) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Text(
            text = "Натиснуто разів: $count",
            style = MaterialTheme.typography.headlineMedium,
        )
        Spacer(modifier = Modifier.height(24.dp))
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            Button(onClick = { count++ }) {
                Text("Додати")
            }
            OutlinedButton(
                onClick = { count = 0 },
                enabled = count > 0,
            ) {
                Text("Скинути")
            }
        }
    }
}
```

Нові імпорти:

```kotlin
import androidx.compose.material3.Button
import androidx.compose.material3.OutlinedButton
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
```

Постав і перевір:

```sh
./gradlew installDebug
```

## Крок 2. Розібрати рядок зі станом

```kotlin
var count by remember { mutableStateOf(0) }
```

Три різні речі, які легко злипаються в одну:

- **`mutableStateOf(0)`** — контейнер зі значенням, за яким Compose **стежить**.
  Читання цього значення всередині `@Composable` підписує функцію на зміни:
  зміниться значення — Compose перемалює саме цю ділянку.
- **`remember { … }`** — «не створюй заново при кожній рекомпозиції». Без нього
  лічильник створювався б із нуля щоразу, коли екран перемальовується, і
  ніколи б не зростав. Спробуй прибрати `remember` — кнопка перестане працювати.
- **`by`** — делегат Kotlin, синтаксичний цукор. З ним пишемо `count` і
  `count++`; без нього довелося б писати `count.value` і `count.value++`.
  Саме заради `by` потрібні імпорти `getValue`/`setValue`.

Порівняй два варіанти — вони роблять одне й те саме:

```kotlin
var count by remember { mutableStateOf(0) }   // count, count++
val count = remember { mutableStateOf(0) }    // count.value, count.value++
```

## Крок 3. Знайти справжню ваду

Поверни телефон боком (якщо автоповорот вимкнений — увімкни його в шторці).

**Лічильник скинувся на нуль.**

Це не баг Compose, а поведінка Android: при повороті екрана система знищує
`Activity` і створює заново, щоб перебудувати UI під нові розміри. Усе, що жило
в пам'яті `Activity`, зникає. `remember` переживає рекомпозицію, але не
переживає перестворення `Activity`.

Швидке лікування — `rememberSaveable`:

```kotlin
var count by rememberSaveable { mutableStateOf(0) }
```

```kotlin
import androidx.compose.runtime.saveable.rememberSaveable
```

Він зберігає значення в `Bundle` — тому саме сховище, куди система складає стан
перед знищенням `Activity`. Працює «з коробки» для простих типів: `Int`,
`String`, `Boolean`, `List` із них тощо. Для власних класів довелося б писати
правила збереження.

Перевір ще раз: постав, натисни кілька разів, поверни телефон — число на місці.

Це напівміра: `rememberSaveable` не переживе того, що система вивантажить
застосунок із пам'яті надовго. Правильне місце для стану — `ViewModel`
(урок 7) і сховище на диску (урок 8). Але для маленького UI-стану типу
«чи розгорнутий блок» `rememberSaveable` — саме те.

## Крок 4. Підняти стан (state hoisting)

Зараз `HelloScreen` і зберігає стан, і малює. Це зручно, поки екран один. Коли
з'явиться другий екран чи тести, стан треба буде винести «нагору», а компонент
зробити безстанним:

```kotlin
@Composable
fun CounterScreen(
    count: Int,
    onIncrement: () -> Unit,
    onReset: () -> Unit,
) {
    // тільки малює, стану не має
}
```

Хто викликає — той і володіє станом:

```kotlin
var count by rememberSaveable { mutableStateOf(0) }
CounterScreen(
    count = count,
    onIncrement = { count++ },
    onReset = { count = 0 },
)
```

Правило: **дані течуть вниз, події — вгору**. Безстанний компонент легко
показати в `@Preview` з будь-якими даними і легко перевикористати. Це той самий
принцип, який в уроці 7 доведемо до `ViewModel`.

Зробити це перетворення — вправа наприкінці уроку.

## Крок 5. Зафіксувати

```sh
git add -A
git commit -m "Урок 3: лічильник зі станом"
```

## Що змінилось у git

```
app/src/main/java/.../MainActivity.kt
```

Знову **один файл**. Інтерактивність не вимагає ні нових залежностей, ні змін у
маніфесті — все це вже є в Compose.

## Вправа

1. Виконай крок 4 по-справжньому: винеси `CounterScreen(count, onIncrement,
   onReset)` в окрему безстанну функцію, а стан лиши в `HelloScreen`.
2. Додай `@Preview` для `CounterScreen` зі значенням `count = 42` — тепер це
   можливо саме тому, що компонент безстанний.
3. Додай кнопку «Відняти», неактивну при `count == 0`.
4. Виведи під числом текст, що змінюється: «багато!», якщо `count > 10`,
   інакше — порожній рядок. Зверни увагу, що ніякого «оновити текст» писати не
   треба: достатньо `if` усередині `@Composable`.

## Далі

[Урок 4. Своя тема: кольори і темний режим](04-theme.md)
