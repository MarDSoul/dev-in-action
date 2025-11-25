# Using MockK library in Jetpack Compose Preview
**Date of publication:** 02 Nov 2025

## Starting
What is Preview? It's a functionality that gives your an opportunity for view a composable UI during development without the need to build and run on an emulator. *(If you use the Android Studio obviously)*

## Problem
In fact you must have a some data that you wanna see in preview. But sometimes it's impossible.

Imagine, you have a screen like this:
```kotlin
@Composable
fun SomeScreen(
    viewModel: SomeViewModel = hiltViewModel(),
){
    //some UI
}
```

And view model of that screen like this:
```kotlin
@HiltViewModel
class SomeViewModel @Inject constructor(
    private val someUseCase1: someUseCase1,
    private val someUseCase2: someUseCase2
    //etc
) : ViewModel() {

    val someDataFLow1 = someUseCase1.getSomeData()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = null
        )

    //etc
}
```

And that's all. How can you make a preview? - Directly, no how.

## Solution
Use [MockK](https://mockk.io/) library. It's an open-source and excellent library for **testing**. Okay, it's for testing... But why can't we use it for debugging and preview? - Of course, we can!

First of all we have to add dependencies
```kotlin
//Compose
implementation(platform("androidx.compose:compose-bom:<bom-version>"))
//other compose dependencies

//Preview
debugImplementation("androidx.compose.ui:ui-tooling")
debugImplementation("androidx.compose.ui:ui-tooling-preview")
//MockK
debugImplementation("io.mockk:mockk:<mockk-version>")
```

Here's our project structure
```
└── src/
    ├── main/
    │   └── kotlin/
    │       └── yourpackage/
    │           ├── SomeScreen.kt      
    │           └── SomeViewModel.kt    
    │
    └── debug/
        └── kotlin/
            └── yourpackage/
                └── preview/
                      ├── SomeScreenPreview.kt
                      └── MockEntities.kt
```

We have to add a mock ViewModel by [MockK](https://mockk.io/) library and setup it's behavior:
```kotlin
internal object MockEntities {
    private val mockData = listOf(
        SomeDataItem(id = 1),
        SomeDataItem(id = 2),
        SomeDataItem(id = 3),
        SomeDataItem(id = 4),
    )
    
    private val mockDataFlow1 = MutableStateFlow(mockData)

    val someViewModel = mockk<SomeViewModel>().apply {
        every { someDataFLow1 } returns mockDataFlow1
    }
}
```

And a final function is a Preview itself:
```kotlin
@Composable
@Preview
private fun SomeScreenPreview() {
    YourTheme {
        SomeScreen(
            viewModel = MockEntities.someViewModel
        )
    }
}
```
---

So, that was a little trick. I hope it helps you in your work :)
