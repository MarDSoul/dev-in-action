# Using Hilt (guide for beginners)
**Date of publication:** 07 Jan 2026

## Intro

This article was born as an answer for question I got:

> I've seen dependency injection in different ways: Injecting in constructor, initializing vía lateinit or, in the di files, sometimes are written with @Provides and others with @Binds... 
>
> So my question is where to find a well defined standard or guidelines where explain all of this, as the documentation isn't very clear, at least from my perspective 

So, I decided to write a guide of using Hilt. It's not about "How to add Hilt in your project?" or "Setup `@HiltAndroidApp` or something like that. I think you can read about this by yourself ([link](#docs)). I tried to concentrate on practice.

*Okay, let's go*

## Docs

Links:

- [Tutorial](https://dagger.dev/hilt/)
- [Releases](https://developer.android.com/jetpack/androidx/releases/hilt)
- [Docs from Google](https://developer.android.com/training/dependency-injection/hilt-android)

## Practice in code

### `@Provides` and `@Binds`

**Where using `@Provides` and `@Binds`?**

#### Short answer

`@Binds` is for using for interface implementation. For example:

```kotlin
interface SomeRepo {
    fun someFun()
}

//you can define the scope of class right there, but it's not useful
class SomeRepoImpl @Inject constructor(
    private val someDep: SomeDep
    //other deps
) : SomeRepo {
    override fun someFun() {
        //fun's body
    }
}

@Module
@InstallIn(SingletonComponent::class) //module's scope
abstract class RepoBindModule { //must be interface or abstract class
    @Binds
    @Singleton //class scope
    abstract fun bindSomeRepo(impl: SomeRepoImpl): SomeRepo
}
```

`@Provides` is for using when you have custom initialization logic. For example:

```kotlin
@Module
@InstallIn(SingletonComponent::class) //module's scope
object SomeProvideModule { //must be class, not abstract
    @Provides
    @Singleton //class scope
    fun provideSomeClass(someDep: SomeDep) = SomeClass.getInstance(someDep)
}
```

#### Long answer

The [Binds](https://dagger.dev/api/latest/dagger/Binds.html) annotation is abstract and doesn't create factory. It's just for saying **Dagger** "here's you can define the type of object you have to return". More info by [link](https://dagger.dev/tutorial/04-depending-on-interface).

The [Provides](https://dagger.dev/api/latest/dagger/Provides.html) annotation means that the object is creating inside the module. More info by [link](https://dagger.dev/tutorial/05-abstraction-for-output).

Pay attention that the **module** with `@Provides` have to be `object`(more effective) or `class` against with `@Binds` which have to be *abstract*.

### Ways of injecting

- by constructor

It looks like this:

```kotlin
class SomeClass @Inject constructor(
    private val someDep: SomeDep //of course you need to have this type in dependency graph
) {
    //code
}
```

- like variable (have to be public) in class by `@Inject`

It looks like this:

```kotlin
class SomeClass() {
    @Inject
    lateinit var someDep: SomeDep

    //code
}
```

The most common way is inject **by constructor** for better maintainability and testing (you have an opportunity to make mock entity special for testing). Another way of injecting **like a variable** is used for classes where you don't access to constructor (such as `Activity`, `Service` etc.)

### Using scopes

The documentation by the [link](https://developer.android.com/training/dependency-injection/hilt-android#generated-components).

Important notes:

- `@InstallIn` - is about components

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
abstract class SomeModule {
    @Binds
    abstract fun bindSomeDep(impl: SomeDepImpl): SomeDep
}
```

You **can't** inject that dependency into `Activity`

- `@Singleton`, `@ViewModelScoped` etc. - is about instances of classes

**Without scopes:**

```kotlin
@HiltViewModel
class SomeViewModel @Inject constructor(
    private val dep1: SomeDep1,
    private val dep2: SomeDep2
) : ViewModel() {

    //some code

}

class SomeDep1 @Inject constructor(
    private val commonDep: CommonDep
) {
    //some code
}

class SomeDep2 @Inject constructor(
    private val commonDep: CommonDep
) {
    //some code
}

class CommonDep @Inject constructor() {
    //some code
}
```

In the example above the instances `dep1` and `dep2` will get their own instances of `commonDep`, each of them will keep a new instance, not the same.

**With scopes:**

```kotlin
@ViewModelScoped
class SomeDep1 @Inject constructor(
    private val commonDep: CommonDep
) {
    //some code
}

@ViewModelScoped
class SomeDep2 @Inject constructor(
    private val commonDep: CommonDep
) {
    //some code
}

@ViewModelScoped
class CommonDep @Inject constructor() {
    //some code
}
```

In the example above the instance of `commonDep` will be the same.

**Note:** the `@ViewModelScoped` guarantee the single instances for the **one** `ViewModel`. In another `ViewModel` will be another instances.

### Qualifiers

Used when you need create the different instances of the same type. Documents by [link](https://developer.android.com/training/dependency-injection/hilt-android#multiple-bindings), example [below](#bonus).

## Practice in testing

The documentation by the [link](https://developer.android.com/training/dependency-injection/hilt-testing). I don't wanna copy official docs, so, I'll show power of Hilt for testing. *That's why I really like it!*

Imagine, you have some data source, `Proto DataStore` for example:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    private const val DATASTORE_FILE = "filename.pb"

    @Singleton
    @Provides
    fun provideDataStore(
        @ApplicationContext context: Context,
        @IoDispatcher ioDispatcher: CoroutineDispatcher,
    ): DataStore<YourProtoClass> {
        return DataStoreFactory.create(
            serializer = YourProtoClassSerializer,
            corruptionHandler = ReplaceFileCorruptionHandler { YourProtoClass.getDefaultInstance() },
            scope = CoroutineScope(ioDispatcher + SupervisorJob()),
            produceFile = { context.dataStoreFile(DATASTORE_FILE ) },
        )
    }
}
```

And you have a long-long delivery way that source to `ViewModel`:

```kotlin
//repo

interface SomeRepo {
    suspend fun action1()
    suspend fun action2()
}

class SomeRepoImpl @Inject constructor(
    private val dataStore: DataStore<YourProtoClass>
) : SomeRepo {
    override suspend fun action1() = dataStore.update { /*data 1*/ }
    override suspend fun action2() = dataStore.update { /*data 2*/ }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepoModule {
    @Binds
    @Singleton
    abstract fun bindSomeRepo(impl: SomeRepoImpl): SomeRepo
}

//usecases

interface UseCase1 {
    suspend fun use()
}

class UseCase1Impl @Inject constructor(
    private val someRepo: SomeRepo
) : UseCase1 {
    override suspend fun use() = someRepo.action1()
}

interface UseCase2 {
    suspend fun use()
}

class UseCase2Impl @Inject constructor(
    private val someRepo: SomeRepo
) : UseCase2 {
    override suspend fun use() = someRepo.action2()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class UseCaseModule {

    @Binds
    abstract fun bindUseCase1(impl: UseCase1Impl): UseCase1

    @Binds
    abstract fun bindUseCase2(impl: UseCase2Impl): UseCase2
}

//viewmodel

@HiltViewModel
class SomeViewModel @Inject constructor(
    private val useCase1: UseCase1,
    private val useCase2: UseCase2
) : ViewModel() {

    //some state

    fun do1() {
        viewModelScope.launch {
            useCase1.use()
        }
    }

    fun do2() {
        viewModelScope.launch {
            useCase2.use()
        }
    }

    //some code
}
```

And we wanna test UI with the `ViewModel` (integration tests). So, the simple way:

- change real data source to test source

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [DataStoreModule::class],
)
object TestDataStoreModule {
    private const val TEST_FILE_NAME = "test_filename"
    private const val PREFIX = ".pb"

    @Provides
    @Singleton
    fun provideTestDataStore(
        testDispatcher: TestDispatcher,
    ): DataStore<YourProtoClass> {
        val tempFile = File.createTempFile(TEST_FILE_NAME, PREFIX)
        return DataStoreFactory.create(
            serializer = YourProtoClassSerializer,
            scope = CoroutineScope(testDispatcher + SupervisorJob()),
            produceFile = {
                tempFile.apply { deleteOnExit() }
            },
        )
    }
}
```

- write tests

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class ViewModelTest {
    @Inject
    lateinit var testDispatcher: TestDispatcher

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<TestActivity>()

    protected lateinit var viewModel: SomeViewModel

    @Before
    fun setUp() {
        hiltRule.inject()
        Dispatchers.setMain(testDispatcher)
        composeTestRule.setContent {
            viewModel = hiltViewModel()
        }
        composeTestRule.waitForIdle()
    }

    @After
    fun tearDownBase() {
        Dispatchers.resetMain()
    }

    //some tests
    
}
```

And that's all! Comprehended? You don't need provide manually all those entities, writing mocks for each object and so on. In tests you use your real classes and real objects, just with the fake data source.

**Note:** I omitted info about `@AndroidEntryPoint` on TestActivity, `HiltTestRunner` and so on... look for it in docs, Google, LLM etc.

*`TestInstallIn` is an amazing feature! Like it!*

## Uncommon cases

### "Single-job" instances

Imagine, you have to inject some instance of class for only one job. Usually it makes by [delegates](https://kotlinlang.org/docs/delegation.html) interfaces. But what is that action needs on other data. 

Example:

```kotlin
class SomeSingleJobClass @Inject constructor(
    private val source1: Source1,
    private val source2: Source2,
    //etc
    private val executorClass: ExecutorClass
) {
    fun doWork(data: Data) {
        val sourceData1 = source1.get()
        val sourceData2 = source2.get()
        val collectedData = CollectedData(data, sourceData1, sourceData2)
        executorClass.action(collectedData) 
    }
}
```

And here's our target class:

```kotlin
class SomeClass @Inject constructor(
    private val someDep: SomeDep,
    //etc
    private val singleJobProvider: Provider<SomeSingleJobClass>
) {
    //some code
    fun doSingleJob() {
        singleJobProvider.get().doWork(data)
    }
}
```

`Provider<T>` - get a new instance every time you call `get()`.

### Not-supported

I just wanna mention that some Android classes don't support Hilt "out-of-the-box", more details by [link](https://developer.android.com/training/dependency-injection/hilt-android#not-supported).

### `@AssistedInject`

So, this annotation is used when you need injecting some value in runtime.

```kotlin
class SomeDetailService @AssistedInject constructor(
    private val repo: SomeRepo, 
    @Assisted private val detailId: String 
) {
    fun getDetail() = repo.fetch(detailId)

    @AssistedFactory
    interface Factory {
        fun create(detailId: String): SomeDetailService
    }
}
```

**History:** a long-long time ago (before Hilt 2.31) the delivering some *id* into `ViewModel` was not so simple like now with `hiltViewModel()`

## Bonus

Usually I use these templates for injecting dispatchers and scope in some classes where I need in.

```kotlin
@Suppress("UNUSED")
@Module
@InstallIn(SingletonComponent::class)
object CoroutinesDispatchersModule {

    @DefaultDispatcher
    @Provides
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @IoDispatcher
    @Provides
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @MainDispatcher
    @Provides
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}

@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class DefaultDispatcher

@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class IoDispatcher

@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class MainDispatcher
```

```kotlin
@Suppress("UNUSED")
@Module
@InstallIn(SingletonComponent::class)
object ScopeModule {

    @ApplicationScope
    @Provides
    @Singleton
    fun provideApplicationScope(@DefaultDispatcher defaultDispatcher: CoroutineDispatcher): CoroutineScope {
        return CoroutineScope(SupervisorJob() + defaultDispatcher)
    }
}

@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class ApplicationScope
```

---

*Thanks for reading! I hope it was useful for you!*
