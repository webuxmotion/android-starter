# Урок 5. Списки: `LazyColumn` і `data class`

**Мета:** показати список даних, що змінюється, і вперше відділити **модель
даних** від коду екрана.

Застосунок перетворюється на список нотаток — далі курс розвиватиме саме його.

## Крок 1. Модель даних

Створи `…/androidstarter/data/Note.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.data

data class Note(
    val id: Int,
    val title: String,
    val body: String,
)

val sampleNotes = listOf(
    Note(1, "Купити молоко", "І хліб, якщо буде свіжий"),
    Note(2, "Урок 5", "LazyColumn показує список без гальм"),
    Note(3, "Подзвонити мамі", "У неділю після обіду"),
)
```

`data class` — клас, для якого Kotlin сам генерує `equals`, `hashCode`,
`toString` і `copy`. Останній знадобиться постійно: у Compose стан прийнято не
змінювати, а замінювати новою копією:

```kotlin
val edited = note.copy(title = "Новий заголовок")
```

Усі поля тут `val`, тобто об'єкт незмінний (immutable). Це не примха: Compose
порівнює старі й нові дані, щоб зрозуміти, що перемальовувати. Якщо об'єкт
змінюється «на місці», порівнювати нема з чим — і екран може не оновитись.

## Крок 2. Екран зі списком

Створи `…/androidstarter/ui/NotesScreen.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.safeDrawingPadding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.runtime.toMutableStateList
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.webuxmotion.androidstarter.data.Note
import com.webuxmotion.androidstarter.data.sampleNotes

@Composable
fun NotesScreen(modifier: Modifier = Modifier) {
    val notes = remember { sampleNotes.toMutableStateList() }
    var draft by rememberSaveable { mutableStateOf("") }

    Column(
        modifier = modifier
            .fillMaxSize()
            .safeDrawingPadding(),
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
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
                    val nextId = (notes.maxOfOrNull { it.id } ?: 0) + 1
                    notes.add(0, Note(nextId, draft.trim(), "Створено вручну"))
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
                    onDelete = { notes.remove(note) },
                )
            }
        }
    }
}

@Composable
private fun NoteCard(
    note: Note,
    onDelete: () -> Unit,
) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = note.title,
                style = MaterialTheme.typography.titleMedium,
            )
            Text(
                text = note.body,
                style = MaterialTheme.typography.bodyMedium,
            )
            TextButton(onClick = onDelete) {
                Text("Видалити")
            }
        }
    }
}
```

## Крок 3. Показати новий екран

У `MainActivity.kt` заміни виклик `HelloScreen()` на `NotesScreen()` і додай
імпорт:

```kotlin
import com.webuxmotion.androidstarter.ui.NotesScreen
```

Стару функцію `HelloScreen` разом із її `@Preview` можна видалити — вона більше
не потрібна. Якщо шкода, лиши: невикликана `@Composable`-функція нікому не
заважає.

```sh
./gradlew installDebug
```

Введи текст, натисни «Додати» — нотатка з'явиться згори списку. «Видалити» —
зникне.

## Крок 4. Чому `LazyColumn`, а не `Column`

`Column` створює **всі** свої елементи одразу. Для трьох нотаток різниці немає,
для тисячі — застосунок задумається на секунди й з'їсть пам'ять.

`LazyColumn` створює лише те, що видно на екрані (плюс невеликий запас), а при
прокручуванні перевикористовує елементи, що поїхали за край. Це аналог
`RecyclerView` зі старого Android, тільки без адаптерів і `ViewHolder`.

Синтаксис усередині відрізняється: це не звичайний `@Composable`-блок, а
**DSL-опис** списку. Тому там не можна просто написати `Text(...)` — треба
через `item { }` або `items(list) { }`:

```kotlin
LazyColumn {
    item { Text("Заголовок списку") }        // один елемент
    items(notes) { note -> NoteCard(note) }  // багато елементів
    item { Text("Кінець") }
}
```

## Крок 5. Навіщо `key`

```kotlin
items(notes, key = { note -> note.id }) { … }
```

Без `key` Compose ототожнює елементи за позицією. Видалиш перший — і всі решта
«зсунуться», а Compose вважатиме, що кожен елемент змінив вміст: анімації
смикаються, стан усередині елемента (наприклад, розгорнутий текст) переїжджає
не туди.

З `key` елемент упізнається за ідентифікатором, і Compose розуміє, що саме цей
елемент зник, а решта лишились собою. Правило просте: **у списку, який
змінюється, `key` обов'язковий.**

## Крок 6. `mutableStateListOf` замість звичайного списку

```kotlin
val notes = remember { sampleNotes.toMutableStateList() }
```

Звичайний `mutableListOf` тут не спрацював би: Compose не дізнався б про
`notes.add(...)` і не перемалював екран. `toMutableStateList()` робить список
**спостережуваним** — будь-яка зміна вмісту запускає рекомпозицію.

Правило, яке варто запам'ятати: у Compose стан живе або в `mutableStateOf`, або
в спеціальних спостережуваних колекціях (`mutableStateListOf`,
`mutableStateMapOf`).

## Крок 7. Зафіксувати

```sh
git add -A
git commit -m "Урок 5: список нотаток"
```

## Що змінилось у git

```
app/src/main/java/.../data/Note.kt        (новий)
app/src/main/java/.../ui/NotesScreen.kt   (новий)
app/src/main/java/.../MainActivity.kt     (змінений — лише виклик екрана)
```

`MainActivity` схудла: вона більше не описує UI, а лише запускає його. Так і
має бути — надалі вона майже не змінюватиметься. Пакети `data` і `ui` — перший
натяк на архітектуру: **дані окремо, екрани окремо**.

## Вправа

1. Додай лічильник у шапку: «Нотаток: N». Він оновиться сам — переконайся, що
   для цього нічого «оновлювати» не треба.
2. Зроби так, щоб натискання на картку розгортало/згортало текст нотатки
   (підказка: `var expanded by remember { mutableStateOf(false) }` **всередині**
   `NoteCard` + `Modifier.clickable { … }`; тут якраз проявиться користь `key`).
3. Додай `item { }` із заголовком «Мої нотатки» перед списком.
4. Що станеться, якщо прибрати `key`? Видали кілька нотаток і подивись уважно.

## Далі

[Урок 6. Навігація між екранами](06-navigation.md)
