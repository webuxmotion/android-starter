# Урок 7. `ViewModel` і `StateFlow`

**Мета:** винести стан і логіку з UI туди, де вони переживають поворот екрана, —
і розділити застосунок на «що показувати» та «як показувати».

## Проблема, яку розв'язуємо

Поверни телефон на екрані списку. **Додані нотатки зникли** — лишились три
початкові. Причина та сама, що в уроці 3: `remember` живе в `Activity`, а
`Activity` при повороті знищується.

`rememberSaveable` тут не врятує: список об'єктів `Note` у `Bundle` просто так
не запхнеш, та й `Bundle` — сховище для дрібниць, а не для даних застосунку.

`ViewModel` — саме той об'єкт, який Android **не знищує** при повороті. Він
переживає перестворення `Activity` і вмирає лише тоді, коли екран закривають
по-справжньому.

## Крок 1. Підключити залежності

**1.1.** `gradle/libs.versions.toml`, секція `[versions]`:

```toml
lifecycle = "2.8.7"
```

**1.2.** Там же, `[libraries]`:

```toml
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
```

**1.3.** `app/build.gradle.kts`, блок `dependencies`:

```kotlin
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.lifecycle.runtime.compose)
```

Перша дає функцію `viewModel()`, друга — `collectAsStateWithLifecycle()`.

## Крок 2. Описати стан екрана одним об'єктом

Створи `…/androidstarter/ui/NotesViewModel.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.lifecycle.ViewModel
import com.webuxmotion.androidstarter.data.Note
import com.webuxmotion.androidstarter.data.sampleNotes
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

data class NotesUiState(
    val notes: List<Note> = sampleNotes,
)

class NotesViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(NotesUiState())
    val uiState: StateFlow<NotesUiState> = _uiState.asStateFlow()

    fun addNote(title: String) {
        if (title.isBlank()) return
        _uiState.update { state ->
            val nextId = (state.notes.maxOfOrNull { it.id } ?: 0) + 1
            val newNote = Note(nextId, title, "Створено вручну")
            state.copy(notes = listOf(newNote) + state.notes)
        }
    }

    fun deleteNote(note: Note) {
        _uiState.update { state ->
            state.copy(notes = state.notes.filterNot { it.id == note.id })
        }
    }
}
```

Ключові моменти:

- **`NotesUiState`** — увесь стан екрана одним незмінним об'єктом. Коли додасться
  «завантаження» й «помилка» (урок 9), вони стануть просто новими полями, а не
  ще трьома розсипаними змінними.
- **`MutableStateFlow` приватний, `StateFlow` публічний.** Ззовні стан можна
  тільки читати; змінити його можна лише через методи `ViewModel`. Це не
  формальність — саме так логіка не розтікається по екранах.
- **`update { }`** атомарно замінює стан новим: береш поточний, повертаєш новий
  через `copy`. Мутувати старий не можна — див. урок 5 про незмінність.
- `listOf(newNote) + state.notes` створює **новий** список. Compose побачить,
  що об'єкт інший, і перемалює список.

`StateFlow` — це «потік значень, у якого завжди є поточне значення» з
бібліотеки корутин Kotlin. Для UI-стану це стандарт; `mutableStateOf` теж
працює, але `Flow` легко з'єднується з базою даних і мережею, що знадобиться
далі.

## Крок 3. Під'єднати `ViewModel` до навігації

Заміни вміст `…/androidstarter/ui/AppNavHost.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.NavType
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.navArgument

@Composable
fun AppNavHost(viewModel: NotesViewModel = viewModel()) {
    val navController = rememberNavController()
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    NavHost(
        navController = navController,
        startDestination = "notes",
    ) {
        composable(route = "notes") {
            NotesScreen(
                notes = uiState.notes,
                onAdd = viewModel::addNote,
                onDelete = viewModel::deleteNote,
                onOpen = { noteId -> navController.navigate("note/$noteId") },
            )
        }

        composable(
            route = "note/{noteId}",
            arguments = listOf(navArgument("noteId") { type = NavType.IntType }),
        ) { backStackEntry ->
            val noteId = backStackEntry.arguments?.getInt("noteId") ?: 0
            NoteDetailScreen(
                note = uiState.notes.firstOrNull { it.id == noteId },
                onBack = { navController.popBackStack() },
            )
        }
    }
}
```

Що сталося з файлом: зникли `remember`, `toMutableStateList`, створення `Note` і
вся логіка додавання. Лишилась чиста навігація. **Екрани не змінились узагалі** —
вони й раніше отримували дані та лямбди ззовні. Це дивіденд від state hoisting
з уроку 6.

Дрібниці синтаксису:

- `viewModel()` знаходить або створює `NotesViewModel`, прив'язаний до
  `Activity`. Виклик у параметрі за замовчуванням — поширений прийом: у
  `@Preview` чи тесті можна передати свій екземпляр.
- `viewModel::addNote` — посилання на метод, коротший запис
  `{ title -> viewModel.addNote(title) }`.
- `collectAsStateWithLifecycle()` підписується на `StateFlow` і перетворює його
  на стан Compose. «WithLifecycle» означає, що підписка **зупиняється**, коли
  застосунок згорнутий, і поновлюється при поверненні — інакше згорнутий
  застосунок марно молотив би оновлення.

## Крок 4. Перевірити те, заради чого все робилось

```sh
./gradlew installDebug
```

Додай дві-три нотатки. **Поверни телефон.** Список на місці.

Ще один спосіб побачити ефект — примусово «вбити» процес і подивитись, що це
вже не рятує:

```sh
adb shell am force-stop com.webuxmotion.androidstarter
```

Запусти знову — нотатки зникли. `ViewModel` живе в пам'яті процесу; щоб дані
пережили перезапуск, потрібне сховище на диску. Це наступний урок.

## Крок 5. Зафіксувати

```sh
git add -A
git commit -m "Урок 7: ViewModel і StateFlow"
```

## Що змінилось у git

```
gradle/libs.versions.toml                  # +1 версія, +2 бібліотеки
app/build.gradle.kts                       # +2 залежності
app/src/main/java/.../ui/NotesViewModel.kt (новий)
app/src/main/java/.../ui/AppNavHost.kt     (змінений)
```

`NotesScreen.kt` і `NoteDetailScreen.kt` **не в списку** — і це головний
результат уроку. Ми замінили сховище стану, не торкнувшись жодного екрана.
Так виглядає вдало проведена межа між шарами.

## Як тепер думати про архітектуру

```
MainActivity        — точка входу, майже порожня
AppNavHost          — які екрани існують і як між ними ходити
NotesViewModel      — стан і логіка (переживає поворот)
NotesScreen         — як це виглядає (без логіки, без стану)
data/Note           — модель даних
```

Це спрощений варіант того, що Google називає рекомендованою архітектурою:
**UI layer** (екрани + ViewModel) і **data layer** (моделі + репозиторії).
Репозиторій з'явиться в наступному уроці.

## Вправа

1. Додай у `NotesViewModel` метод `clearAll()` і кнопку в `TopAppBar`, що його
   викликає. Зверни увагу: змінюються рівно два файли.
2. Додай у `NotesUiState` поле `val filter: String = ""` і зроби пошук по
   заголовку. Фільтрувати треба у `ViewModel`, а не в екрані — чому?
3. Спробуй передати `viewModel()` **всередині** `composable("notes") { … }`
   замість параметра `AppNavHost`. Знайди різницю в тому, з чим саме буде
   пов'язаний час життя об'єкта (підказка: `viewModel()` бере найближчого
   власника — `NavBackStackEntry`, а не `Activity`).

## Далі

[Урок 8. Збереження налаштувань (DataStore)](08-persistence.md)
