# Lab 3 - Dynamic City List + Participation Exercise

This writeup covers the completed Lab 3 participation exercise using the **six hints from the lab handout**.

The guided portion of the lab already makes the city list observable, adds City/Province input fields, and hides the Add City controls behind a floating `+` button. The participation exercise extends the app so a user can **select an existing city, edit it, and replace it in the list while the app is running**.

---

# Final Behavior

After the participation exercise is complete:

- The original city list still displays normally.
- `+` opens the Add City controls.
- New cities can still be added.
- Clicking an existing city selects it for editing.
- The selected city's current name and province appear in edit fields.
- Pressing **Update City** replaces the original `City` object.
- The list updates immediately because the repository uses `mutableStateListOf`.
- No permanent storage is required.

---

# Participation Exercise Walkthrough

## Hint 1 - Keep track of the selected city

Inside `CityListScreen`, add state to remember which city the user clicked:

```kotlin
var selectedCity by remember { mutableStateOf<City?>(null) }
```

`City?` is used because there may be no selected city. Initially:

```kotlin
selectedCity == null
```

Once the user clicks a row:

```kotlin
selectedCity = city
```

We also add state for the editable values:

```kotlin
var updatedCityName by remember { mutableStateOf("") }
var updatedProvinceName by remember { mutableStateOf("") }
```

The relevant state at the top of `CityListScreen` becomes:

```kotlin
var newCityName by remember { mutableStateOf("") }
var newProvinceName by remember { mutableStateOf("") }
var showAddCityFields by remember { mutableStateOf(false) }

var selectedCity by remember { mutableStateOf<City?>(null) }
var updatedCityName by remember { mutableStateOf("") }
var updatedProvinceName by remember { mutableStateOf("") }
```

---

## Hint 2 - Make each city row clickable

Add the import:

```kotlin
import androidx.compose.foundation.clickable
```

Change `CityRow` so it receives an `onClick` callback:

```kotlin
@Composable
fun CityRow(
    city: City,
    onClick: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(onClick = onClick)
            .padding(horizontal = 20.dp, vertical = 16.dp)
    ) {
        Text(
            text = city.name,
            fontSize = 30.sp,
            modifier = Modifier.weight(1f)
        )

        Text(
            text = city.province,
            fontSize = 30.sp,
            modifier = Modifier.weight(1f)
        )
    }
}
```

Now each city can report when it is clicked.

Inside the `LazyColumn`, pass an action into `CityRow`:

```kotlin
CityRow(
    city = city,
    onClick = {
        selectedCity = city
        updatedCityName = city.name
        updatedProvinceName = city.province
        showAddCityFields = false
    }
)
```

When a city is clicked:

1. The original `City` object is stored in `selectedCity`.
2. Its name is copied into `updatedCityName`.
3. Its province is copied into `updatedProvinceName`.
4. Add mode is closed so Add and Edit controls are not shown at the same time.

---

## Hint 3 - Add fields for the updated name and province

Display the edit controls only if a city has been selected:

```kotlin
if (selectedCity != null) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        OutlinedTextField(
            value = updatedCityName,
            onValueChange = { updatedCityName = it },
            label = { Text("Updated City") },
            modifier = Modifier.weight(1f)
        )

        Spacer(modifier = Modifier.width(8.dp))

        OutlinedTextField(
            value = updatedProvinceName,
            onValueChange = { updatedProvinceName = it },
            label = { Text("Updated Province") },
            modifier = Modifier.weight(1f)
        )

        Spacer(modifier = Modifier.width(8.dp))

        Button(
            modifier = Modifier.padding(vertical = 12.dp),
            onClick = {
                // update logic added below
            }
        ) {
            Text("Update City")
        }
    }
}
```

Because the values were copied when the row was clicked, the user edits the city's existing values instead of starting with empty fields.

---

## Hint 4 - Add `updateCity` to `CityRepository`

The repository is responsible for changing the city list.

Add:

```kotlin
fun updateCity(oldCity: City, updatedCity: City) {
    val index = _cities.indexOf(oldCity)

    if (index != -1) {
        _cities[index] = updatedCity
    }
}
```

The method:

1. Finds the position of the original city.
2. Checks that the city actually exists.
3. Replaces the old object with the updated object.

The completed repository is:

```kotlin
package com.example.listycity3

import androidx.compose.runtime.mutableStateListOf

class CityRepository {
    private val _cities = mutableStateListOf(
        City("Edmonton", "AB"),
        City("Vancouver", "BC"),
        City("Toronto", "ON")
    )

    val cities: List<City>
        get() = _cities

    fun addCity(city: City) {
        _cities.add(city)
    }

    fun updateCity(oldCity: City, updatedCity: City) {
        val index = _cities.indexOf(oldCity)
        if (index != -1) {
            _cities[index] = updatedCity
        }
    }
}
```

Because `_cities` is a `mutableStateListOf`, replacing an item causes Compose to update the visible list.

---

# Connect the update callback through `MainActivity`

`CityListScreen` should not directly own the repository. Instead, give it an update callback.

Change the function header to:

```kotlin
@Composable
fun CityListScreen(
    cities: List<City>,
    onAddCity: (City) -> Unit,
    onUpdateCity: (City, City) -> Unit,
    modifier: Modifier = Modifier
) {
```

`onUpdateCity` receives:

```text
old City, updated City
```

Connect that callback in `MainActivity`:

```kotlin
CityListScreen(
    cities = cityRepository.cities,
    onAddCity = { cityRepository.addCity(it) },
    onUpdateCity = { oldCity, updatedCity ->
        cityRepository.updateCity(oldCity, updatedCity)
    },
    modifier = Modifier.padding(innerPadding)
)
```

The screen can now request an update without directly modifying the repository.

---

## Hint 5 - Permanent storage is not required

Nothing else needs to be added for persistence.

The city list only needs to stay updated while the application is running. When the app restarts, the repository is recreated with:

```kotlin
City("Edmonton", "AB")
City("Vancouver", "BC")
City("Toronto", "ON")
```

No Room database, file storage, DataStore, or SharedPreferences are required for this lab.

---

## Hint 6 - Replace the `City` instead of modifying its `val` properties

`City` is defined using immutable properties:

```kotlin
data class City(
    val name: String,
    val province: String
)
```

Therefore this is not allowed:

```kotlin
oldCity.name = updatedCityName
oldCity.province = updatedProvinceName
```

Instead, create a new `City` object inside the Update button:

```kotlin
Button(
    modifier = Modifier.padding(vertical = 12.dp),
    onClick = {
        val oldCity = selectedCity

        if (
            oldCity != null &&
            updatedCityName.isNotBlank() &&
            updatedProvinceName.isNotBlank()
        ) {
            val updatedCity = City(
                name = updatedCityName.trim(),
                province = updatedProvinceName.trim()
            )

            onUpdateCity(oldCity, updatedCity)

            selectedCity = null
            updatedCityName = ""
            updatedProvinceName = ""
        }
    }
) {
    Text("Update City")
}
```

This leaves the original city unchanged until the repository replaces it with the new object.

After the update, the selected city and edit fields are cleared so the edit controls disappear.

---

# Preview Update

Because `CityListScreen` now requires `onUpdateCity`, the preview also needs that callback:

```kotlin
@Preview(showBackground = true)
@Composable
fun CityListScreenPreview() {
    ListyCity3Theme {
        CityListScreen(
            cities = listOf(
                City("Edmonton", "AB"),
                City("Vancouver", "BC"),
                City("Calgary", "AB")
            ),
            onAddCity = {},
            onUpdateCity = { _, _ -> }
        )
    }
}
```

The preview does not need to actually update anything, so the callback can be empty.

---

# Small UI Adjustments

## Keep Add and Edit modes separate

When the `+` button opens the Add City controls, clear any current edit selection:

```kotlin
onClick = {
    val openingAddFields = !showAddCityFields
    showAddCityFields = openingAddFields

    if (openingAddFields) {
        selectedCity = null
        updatedCityName = ""
        updatedProvinceName = ""
    }
}
```

Likewise, selecting a city closes Add mode:

```kotlin
showAddCityFields = false
```

This prevents the Add and Edit forms from appearing at the same time.

## Let the list use the remaining screen space

Use:

```kotlin
LazyColumn(
    modifier = Modifier
        .weight(1f)
        .fillMaxWidth()
)
```

`weight(1f)` allows the list to occupy the remaining vertical space after the Add/Edit controls.

---


# Final Data Flow

When the user edits a city:

```text
User clicks city
      ↓
CityRow calls onClick
      ↓
CityListScreen stores selectedCity
      ↓
Current values are copied into edit fields
      ↓
User edits name/province
      ↓
User presses Update City
      ↓
A new City object is created
      ↓
onUpdateCity(oldCity, updatedCity)
      ↓
MainActivity calls CityRepository.updateCity(...)
      ↓
Repository replaces the old item in mutableStateListOf
      ↓
Compose redraws the updated list
```

---

# Pitfalls

- `selectedCity` should be `City?` because no city is selected initially.
- Use `remember { mutableStateOf(...) }` for interactive UI state.
- Import `androidx.compose.foundation.clickable` before using `.clickable(...)`.
- Put `.clickable(...)` on the full `CityRow` if the entire row should respond.
- Copy the selected city's current values into the edit fields when it is clicked.
- Keep Add state and Edit state separate (`newCityName` vs. `updatedCityName`).
- Add `onUpdateCity` everywhere `CityListScreen` is called, including the Preview.
- Check `index != -1` before replacing an item returned by `indexOf`.
- Do not try to assign to `city.name` or `city.province`; they are `val` properties.
- Use `isNotBlank()` so whitespace-only values are rejected.
- `trim()` removes accidental leading/trailing spaces before storing values.
- `mutableStateListOf` is important so Compose notices list changes.
- `remember` is not permanent storage; restarting the app resets the repository.
- If duplicate identical `City` objects exist, `indexOf` updates the first matching one.

---

# File Responsibilities

### `City.kt`
Defines the immutable city model:

```kotlin
data class City(
    val name: String,
    val province: String
)
```

### `CityRepository.kt`
Owns the observable city list and provides:

```kotlin
addCity(...)
updateCity(...)
```

### `CityListScreen.kt`
Owns the screen UI state, displays the list, handles selection, shows the Add/Edit controls, and calls callbacks.

### `MainActivity.kt`
Creates `CityRepository` and connects the screen callbacks to repository functions.

---

# Summary

The participation exercise adds editing without changing the overall structure of the app:

1. Remember the selected city.
2. Make each city row clickable.
3. Show editable fields for the selected city.
4. Add `updateCity` to the repository.
5. Keep everything in memory for the current app session.
6. Create a new `City` object and replace the original because `City` uses `val` properties.

The key idea is that `CityListScreen` manages the user's interaction, while `CityRepository` remains responsible for changing the city list.
