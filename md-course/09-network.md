# Урок 9. Мережа: завантаження даних з API

**Мета:** замінити вигадані нотатки справжніми даними з інтернету й навчитися
правильно показувати три стани будь-якого екрана: **завантаження**, **дані**,
**помилка**.

Джерело — [jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com),
безкоштовне тестове API без реєстрації та ключів.

## Крок 1. Дозвіл на інтернет

Без цього рядка запит впаде з `SecurityException`, і це класична перша
помилка новачка.

`app/src/main/AndroidManifest.xml`, **перед** тегом `<application>`:

```xml
    <uses-permission android:name="android.permission.INTERNET" />
```

Цей дозвіл — «звичайний» (normal): система видає його при встановленні, нічого
питати в користувача не треба. А от геолокація чи камера — «небезпечні»
(dangerous), їх треба запитувати в рантаймі окремим діалогом.

## Крок 2. Підключити Retrofit

`gradle/libs.versions.toml`:

```toml
[versions]
retrofit = "2.11.0"

[libraries]
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
```

`app/build.gradle.kts`:

```kotlin
    implementation(libs.retrofit)
    implementation(libs.retrofit.converter.gson)
```

**Retrofit** перетворює HTTP-запити на виклики методів Kotlin, а **конвертер** —
JSON на об'єкти. Одна версія на дві бібліотеки (`version.ref = "retrofit"`) —
їх завжди оновлюють разом.

## Крок 3. Описати API

Створи `…/androidstarter/data/remote/NotesApi.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.data.remote

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.http.GET

data class PostDto(
    val id: Int,
    val title: String,
    val body: String,
)

interface NotesApi {
    @GET("posts")
    suspend fun posts(): List<PostDto>
}

object NetworkModule {
    val api: NotesApi = Retrofit.Builder()
        .baseUrl("https://jsonplaceholder.typicode.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build()
        .create(NotesApi::class.java)
}
```

Про що тут варто знати:

- **Інтерфейс без реалізації.** Retrofit генерує її сам у рантаймі за
  анотаціями. `@GET("posts")` + `baseUrl` дають
  `https://jsonplaceholder.typicode.com/posts`. `baseUrl` **мусить**
  закінчуватись на `/` — інакше падіння при старті.
- **`suspend`** перетворює запит на звичайний послідовний код: жодних колбеків,
  Retrofit сам виконає його у фоновому потоці.
- **DTO ≠ модель.** `PostDto` повторює формат сервера, `Note` — те, що потрібно
  застосунку. Розділяти їх варто одразу: сервер зміниться, а екрани — ні,
  правити доведеться лише перетворення.
- **`object`** — синглтон Kotlin. Retrofit важкий, створювати його на кожен
  запит не можна.

## Крок 4. Три стани в `NotesUiState`

Заміни вміст `…/androidstarter/ui/NotesViewModel.kt` — **повний вміст**:

```kotlin
package com.webuxmotion.androidstarter.ui

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.webuxmotion.androidstarter.data.Note
import com.webuxmotion.androidstarter.data.remote.NetworkModule
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

data class NotesUiState(
    val notes: List<Note> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
)

class NotesViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(NotesUiState())
    val uiState: StateFlow<NotesUiState> = _uiState.asStateFlow()

    init {
        refresh()
    }

    fun refresh() {
        viewModelScope.launch {
            _uiState.update { state -> state.copy(isLoading = true, error = null) }

            val result = runCatching { NetworkModule.api.posts() }

            result.onSuccess { posts ->
                val notes = posts.take(20).map { post ->
                    Note(id = post.id, title = post.title, body = post.body)
                }
                _uiState.update { state -> state.copy(isLoading = false, notes = notes) }
            }

            result.onFailure { throwable ->
                _uiState.update { state ->
                    state.copy(
                        isLoading = false,
                        error = throwable.message ?: "Не вдалося завантажити",
                    )
                }
            }
        }
    }

    fun addNote(title: String) {
        if (title.isBlank()) return
        _uiState.update { state ->
            val nextId = (state.notes.maxOfOrNull { it.id } ?: 0) + 1
            state.copy(notes = listOf(Note(nextId, title, "Створено вручну")) + state.notes)
        }
    }

    fun deleteNote(note: Note) {
        _uiState.update { state ->
            state.copy(notes = state.notes.filterNot { it.id == note.id })
        }
    }
}
```

- **`init { refresh() }`** — завантаження стартує при створенні `ViewModel`,
  тобто один раз, а не на кожну рекомпозицію. Це важливо: якби запит стояв прямо
  в `@Composable`, він летів би десятки разів на секунду.
- **`runCatching { }`** ловить будь-який виняток і повертає `Result`. Мережа
  падає постійно: немає зв'язку, таймаут, сервер віддав 500 — і кожен такий
  випадок без обробки означає падіння застосунку.
- Локальне додавання нотаток лишилось: воно живе лише в пам'яті й зникне після
  `refresh()`. Це нормально для навчального прикладу — «правильне» рішення
  вимагає локальної бази як єдиного джерела істини.

`data/Note.kt` тепер можна почистити: `sampleNotes` більше не використовується.
Прибери його або лиши для `@Preview`.

## Крок 5. Показати стани на екрані

У `NotesScreen.kt` зміни сигнатуру й тіло списку.

**5.1.** Сигнатура — три нових параметри:

```kotlin
fun NotesScreen(
    notes: List<Note>,
    isLoading: Boolean,
    error: String?,
    onAdd: (String) -> Unit,
    onDelete: (Note) -> Unit,
    onOpen: (Int) -> Unit,
    onRefresh: () -> Unit,
    onToggleTheme: () -> Unit,
    modifier: Modifier = Modifier,
) {
```

**5.2.** Заміни блок `LazyColumn { … }` на:

```kotlin
            when {
                isLoading -> {
                    Box(
                        modifier = Modifier.fillMaxSize(),
                        contentAlignment = Alignment.Center,
                    ) {
                        CircularProgressIndicator()
                    }
                }

                error != null -> {
                    Column(
                        modifier = Modifier
                            .fillMaxSize()
                            .padding(24.dp),
                        verticalArrangement = Arrangement.Center,
                        horizontalAlignment = Alignment.CenterHorizontally,
                    ) {
                        Text(
                            text = "Помилка: $error",
                            style = MaterialTheme.typography.bodyLarge,
                        )
                        Button(onClick = onRefresh) {
                            Text("Спробувати ще раз")
                        }
                    }
                }

                else -> {
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
```

**5.3.** Додай кнопку оновлення в `actions` шапки, поруч із «Тема»:

```kotlin
                actions = {
                    TextButton(onClick = onRefresh) { Text("Оновити") }
                    TextButton(onClick = onToggleTheme) { Text("Тема") }
                },
```

**5.4.** Нові імпорти:

```kotlin
import androidx.compose.foundation.layout.Box
import androidx.compose.material3.CircularProgressIndicator
```

`when { }` — це `if/else if/else` у Kotlin. Порядок гілок важливий: спершу
завантаження, потім помилка, і лише потім дані.

## Крок 6. Прокинути параметри в `AppNavHost.kt`

У виклику `NotesScreen` додай:

```kotlin
                isLoading = uiState.isLoading,
                error = uiState.error,
                onRefresh = viewModel::refresh,
```

## Крок 7. Перевірити всі три стани

```sh
./gradlew installDebug
```

1. **Дані.** При старті — короткий індикатор, потім 20 «нотаток» із сервера.
2. **Помилка.** Увімкни на телефоні режим польоту й натисни «Оновити» — має
   з'явитись текст помилки й кнопка повтору. З терміналу:

```sh
adb shell svc wifi disable && adb shell svc data disable   # вимкнути мережу
adb shell svc wifi enable  && adb shell svc data enable    # повернути
```

3. **Що відбувається насправді** — подивись логи застосунку:

```sh
adb logcat --pid=$(adb shell pidof com.webuxmotion.androidstarter)
```

## Крок 8. Зафіксувати

```sh
git add -A
git commit -m "Урок 9: завантаження нотаток з API"
```

## Що змінилось у git

```
app/src/main/AndroidManifest.xml                # +дозвіл INTERNET
gradle/libs.versions.toml                       # +1 версія, +2 бібліотеки
app/build.gradle.kts                            # +2 залежності
app/src/main/java/.../data/remote/NotesApi.kt   (новий)
app/src/main/java/.../ui/NotesViewModel.kt      (сильно змінений)
app/src/main/java/.../ui/NotesScreen.kt         (змінений)
app/src/main/java/.../ui/AppNavHost.kt          (+3 рядки)
```

Уперше за курс змінився **`AndroidManifest.xml`** — і це закономірність: маніфест
чіпають тоді, коли застосунку потрібне щось **від системи** (дозвіл, новий
екран-точка входу, фонова служба). Побачив зміну маніфесту в чужому коміті —
шукай, які нові можливості системи там задіяли.

## Вправа

1. Додай логування HTTP: залежність
   `com.squareup.okhttp3:logging-interceptor` і `OkHttpClient` з
   `HttpLoggingInterceptor` у `NetworkModule`. У logcat буде видно кожен запит і
   відповідь.
2. Зроби екран деталей самостійним: `@GET("posts/{id}")` і окремий запит замість
   пошуку в уже завантаженому списку.
3. Заміни `error = throwable.message` на людські тексти: окремо «немає
   інтернету» (`java.io.IOException`) і окремо «сервер відповів помилкою»
   (`retrofit2.HttpException`).

## Далі

[Урок 10. Реліз: іконка, назва, підписаний APK](10-release.md)
