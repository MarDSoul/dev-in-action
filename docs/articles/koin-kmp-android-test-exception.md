# Fixing RuntimeException caused by InitializationProvider in KMP/Koin integration tests
**Date of publication:** 24 Jun 2026

## Problem

### Case description

Repository with reproducing this failure by the [link](https://github.com/MarDSoul/example-reproduce-koin-test-behavior)

- Project is on KMP/CMP framework
- Integration tests are for Android build
- In the project and tests there is a Koin DI framework
- Versions:
    - kotlin 2.3.21
    - koin 4.2.1

An exception occurs during starting tests with using `koin-test` pack of libraries:
```text
java.lang.RuntimeException: Unable to get provider androidx.startup.InitializationProvider: java.lang.ClassNotFoundException: Didn't find class "androidx.startup.InitializationProvider" ... //etc
```

### Unexpected behavior

Regardless of catching exception the tests can be passed. Check the file: [Test_Results_1.html](https://raw.githack.com/MarDSoul/example-reproduce-koin-test-behavior/461c0fb607ea3d065f182ef6a06776dbdc2b6291/docs/Test_Results_1.html). Pay attention to the time *1m 42s* of tests. The tests were passed by timeout.

Of course we've got a stacktrace in *Android Studio* like this:
```text
Dev_Stable_35(AVD) - 15 Tests 0/2 completed. (0 skipped) (0 failed)
Dev_Stable_35(AVD) - 15 Tests 1/2 completed. (0 skipped) (0 failed)
Finished 2 tests on Dev_Stable_35(AVD) - 15

> Task :androidApp:connectedDebugAndroidTest
Logcat of last crash: 
Process: com.example.test, PID: 7041
java.lang.RuntimeException: Unable to get provider androidx.startup.InitializationProvider: java.lang.ClassNotFoundException: Didn't find class "androidx.startup.InitializationProvider" on path: DexPathList ... //etc

//stacktrace

Caused by: java.lang.ClassNotFoundException: Didn't find class "androidx.startup.InitializationProvider" on path: DexPathList ... //etc

//stacktrace

BUILD SUCCESSFUL in 2m 16s
```
But if you use some pipeline you may not see that exception and it will look like everything is OK. Commit with this behavior by the [link](https://github.com/MarDSoul/example-reproduce-koin-test-behavior/tree/ecd1ffda973d1d676965912d69242092878c7636)

*Hmm... I can't imagine why it happenes but it's really not Okay!*

## Localization problem

- Find `InitializationProvider` and what tries calling it.

Check *manifest* by the path `build/intermediates/packaged_manifests/debugAndroidTest/processDebugAndroidTestManifest/AndroidManifest.xml`:
```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="com.example.test.androidx-startup"
    android:exported="false" >
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup" />
</provider>
```

The `WorkManager` is found. But there is no `WorkManager` in the project. So I suppose that something applies it for test build.

- Find *Gradle* dependencies with `WorkManager`

```bash
./gradlew androidApp:dependencies > docs/androidApp_dep.log
cat docs/androidApp_dep.log | grep -B 5 -A 5 'workmanager'
```

```text
|    +--- io.insert-koin:koin-test-junit4:4.2.1 (c)
|    +--- io.insert-koin:koin-android-test:4.2.1 (c)
|    +--- io.insert-koin:koin-compose:4.2.1 (c)
|    +--- io.insert-koin:koin-compose-viewmodel:4.2.1 (c)
|    +--- io.insert-koin:koin-android:4.2.1 (c)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1 (c)
|    +--- io.insert-koin:koin-core:4.2.1 (c)
|    +--- io.insert-koin:koin-core-annotations:4.2.1 (c)
|    +--- io.insert-koin:koin-core-viewmodel:4.2.1 (c)
|    \--- io.insert-koin:koin-core-jvm:4.2.1 (c)
+--- io.insert-koin:koin-test -> 4.2.1
--
|    |    |    +--- androidx.savedstate:savedstate-ktx:1.2.1 -> 1.4.0 (*)
|    |    |    +--- org.jetbrains.kotlin:kotlin-stdlib:1.8.22 -> 2.3.21 (*)
|    |    |    \--- androidx.fragment:fragment:1.8.9 (c)
|    |    +--- androidx.lifecycle:lifecycle-viewmodel-ktx:2.10.0 (*)
|    |    \--- org.jetbrains.kotlin:kotlin-stdlib:2.3.20 -> 2.3.21 (*)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1
|    |    +--- io.insert-koin:koin-android:4.2.1 (*)
|    |    +--- androidx.work:work-runtime-ktx:2.11.1
|    |    |    +--- androidx.work:work-runtime:2.11.1
|    |    |    |    +--- androidx.annotation:annotation-experimental:1.4.1 (*)
|    |    |    |    +--- androidx.core:core:1.12.0 -> 1.18.0 (*)
--
|    +--- io.insert-koin:koin-compose:4.2.1 (c)
|    +--- io.insert-koin:koin-compose-viewmodel:4.2.1 (c)
|    +--- io.insert-koin:koin-core:4.2.1 (c)
|    +--- io.insert-koin:koin-core-coroutines:4.2.1 (c)
|    +--- io.insert-koin:koin-android:4.2.1 (c)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1 (c)
|    +--- io.insert-koin:koin-core-jvm:4.2.1 (c)
|    +--- io.insert-koin:koin-core-annotations:4.2.1 (c)
|    \--- io.insert-koin:koin-core-viewmodel:4.2.1 (c)
+--- io.insert-koin:koin-test -> 4.2.1
|    \--- io.insert-koin:koin-test-jvm:4.2.1
--
|    |    |    +--- androidx.savedstate:savedstate-ktx:1.2.1 -> 1.4.0 (*)
|    |    |    +--- org.jetbrains.kotlin:kotlin-stdlib:1.8.22 -> 2.3.21 (*)
|    |    |    \--- androidx.fragment:fragment:1.8.9 (c)
|    |    +--- androidx.lifecycle:lifecycle-viewmodel-ktx:2.10.0 (*)
|    |    \--- org.jetbrains.kotlin:kotlin-stdlib:2.3.20 -> 2.3.21 (*)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1
|    |    +--- io.insert-koin:koin-android:4.2.1 (*)
|    |    +--- androidx.work:work-runtime-ktx:2.11.1
|    |    |    +--- androidx.work:work-runtime:2.11.1
|    |    |    |    +--- androidx.annotation:annotation-experimental:1.4.1 (*)
|    |    |    |    +--- androidx.concurrent:concurrent-futures-ktx:1.1.0
--
|    +--- io.insert-koin:koin-test-junit4:4.2.1 (c)
|    +--- io.insert-koin:koin-android-test:4.2.1 (c)
|    +--- io.insert-koin:koin-compose:4.2.1 (c)
|    +--- io.insert-koin:koin-compose-viewmodel:4.2.1 (c)
|    +--- io.insert-koin:koin-android:4.2.1 (c)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1 (c)
|    +--- io.insert-koin:koin-core:4.2.1 (c)
|    +--- io.insert-koin:koin-core-viewmodel:4.2.1 (c)
|    +--- io.insert-koin:koin-core-annotations:4.2.1 (c)
|    \--- io.insert-koin:koin-core-jvm:4.2.1 (c)
+--- io.insert-koin:koin-test -> 4.2.1
--
|    |    |    +--- androidx.savedstate:savedstate-ktx:1.2.1 -> 1.4.0 (*)
|    |    |    +--- org.jetbrains.kotlin:kotlin-stdlib:1.8.22 -> 2.3.21 (*)
|    |    |    \--- androidx.fragment:fragment:1.8.9 (c)
|    |    +--- androidx.lifecycle:lifecycle-viewmodel-ktx:2.10.0 (*)
|    |    \--- org.jetbrains.kotlin:kotlin-stdlib:2.3.20 -> 2.3.21 (*)
|    +--- io.insert-koin:koin-androidx-workmanager:4.2.1
|    |    +--- io.insert-koin:koin-android:4.2.1 (*)
|    |    +--- androidx.work:work-runtime-ktx:2.11.1
|    |    |    +--- androidx.work:work-runtime:2.11.1
|    |    |    |    +--- androidx.annotation:annotation-experimental:1.4.1 (*)
|    |    |    |    +--- androidx.concurrent:concurrent-futures-ktx:1.1.0
```

Okay, discovered that the *Koin* in the cause.

## Solution

Obviously, it needs to remove `koin-androidx-workmanager` from dependencies:
```kotlin
androidTestImplementation(dependencies.platform(libs.koin.bom))
androidTestImplementation(libs.bundles.koin.test.android) {
    exclude(group = "io.insert-koin", module = "koin-androidx-workmanager")
}
```

---

## Summary

As you can see, you can be blindsided by transitive dependencies. What was the barrier for Koin developers to provide a core library without unnecessary features? I don't know. Anyway, `koin-test` and `koin-test-android` include `koin-androidx-workmanager` by default, which pulls in `WorkManager`. That requires `InitializationProvider`, which doesn't exist in the project, leading to a crash. The straightforward solution is to simply exclude that dependency.

*Okay, thanks for reading! Implicit content is evil...*
