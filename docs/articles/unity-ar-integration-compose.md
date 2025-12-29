# Integration Unity AR to Compose
**Date of publication:** 29 Dec 2025

Okay, today I gonna talk about a such specific task as integration Unity library with AR functionality into Android application. Here I'll write about obstacles you can meet and solution I suggest.

Unity is engine that trying to completely replace the process of application. And the traditional way for implementing Unity is creation UnityPlayerActivity (or UnityGamePlayerActivity) which is creating by Unity IDE.

By default we can't rule this. And the main task in this article - **setup Unity as a simple composable function**.

*Let's go*

---

## Problems

### No ready-made solution

Sounds funny :) But really... first of all we wanna find some library for the issue. And we can find [Unity as a Library](https://unity.com/features/unity-as-a-library). And it works... sometimes. *I coldn't start this lybrary correctly. Maybe the reason was in lib version, maybe Unity version, maybe third-part AR library.* Anyway it's solution for implementing AR but by an hard copuling Activity with Unity engine.

### The specific of Unity lifecycle

We're all already used to it that an Activity has its lifecycle. The Unity also has its lifecycle. But their cycles a little bit different. For instance, you will be laughing but Unity send `SIGKILL` to process when its call `onDestroy()` function.

## Suggested solution

### The core of solution

The main idea - **rewrite the lifecycle of Unity engine**. The key class - `UnityPlayerForActivityOrService`. This class is from Unity when you import the project to Android `aar` library.

```kotlin
/**
 * Manages the UnityPlayer instance and its lifecycle.
 *
 * This object is responsible for creating, initializing, and destroying the UnityPlayer instance.
 * It also handles the UnityPlayer's lifecycle events by observing the lifecycle of the attached
 * activity.
 */
object UnityPlayerManager {
    private var currentActivity: ComponentActivity? = null
    private var currentLifecycleOwner: LifecycleOwner? = null
    private lateinit var unityPlayer: UnityPlayerForActivityOrService

    /**
     * Observes the lifecycle of the attached activity and calls the corresponding UnityPlayer
     * lifecycle methods.
     *
     * This observer ensures that the UnityPlayer's lifecycle is synchronized with the activity's
     * lifecycle.
     */
    var lifecycleEventObserver = LifecycleEventObserver { _, event ->
        when (event) {
            Lifecycle.Event.ON_START -> {
                unityPlayer.onStart()
            }

            Lifecycle.Event.ON_RESUME -> {
                unityPlayer.onResume()
                unityPlayer.windowFocusChanged(true)
            }

            Lifecycle.Event.ON_PAUSE -> {
                unityPlayer.windowFocusChanged(false)
                unityPlayer.onPause()
            }

            Lifecycle.Event.ON_STOP -> {
                unityPlayer.onStop()
            }

            Lifecycle.Event.ON_DESTROY -> {
                unityPlayer.destroy()
                detachActivity()
            }

            else -> {
            }
        }
    }

    /**
     * Attaches the UnityPlayerManager to the given activity.
     *
     * This method initializes the UnityPlayer instance and registers the lifecycle observer.
     *
     * @param activity The activity to attach to.
     */
    fun attachActivity(activity: ComponentActivity) {
        currentActivity = activity
        currentLifecycleOwner = activity
        initUnityPlayer(activity)
        currentLifecycleOwner?.lifecycle?.addObserver(lifecycleEventObserver)
    }

    /**
     * Detaches the UnityPlayerManager from the current activity.
     *
     * This method removes the lifecycle observer and clears the references to the activity and
     * lifecycle owner.
     */
    private fun detachActivity() {
        currentActivity = null
        currentLifecycleOwner?.lifecycle?.removeObserver(lifecycleEventObserver)
        currentLifecycleOwner = null
    }

    /**
     * Initializes the UnityPlayer instance.
     *
     * @param context The context to use for creating the UnityPlayer instance.
     */
    private fun initUnityPlayer(context: Context) {
        val lifecycleEvents = object : IUnityPlayerLifecycleEvents {
            override fun onUnityPlayerUnloaded() {
            }

            override fun onUnityPlayerQuitted() {
            }
        }
        unityPlayer = UnityPlayerForActivityOrService(context, lifecycleEvents)
    }

    /**
     * Returns the UnityPlayer instance.
     *
     * @return The UnityPlayer instance.
     */
    fun getUnityPlayer(): UnityPlayerForActivityOrService = unityPlayer
}
```

### Receving messages from Unity

Here's we have to collaborate with a Unity developer. The key concept - **static methods**. Unity have to call our static method for sending some string to a native part, such as `your.package.name.UnityBridge.reciveMessageFromUnity(message)`. The same way we have to call static method from Unity code. This process is pretty old, so, you can find more detail in the Internet.

```kotlin
package your.package.name //package name is important for Unity to find this file

// imports

// have to be defined by name in Unity script
internal object UnityBridge {

    private var msgSender: ((String) -> Unit)? = null

    @OptIn(DelicateCoroutinesApi::class)
    val unityEventFlow = callbackFlow {
        msgSender = { message ->
            val event = try {
                parseUnityEvent(message)
            } catch (e: Exception) {
                null
            }
            event?.let { trySend(it) }
        }

        awaitClose {
            msgSender = null
        }
    }.shareIn(
        scope = GlobalScope,
        started = SharingStarted.WhileSubscribed(),
        replay = 0
    )

    // have to be defined by name in Unity script
    @Suppress("UNUSED")
    @JvmStatic
    fun receiveMessageFromUnity(message: String) {
        msgSender?.invoke(message)
    }

    fun sendMessageToUnity(message: String) {
        /**
         * p0 - name of object in Unity, have to be attached to scene in Unity
         * p1 - name of method
         * p2 - string
         **/
        UnityPlayer.UnitySendMessage(<p0>, <p1>, message)
    }

    private fun parseUnityEvent(message: String): UnityEvent? {
        // your parse logic
    }
}

sealed interface UnityEvent {
    // your events
}

```

*Yeah, yeah... I used `GlobalScope`... and so what? In this case it is completely justified*

### Composable Unity function

```kotlin
/**
 * Composable function that displays a Unity scene using [AndroidView].
 * It handles the lifecycle of the Unity player and allows overlaying custom content.
 *
 * @param unityPlayer The [UnityPlayerForActivityOrService] instance to display.
 * @param clickOnBack A lambda function to be invoked when the back button is pressed.
 * Defaults to an empty lambda.
 * @param topContent A composable lambda that allows adding custom content on top of the Unity scene.
 */
@Composable
internal fun UnityScene(
    unityPlayer: UnityPlayerForActivityOrService,
    clickOnBack: () -> Unit = {},
    topContent: @Composable BoxScope.() -> Unit = {},
) {
    BackHandler { clickOnBack() }
    Box(modifier = Modifier.fillMaxSize()) {
        AndroidView(
            modifier = Modifier,
            factory = { context ->
                unityPlayer.onResume()
                unityPlayer.windowFocusChanged(true)
                unityPlayer.frameLayout
            },
            update = { frameLayout ->
                unityPlayer.windowFocusChanged(true)
            },
            onRelease = { frameLayout ->
                unityPlayer.windowFocusChanged(false)
                unityPlayer.onPause()
//              unityPlayer.onStop()
//              Unity don't stop itself, it's kill process of application
            }
        )

        topContent()
    }
}
```

### Screen

Here's example of a screen with pasting `UnityScene`. We have the opportunity to put content on the top but there will some problems with focuses of keyboard *if you put TextField, for example*. It solves on the Unity side.

```kotlin
@Composable
internal fun SomeScreen(
    unityPlayer: UnityPlayerForActivityOrService,
    finishActivity: () -> Unit,
) {
    var showUnity by remember { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        UnityBridge.unityEventFlow.collect { event ->
            when (event) {
                // flow with events from Unity
            }
        }
    }

    BackHandler { finishActivity() }

    if (showUnity) {
        UnityScene(
            unityPlayer = unityPlayer,
            clickOnBack = { showUnity = false }
        ) {
            // top composable content
        }
    } else {
        Box(
            modifier = Modifier.fillMaxSize(),
            contentAlignment = Alignment.Center
        ) {

            Button(onClick = { showUnity = true }) {
                Text("Start Unity")
            }

        }
    }
}
```

### Activity

Also you may have problems with Hardware Acceleration, you can set it manually if need `android:hardwareAccelerated="true"` 

Don't forget setup a another process to Activity if you wanna upload engine out of memory. **This isolates `SIGKILL` from your main process**

```kotlin
// some code
<application>
    <activity
        // some params
        android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale|layoutDirection|density"
        android:process=":unity_process"
        // some params
// some code
```

Okay, here's Activity.

```kotlin
class UnityActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        UnityPlayerManager.attachActivity(this)
        val unityPlayer = UnityPlayerManager.getUnityPlayer()

        enableEdgeToEdge()
        setContent {
            SomeScreen(unityPlayer, ::finish)
        }
    }
}
```

## Summary

1. We have an opportunity to set UI elements right on the top of Unity scene.
2. We can set pause to engine and hide the scene as simple composable function. Yeah, the engine will be on memory but we have an opportunity go back to scene fast like "hot start".
3. The Unity can be destroyed if we need. It will kill our process but not the whole application *(if you setup manifest of course)*. 

**Note:** I removed parts of code snippets under the NDA, so, the communication between processes via IPC by Intents, Services etc.

---

*P.S. I spent about 8 hours with a Unity developer with non-stop coding for create this solution. I hope it will help you if you have to struggle with the same problem. Kind regards!*
