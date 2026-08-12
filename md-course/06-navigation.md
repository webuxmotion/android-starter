# Урок 6. Навігація між екранами

**Мета:** додати другий екран — деталі нотатки — і навчитися ходити між
екранами з робочою системною кнопкою «Назад».

## Крок 1. Підключити Navigation Compose

**1.1.** У `gradle/libs.versions.toml`, секція `[versions]`:

```toml
navigationCompose = "2.8.5"
```

**1.2.** Там же, секція `[libraries]`:

```toml
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
```

**1.3.** У `app/build.gradle.kts`, блок `dependencies`:

```kotlin
    implementation(libs.androidx.navigation.compose)
```

**1.4.** Синхронізуй проєкт (`Sync Now` в Android Studio) або просто збери з
терміналу — Gradle сам завантажить бібліотеку:

```sh
./gradlew assembleDebug
```

Тут, на відміну від Compose, версія вказана явно: Navigation не входить до
Compose BOM.

## Крок 2. Підняти стан над екранами

Зараз список нотаток живе всередині `NotesScreen`. Другий екран до нього не
дістанеться. Тому виносимо стан вище — у спільного власника обох екранів. Це та
сама техніка, що в уроці 3 (**дані течуть вниз, події — вгору**), тільки на
рівні застосунку.

Створи `…/androidstarter/ui/AppNavHost.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.runtime.toMutableStateList
import androidx.navigation.NavType
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.navArgument
import com.webuxmotion.androidstarter.data.Note
import com.webuxmotion.androidstarter.data.sampleNotes

@Composable
fun AppNavHost() {
    val navController = rememberNavController()
    val notes = remember { sampleNotes.toMutableStateList() }

    NavHost(
        navController = navController,
        startDestination = "notes",
    ) {
        composable(route = "notes") {
            NotesScreen(
                notes = notes,
                onAdd = { title ->
                    val nextId = (notes.maxOfOrNull { it.id } ?: 0) + 1
                    notes.add(0, Note(nextId, title, "Створено вручну"))
                },
                onDelete = { note -> notes.remove(note) },
                onOpen = { noteId -> navController.navigate("note/$noteId") },
            )
        }

        composable(
            route = "note/{noteId}",
            arguments = listOf(navArgument("noteId") { type = NavType.IntType }),
        ) { backStackEntry ->
            val noteId = backStackEntry.arguments?.getInt("noteId") ?: 0
            NoteDetailScreen(
                note = notes.firstOrNull { it.id == noteId },
                onBack = { navController.popBackStack() },
            )
        }
    }
}
```

Розбір:

- **`NavHost`** — контейнер, у якому показується поточний екран. `NavController`
  вирішує, який саме.
- **Маршрут (route)** — рядок-адреса екрана, як URL: `"notes"`,
  `"note/{noteId}"`. Фігурні дужки — це параметр.
- **`navigate("note/5")`** кладе екран **на стек** зверху. `popBackStack()`
  знімає його. Системна кнопка «Назад» працює з цим стеком автоматично — окремо
  нічого писати не треба.
- Стан `notes` створений **над** `NavHost`, тому переживає переходи між
  екранами: підеш у деталі й повернешся — список на місці.

## Крок 3. Переробити `NotesScreen` на безстанний

Заміни вміст `…/androidstarter/ui/NotesScreen.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.webuxmotion.androidstarter.data.Note

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun NotesScreen(
    notes: List<Note>,
    onAdd: (String) -> Unit,
    onDelete: (Note) -> Unit,
    onOpen: (Int) -> Unit,
    modifier: Modifier = Modifier,
) {
    var draft by rememberSaveable { mutableStateOf("") }

    Scaffold(
        modifier = modifier.fillMaxSize(),
        topBar = { TopAppBar(title = { Text("Нотатки (${notes.size})") }) },
    ) { innerPadding ->
        Column(modifier = Modifier.padding(innerPadding)) {
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                OutlinedTextField(
                    value = draft,
                    onValueChange = { draft = it },
                    label = { Text("Нова нотатка") },
                    singleLine = true,
                    modifier = Modifier.weight(1f),
                )
                Button(
                    onClick = {
                        onAdd(draft.trim())
                        draft = ""
                    },
                    enabled = draft.isNotBlank(),
                ) {
                    Text("Додати")
                }
            }

            LazyColumn(
                contentPadding = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(12.dp),
            ) {
                items(notes, key = { note -> note.id }) { note ->
                    NoteCard(
                        note = note,
                        onClick = { onOpen(note.id) },
                        onDelete = { onDelete(note) },
                    )
                }
            }
        }
    }
}

@Composable
private fun NoteCard(
    note: Note,
    onClick: () -> Unit,
    onDelete: () -> Unit,
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(onClick = onClick),
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = note.title, style = MaterialTheme.typography.titleMedium)
            Text(text = note.body, style = MaterialTheme.typography.bodyMedium)
            TextButton(onClick = onDelete) {
                Text("Видалити")
            }
        }
    }
}
```

Що змінилось по суті: екран більше **не володіє** списком. Він отримує готові
дані й чотири лямбди-події. Такий компонент легко показати в `@Preview`, легко
протестувати й неможливо випадково зламати логікою.

`draft` лишився всередині — це чисто UI-стан поля вводу, наверх його тягнути
немає сенсу.

## Крок 4. Екран деталей

Створи `…/androidstarter/ui/NoteDetailScreen.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.webuxmotion.androidstarter.data.Note

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun NoteDetailScreen(
    note: Note?,
    onBack: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Scaffold(
        modifier = modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                title = { Text(note?.title ?: "Нотатку не знайдено") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(
                            imageVector = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Назад",
                        )
                    }
                },
            )
        },
    ) { innerPadding ->
        Column(
            modifier = Modifier
                .padding(innerPadding)
                .padding(16.dp),
        ) {
            if (note == null) {
                Text(
                    text = "Схоже, нотатку видалили.",
                    style = MaterialTheme.typography.bodyLarge,
                )
            } else {
                Text(text = note.body, style = MaterialTheme.typography.bodyLarge)
                Text(
                    text = "ID: ${note.id}",
                    style = MaterialTheme.typography.labelSmall,
                )
            }
        }
    }
}
```

`contentDescription` — текст для незрячих користувачів, який зачитає
TalkBack. Для декоративних картинок туди передають `null`, для кнопок —
завжди осмислений опис.

## Крок 5. Запустити навігацію з `MainActivity`

У `MainActivity.kt` заміни виклик `NotesScreen()` на `AppNavHost()`, а імпорт —
на:

```kotlin
import com.webuxmotion.androidstarter.ui.AppNavHost
```

```sh
./gradlew installDebug
```

Натисни на картку → відкриється екран деталей. Кнопка «Назад» у шапці й системна
кнопка/жест «Назад» повертають до списку. Додай нотатку, зайди в неї,
повернись — список цілий.

## Крок 6. Зафіксувати

```sh
git add -A
git commit -m "Урок 6: навігація між екранами"
```

## Що змінилось у git

```
gradle/libs.versions.toml                     # версія + бібліотека
app/build.gradle.kts                          # +1 залежність
app/src/main/java/.../ui/AppNavHost.kt        (новий)
app/src/main/java/.../ui/NoteDetailScreen.kt  (новий)
app/src/main/java/.../ui/NotesScreen.kt       (сильно змінений)
app/src/main/java/.../MainActivity.kt         (один рядок)
```

Порівняй із уроком 5, де було два нових файли й жодної зміни в gradle. Тут
з'явилася **зовнішня бібліотека** — і одразу видно її «слід» у двох gradle-файлах.
Саме такий слід і корисно вміти читати в чужих комітах.

## Вправа

1. Додай третій екран «Про застосунок» із маршрутом `"about"` і кнопкою в
   `TopAppBar` списку (`actions = { IconButton(...) }`).
2. Спробуй `navController.navigate("notes")` з екрана деталей і подивись у
   стек: натискай «Назад» кілька разів. Потім додай
   `navigate("notes") { popUpTo("notes") { inclusive = true } }` і порівняй.
3. Поверни телефон на екрані деталей — чи лишився ти на ньому? (Так: маршрут
   зберігається. А от `draft` у полі вводу — завдяки `rememberSaveable`.)

## Далі

[Урок 7. `ViewModel` і `StateFlow`](07-viewmodel.md)
