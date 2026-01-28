# Save money on image loading with Coil library
**Date of publication:** 28 Jan 2026

## Problem

**Coil** is an amazing library and prevent trafic costs effectively if an element with loading image from network doesn't need more. Example in case with `LazyColumn` by scrolling down fast. Regardless of this there is a key problem with payment API: the money is spent with connection, not with loading. So, I decided to prevent a request to the network with fast scrolling list.

## Solution

So, the solution is pretty obvious - we have to reject request before it leaves a device. And with this task the `Fetcher` will help. [Link to official docs](https://coil-kt.github.io/coil/image_pipeline/#chaining-components).

### Steps

1. Create our own `Fetcher` with debounce

    ```kotlin
    class DebounceFetcher(private val delay: Long) : Fetcher {
        override suspend fun fetch(): FetchResult? {
            delay(delay)
            return null
        }

        class Factory(private val delay: Long) : Fetcher.Factory<Any> {
            override fun create(
                data: Any,
                options: Options,
                imageLoader: ImageLoader
            ): Fetcher? {
                val dataStr = data.toString()
                val isNetwork = (dataStr.startsWith("http") || dataStr.startsWith("https")) //depends on type of data
                return if (isNetwork) DebounceFetcher(delay) else null
            }
        }
    }
    ```

2. And we have to add our `DebounceFetcher` into `ImageLoader`.

    ```kotlin
    val debounceImageLoader = ImageLoader(this).newBuilder()
        .components {
            add(DebounceFetcher.Factory(200L))
        }
        .build()
    ```

3. At last adding `ImageLoader` into `AsyncImage`.

    ```kotlin
    AsyncImage(
        model = "link to load image",
        contentDescription = null,
        imageLoader = debounceImageLoader
    )
    ```

### What happens?

`AsyncImage` appears on a screen as an item of `LazyColumn` -> our `DebounceFetcher` as the first of chain of *fetchers* suspend coroutine -> with fast scroll the item move away from screen -> the couroutine is cancelled -> all chain is cancel -> profit

## Testing on the real device with real payment API

### Case description

Link to repository with code example: [link](https://github.com/MarDSoul/example-coil-list).

Requirements:

- caching of **Coil** is disabled for clear testing amount of requests
- there are 2 lists:
    - list *AsIs* - for default behavior of **Coil** (without caching) with `LazyColumn`
    - list *Optimized* - for custom `ImageLoader`
- payment API: [Google Maps Static API](https://developers.google.com/maps/documentation/maps-static)
- main metric: difference of sent requests between default behavior and *optimized* behavior
- secondary metric: performance with working on budget device

### Device params

- model: [Cubot KingKong Mini 2](https://cubot.net/phones/rugged-phone/KingKong-mini-2-specs/60)
- CPU: MT6761, Quad-Core, 64-bit / GPU: PowerVR GE8300
- OS: Android 10 (32-bit version)
- heap limit: 256 MB
- display: 4.0''

*It's funny but the vendor setup 32-bit version of OS on the device with 64-bit arch. Also he limited heap to 256MB only.*

```
:~$ adb devices
List of devices attached
KKMN2201127003240	device

:~$ adb shell getprop ro.product.cpu.abilist
armeabi-v7a,armeabi

:~$ adb shell getprop | grep dalvik.vm.heapgrowthlimit
[dalvik.vm.heapgrowthlimit]: [256m]
```

As you may see, it's a budget device, far-far from flagships. On this kind of devices it's a **crucial** struggling for each MB in memory. And this kind of devices makes more visible issues with requests.

### Results

Scenario:

1. Start from `Empty` screen
2. Switch to `AsIs` screen
3. Aggresive scroll down to last and next up to first
4. Get logs
5. Switch to `Empty` screen
6. Switch to `AsIs` screen
7. Smooth scroll down to last and next up to first
8. Get logs
9. Switch to `Empty` screen
10. Switch to `Optimized` screen
11. Aggresive scroll down to last and next up to first
12. Get logs
13. Switch to `Empty` screen
14. Switch to `Optimized` screen
15. Smooth scroll down to last and next up to first
15. Get logs

| Screen | Scroll | Logs |
| --- | --- | --- |
| `AsIs` | aggressive | `Total: 195, Saved: 0, Executed: 195 (Efficiency: 0%)` |
| `AsIs` | smooth | `Total: 195, Saved: 0, Executed: 195 (Efficiency: 0%)` |
| `Optimized` | aggressive | `Total: 195, Saved: 112, Executed: 83 (Efficiency: 57%)` |
| `Optimized` | smooth | `Total: 195, Saved: 0, Executed: 195 (Efficiency: 0%)` |

??? info "Video aggressive"  
    | `AsIs` | `Optimized` |
    | :---: | :---: |
    | ![type:video](../assets/videos/asis_aggressive.mp4){ width="300" } | ![type:video](../assets/videos/optimized_aggressive.mp4){ width="300" } |
    | 🔴 195 requests | 🟢 83 requests |

??? info "Video smooth"  
    | `AsIs` | `Optimized` |
    | :---: | :---: |
    | ![type:video](../assets/videos/asis_smooth.mp4){ width="300" } | ![type:video](../assets/videos/optimized_smooth.mp4){ width="300" } |
    | 195 requests | 195 requests |

I dunno can you see on video freezes with aggressive scrolling? I feel :) With `AsIs` aggresive case there are lots of requests force wokr hard CPU. Unfortunatly, I can't start profiler on this device - it just stop works...

## Summary

Hmm... what can I summarize? You may see that with aggressive scroll of list you can prevent 57% of requests. Of course, in reality, the number will be lower (I achieved about 8% in average). But you have to comprehend that 8% or even 1-2% means **thousands of dollars** on payment API. It's definitely worth the effort of 10 minutes of writing custom *fetcher*.

**Note:** there was one of many variants of optimizing scrolling list to prevent unnecessary requests. You may turn on caches, compare links or using [schedulePrefetch](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/LazyListPrefetchScope#schedulePrefetch(kotlin.Int,kotlin.Function1)) from new [Compose 1.10](https://developer.android.com/jetpack/androidx/releases/compose-foundation#version_110_3) or something else.


*Okay, the theory was confirmed, the solution works, the money is saving :)*

*Deep diving to the Coil's entrails is below*

---

## Coil is not a magic

All code snippets in this topic are received from the official [Coil repository (release 3.3.0)](https://github.com/coil-kt/coil/tree/3.3.0) under the [Apache-2.0 License](https://github.com/coil-kt/coil/blob/3.3.0/LICENSE.txt).

### Step 1. UI: the trigger

- [AsyncImage.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-compose-core/src/commonMain/kotlin/coil3/compose/AsyncImage.kt)

Public composable `AsycImage` delegates to private `AsyncImage` which create `ContentPainterElement`.

- [ContentPainterModifier.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-compose-core/src/commonMain/kotlin/coil3/compose/internal/ContentPainterModifier.kt)

`ContentPainterElement` creates `ContentPainterNode` which received `AsyncImagePainter`.

```kotlin
internal class ContentPainterNode(
    override val painter: AsyncImagePainter,
    //other params
) : AbstractContentPainterNode(
    //setup fields from abstract class
) {

    override fun onAttach() { //attention!
        painter.scope = coroutineScope
        painter.onRemembered()
    }

    override fun onDetach() { //attention!
        painter.onForgotten()
    }

    //some code
}
```

Pay attention on `ContentPainterNode` class. There's the bridge between UI's and request's events: `onAttach()` (when we can see item of `LazyColumn` on the screen) - `onRemembered()`; `onDetach()` (when that element move outside of screen) - `onForgotten()`. 

### Step 2. Bridge

- [AsyncImagePainter.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-compose-core/src/commonMain/kotlin/coil3/compose/AsyncImagePainter.kt)

So, here's the coroutine is cancelled and throws `CancellationException` as a reaction to `onDetach()`. Also `AsyncImagePainter` delegates execution to `ImageLoader`.

```kotlin
@Stable
class AsyncImagePainter internal constructor(
    input: Input,
) : Painter(), RememberObserver {

    //some fields

    private var rememberJob: Job? = null //nice cancellation. Like!
        set(value) {
            field?.cancel()
            field = value
        }

    //some fields

    internal lateinit var scope: CoroutineScope //defined in onAttach() method above

    //some code

    override fun onRemembered() = trace("AsyncImagePainter.onRemembered") {
        (painter as? RememberObserver)?.onRemembered()
        launchJob()
        isRemembered = true
    }

    private fun launchJob() {
        val input = _input ?: return

        rememberJob = scope.launchWithDeferredDispatch {
                //some code
                input.imageLoader.execute(request).toState() //key function
                //some code
        }
    }

    override fun onForgotten() { //point with cancellation
        rememberJob = null
        //some code
    }

    override fun onAbandoned() {
        rememberJob = null
        //some code
    }

    //some code

}
```

- [DefferedDispatch.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-compose-core/src/commonMain/kotlin/coil3/compose/internal/DeferredDispatch.kt)

Custom wrap on `launch`. *Here's the magic of 5th level. Amazing and elegance solution! The guys on the Coil team are geniuses!*

- [ImageLoader.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-core/src/commonMain/kotlin/coil3/ImageLoader.kt), [RealImageLoader.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-core/src/commonMain/kotlin/coil3/RealImageLoader.kt)

```kotlin
internal class RealImageLoader(
    val options: Options,
) : ImageLoader {
    //fields
    override val components = options.componentRegistry.newBuilder()
        .addServiceLoaderComponents(options)
        .addAndroidComponents(options)
        .addJvmComponents(options)
        .addAppleComponents(options)
        .addCommonComponents()
        .add(EngineInterceptor(this, systemCallbacks, requestService, options.logger)) //EngineInterceptor like a final gate before network request
        .build()


    private suspend fun execute(initialRequest: ImageRequest, type: Int): ImageResult {
        //some code

        try {
            //some code

            // Execute the interceptor chain.
            val result = withContext(request.interceptorCoroutineContext) {
                RealInterceptorChain(
                    //params
                ).proceed()
            }

            //some code
        } catch (throwable: Throwable) {
            if (throwable is CancellationException) {
                onCancel(request, eventListener)
                throw throwable

            //else, finally etc.
        }
    }

    //some code
}
```

Okay, `execute()` is a *suspend* functions, so, when we cancel coroutine above, the coroutine will stop onto next suspend point.

### Step 3. Interceptors

- [RealInterceptorChain.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-core/src/commonMain/kotlin/coil3/intercept/RealInterceptorChain.kt)

The `proceed()` function calls *interceptors* by chain and the last is `EngineInterceptor`.

- [EngineInterceptor.kt](https://github.com/coil-kt/coil/blob/3.3.0/coil-core/src/commonMain/kotlin/coil3/intercept/EngineInterceptor.kt)

```kotlin
internal class EngineInterceptor(
    //params
) : Interceptor {
    //some code

    override suspend fun intercept(chain: Interceptor.Chain): ImageResult {
        try {
            //define variables, cache etc.

            return withContext(request.fetcherCoroutineContext) {
                val result = execute(request, mappedData, options, eventListener) //go to :)

                //some code

        } catch (throwable: Throwable) {
            if (throwable is CancellationException) {
                throw throwable
            //else block
        }
    }

    private suspend fun execute(
        //parms
    ): ExecuteResult {
        //variables
        //some code

        fetchResult = fetch(components, request, mappedData, options, eventListener) //go to next

        //some code
    }

    private suspend fun fetch(
        //params
    ): FetchResult {
        val fetchResult: FetchResult
        var searchIndex = 0
        while (true) {
            val pair = components.newFetcher(mappedData, options, imageLoader, searchIndex)
            checkNotNull(pair) { "Unable to create a fetcher that supports: $mappedData" }
            val fetcher = pair.first
            searchIndex = pair.second + 1

            //some code

            val result = fetcher.fetch() //our custom DebounceFetcher call suspend delay() and return null

            //some code

            if (result != null) {
                fetchResult = result
                break
            }
        }
        return fetchResult
    }

    //some code
}
```

Okay, Coil automatically set our custom *fetcher* at the start of fetcher's list, so, it will start before network *fetcher*. And as the result we have `delay()` and some time with the opportunity to drop request before going to the Internet.

*Huh... that was a long way...*

---

*Colleague, save the customer's money! Thanks for reading!*
