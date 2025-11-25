# Implementing Chaos-Proof Custom Typography in Compose
**Date of publication:** 26 Nov 2025

## Problem case
Recently I met a case when the typography system in design was so terrible that I spent lots of time with tries to make it into some more beautiful code. What was in that case? - Many different font weghts and sizes.

You may ask: "And so what?" - There were simply too many, such as on one screen a title can be with size 60px and font weight SemiBold, on another - size 54px and font weght Medium. And lots of them... The font list was impossble to be grouped into the Material Design typography system (you know, DisplayLarge, DisplayMedium and so on).

*Earlier I wanna named this article like a "How to be if your designer is an ass-hole" but it have been incorrectly because in generally I meet professionals. It was an exception case.*

---

## Local solution

### On-site definition

Let's define text style in place.

```kotlin
@Composable
fun SomeScreen() {

    val textStyle = remember {
        TextStyle(
            fontFamily = <family>,
            fontWeight = <weght>,
            fontSize = <size>,
        )
    }
    
    Text(
        text = "some text",
        style = textStyle
    )
    
// etc

}
```

**Pros:**

- easy
- visible
- easy to maintain if the local changes is needs

**Cons:**

- unusable if you need apply in another functions
- `remember` block or composable resources *(can't use composable functions in the `remember` block)*
- hard to maintain if the global changes is needs

**Summary:**
For my case it's the worst approach. But in another cases like, you have a clean design with consistent typography and in one place you have to set some exclusive font, it make sense.

### In the module defenition

Like the previous approach you define text styles but at another visible level - in the module.

```kotlin
internal object ModuleTypography {
    val titleStyle = TextStyle(
        fontFamily = <family>,
        fontWeight = <weight>,
        fontSize = <size>,
    )

    val contentStyle = TextStyle(
        fontFamily = <family>,
        fontWeight = <weight>,
        fontSize = <size>,
    )
    
//etc

}
```

**Pros:**

- easy
- visible
- available in module

**Cons:**

- unusable if you need apply in another module
- duplicate styles in different modules

**Summary:**
It's quite a neat approach. And in case you have different fonts mostly strongly by features it make sense.

## Custom Typography system

### Create a system

So, okay, the traditional system is insufficient for us. It means that we have to make our own typography.

```kotlin
data class AppTypo(
    val topBar: TextStyle,
    val screen1: Screen1Typo,
    val screen2: Screen2Typo,
// etc
)

data class Screen1Typo(
    val tableTitle: TextStyle,
    val tableContent: TextStyle,
    val common: TextStyle,
    
// etc

)

// etc
```

Now we have to define values. We can can set them to a class constructor as a default or make a singletone variable.

```kotlin
internal val CustomTypo = AppTypo(
    topBar = TextStyle(
        fontFamily = <family>,
        fontWeight = <weight>,
        fontSize = <size>,
    ),
    screen1 = Screen1Typo(
        tableTitle = TextStyle(
            fontFamily = <family>,
            fontWeight = <weight>,
            fontSize = <size>,
        ),
        tableContent = TextStyle(
            fontFamily = <family>,
            fontWeight = <weight>,
            fontSize = <size>,
        ),
        
// etc

)
```

### Integrate to Compose

For this action we will use [CompositionLocal](https://developer.android.com/develop/ui/compose/compositionlocal), in simple words it's a dependency injection for Compose UI system.

 - define the `CompositionLocal`

```kotlin
internal val LocalAppTypography = staticCompositionLocalOf { CustomTypo } //static for avoiding tracking
```

- provide into the whole theme

```kotlin
@Composable
fun MaterialStaticTheme(
    isDark: Boolean = true,
    content: @Composable () -> Unit,
) {

//some code

    CompositionLocalProvider(
        LocalAppTypography provides CustomTypo
    ) {
        MaterialTheme() {
            content()
        }
    }
    
//some code

}
```

- add direct access to an object

```kotlin
object AppTheme {
    val typo: AppTypo
        @Composable
        @ReadOnlyComposable
        get() = LocalAppTypography.current
}
```

- use in any place of UI

```kotlin
@Composable
fun SomeScreen() {

//some code
    
    Text(
        text = "some text",
        style = AppTheme.typo.screen1.tableContent
    )
    
//some code

}
```

#### Additional

It make sense to create a single point of enter to application theme.

```kotlin
object AppTheme {
    val typo: AppTypo
        @Composable
        @ReadOnlyComposable
        get() = LocalAppTypography.current

    val colorScheme: ColorScheme
        @Composable
        @ReadOnlyComposable
        get() = MaterialTheme.colorScheme

    val shapes: Shapes
        @Composable
        @ReadOnlyComposable
        get() = MaterialTheme.shapes
}
```

In that case you can use `AppTheme` instead of `MaterialTheme` object also for colors and shapes.

**Pros:**

- available in the whole app
- easy for maintaining

**Cons:**

- much boilerplate code

**Summary:**
So, this solution is for cases like mine. If there are so many styles that it can't be grouped to standard typography.

---

*Thanks for reading. I wish you good designers!* :fontawesome-regular-face-smile:

