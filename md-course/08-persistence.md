# Урок 8. Збереження налаштувань (DataStore)

**Мета:** зробити так, щоб вибір користувача пережив перезапуск застосунку. За
приклад візьмемо перемикач теми — і заразом познайомимось із корутинами.

## Що таке DataStore

Для дрібних налаштувань в Android є два підходи: старий `SharedPreferences` і
сучасний **DataStore**. Другий кращий тим, що не блокує потік UI (усі операції
асинхронні) і віддає дані як `Flow` — той самий тип, з яким ми вже працювали в
уроці 7.

Для списку нотаток DataStore не підійде — там потрібна база даних (Room). Але
логіка «читаємо `Flow`, пишемо через `suspend`-функцію» в обох випадках однакова.

## Крок 1. Підключити залежність

`gradle/libs.versions.toml`:

```toml
[versions]
datastore = "1.1.1"

[libraries]
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
```

`app/build.gradle.kts`:

```kotlin
    implementation(libs.androidx.datastore.preferences)
```

## Крок 2. Репозиторій

Створи `…/androidstarter/data/SettingsRepository.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.data

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

class SettingsRepository(private val context: Context) {

    private object Keys {
        val DARK_THEME = booleanPreferencesKey("dark_theme")
    }

    val darkTheme: Flow<Boolean> = context.dataStore.data
        .map { preferences -> preferences[Keys.DARK_THEME] ?: false }

    suspend fun setDarkTheme(enabled: Boolean) {
        context.dataStore.edit { preferences ->
            preferences[Keys.DARK_THEME] = enabled
        }
    }
}
```

Розбір:

- **`by preferencesDataStore(name = "settings")`** оголошується **один раз на
  весь застосунок**, поза класом. Створиш другий DataStore з тим самим іменем —
  отримаєш падіння в рантаймі. Файл фізично лежить у
  `/data/data/com.webuxmotion.androidstarter/files/datastore/settings.preferences_pb`.
- **`booleanPreferencesKey("dark_theme")`** — типізований ключ. Прочитати за ним
  рядок не вийде: компілятор не дасть.
- **`darkTheme: Flow<Boolean>`** — не одне значення, а **потік**: при кожному
  записі всі підписники отримають нове значення. Ніякого «перечитати
  налаштування» писати не треба.
- **`suspend fun`** — функція, яку можна викликати тільки з корутини. Це позначка
  «я можу почекати»: запис на диск не блокує UI, а призупиняє корутину, поки
  триває.
- **Репозиторій** — шар, що ховає джерело даних. Завтра заміниш DataStore на
  базу чи сервер — `ViewModel` не помітить різниці.

## Крок 3. ViewModel для налаштувань

Створи `…/androidstarter/ui/SettingsViewModel.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import com.webuxmotion.androidstarter.data.SettingsRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class SettingsViewModel(application: Application) : AndroidViewModel(application) {

    private val repository = SettingsRepository(application)

    val darkTheme: StateFlow<Boolean> = repository.darkTheme.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = false,
    )

    fun toggleTheme() {
        viewModelScope.launch {
            repository.setDarkTheme(!darkTheme.value)
        }
    }
}
```

Три нові речі:

- **`AndroidViewModel`** — це `ViewModel`, якому потрібен `Application`
  (DataStore вимагає `Context`). Ніякої фабрики писати не треба: `viewModel()`
  вміє створювати такі класи сам. Важливо: тримати в `ViewModel` `Context`
  **екрана** не можна — це витік пам'яті. `Application` жити стільки ж, скільки
  процес, тож він безпечний.
- **`viewModelScope.launch { }`** — запуск корутини, прив'язаної до життя
  `ViewModel`. Закриється екран — незавершені операції скасуються самі. Саме
  тут можна викликати `suspend`-функції.
- **`stateIn(...)`** перетворює «холодний» `Flow` (який починає працювати лише
  коли його читають) на `StateFlow` із завжди наявним поточним значенням.
  `WhileSubscribed(5_000)` означає: відписатись від диска через 5 секунд після
  того, як екран зник, — але не одразу, щоб поворот телефона не спричиняв
  зайвого перечитування.

## Крок 4. Застосувати тему і додати перемикач

**4.1.** `MainActivity.kt`, тіло `onCreate`:

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            val settingsViewModel: SettingsViewModel = viewModel()
            val darkTheme by settingsViewModel.darkTheme.collectAsStateWithLifecycle()

            AndroidStarterTheme(darkTheme = darkTheme) {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background,
                ) {
                    AppNavHost(onToggleTheme = settingsViewModel::toggleTheme)
                }
            }
        }
    }
```

Нові імпорти в `MainActivity.kt`:

```kotlin
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.webuxmotion.androidstarter.ui.SettingsViewModel
```

Ось де окупився параметр `darkTheme` в темі з уроку 4: тепер тему визначає не
система, а збережений вибір користувача.

**4.2.** `AppNavHost.kt` — прийняти й прокинути подію далі. У сигнатурі:

```kotlin
@Composable
fun AppNavHost(
    onToggleTheme: () -> Unit,
    viewModel: NotesViewModel = viewModel(),
) {
```

і у виклику `NotesScreen` додай останнім параметром:

```kotlin
                onToggleTheme = onToggleTheme,
```

**4.3.** `NotesScreen.kt` — новий параметр і кнопка в шапці. У сигнатурі:

```kotlin
fun NotesScreen(
    notes: List<Note>,
    onAdd: (String) -> Unit,
    onDelete: (Note) -> Unit,
    onOpen: (Int) -> Unit,
    onToggleTheme: () -> Unit,
    modifier: Modifier = Modifier,
) {
```

і `topBar` перетворюється на:

```kotlin
        topBar = {
            TopAppBar(
                title = { Text("Нотатки (${notes.size})") },
                actions = {
                    TextButton(onClick = onToggleTheme) {
                        Text("Тема")
                    }
                },
            )
        },
```

## Крок 5. Перевірити головне

```sh
./gradlew installDebug
```

1. Натисни «Тема» — застосунок перемкнеться.
2. **Закрий його повністю** і запусти знову:

```sh
adb shell am force-stop com.webuxmotion.androidstarter
adb shell am start -n com.webuxmotion.androidstarter/.MainActivity
```

Тема така, як обрав. А нотатки — знову початкові три: вони й далі живуть лише в
пам'яті `ViewModel`. Тепер різниця між «пережити поворот» і «пережити
перезапуск» видно наочно.

Подивитись, що реально лежить на диску:

```sh
adb shell run-as com.webuxmotion.androidstarter ls -l files/datastore/
```

## Крок 6. Зафіксувати

```sh
git add -A
git commit -m "Урок 8: збереження вибору теми"
```

## Що змінилось у git

```
gradle/libs.versions.toml                       # +1 версія, +1 бібліотека
app/build.gradle.kts                            # +1 залежність
app/src/main/java/.../data/SettingsRepository.kt (новий)
app/src/main/java/.../ui/SettingsViewModel.kt    (новий)
app/src/main/java/.../MainActivity.kt            (змінений)
app/src/main/java/.../ui/AppNavHost.kt           (+1 параметр)
app/src/main/java/.../ui/NotesScreen.kt          (+1 параметр, кнопка)
```

Сім файлів заради одного перемикача — і це чесна ціна. Три з них (`AppNavHost`,
`NotesScreen`, `MainActivity`) змінились лише тому, що подію довелося
**прокинути через усі рівні**. У великих застосунках цю рутину прибирають
впровадженням залежностей (Hilt) — але спершу корисно раз відчути, звідки
береться проблема.

## Вправа

1. Додай друге налаштування — `val compactList` — і застосуй його до відступів
   у `LazyColumn`. Другий ключ у тому самому DataStore, нового файлу не треба.
2. Зроби третій стан теми замість двох: «системна / світла / темна». Підказка:
   зберігай `String` через `stringPreferencesKey`, а в `AndroidStarterTheme`
   передавай `isSystemInDarkTheme()`, коли обрано «системна».
3. Помітив коротке блимання світлої теми при старті? Це `initialValue = false`
   у `stateIn`. Подумай, як його прибрати (підказка: `Boolean?` замість
   `Boolean` і не малювати UI, поки значення `null`).

## Далі

[Урок 9. Мережа: завантаження даних з API](09-network.md)
