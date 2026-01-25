# Coroutines: auto-cancel pattern

- date: Jan 2026
- link: [AsyncImagePainter.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-compose-core/src/commonMain/kotlin/coil3/compose/AsyncImagePainter.kt) from [Coil 3.3.0](https://github.com/coil-kt/coil/tree/3.3.0) under [Apache-2.0 License](https://github.com/coil-kt/coil/blob/3.3.0/LICENSE.txt)

When needs to cancel coroutine:

```kotlin
private var coroutineJob: Job? = null
    set(value) {
        field?.cancel()
        field = value
    }

fun nullAndCancelJob() {
    coroutineJob = null
}
```

