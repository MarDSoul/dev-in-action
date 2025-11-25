# Sharing data between ViewModels in Compose
**Date of publication:** 28 Jan 2025

This article was inspired by [this post and comments below](https://www.linkedin.com/posts/jihad-mahfouz-017615209_android-androiddevelopment-androiddev-activity-7286332516918054912-gprl?utm_source=share&utm_medium=member_desktop&lipi=urn%3Ali%3Apage%3Ad_flagship3_pulse_read%3BMP%2FP%2FOafQ3iodAg3iWmI4Q%3D%3D). I wanna write about some common patterns about sharing data. 

_for code samples I'm using Hilt and Navigation frameworks_

---
## Use shared view model
**Description:** we use a single ViewModel with several screens.

It looks something tike this:

_SharedViewModel_

```kotlin
@HiltViewModel
class SharedViewModel : ViewModel() {
    private val _someString = MutableStateFlow("")
    val someString = _someString.asStateFlow()
}
```

_Screen1_
```kotlin
@Composable
fun Screen1(
    viewModel: SharedViewModel = hiltViewModel()
) {
    val someString by viewModel.someString.collectAsState()
}
```

_Screen2_

```kotlin
@Composable
fun Screen1(
    viewModel: SharedViewModel = hiltViewModel()
) {
    val someString by viewModel.someString.collectAsState()
}
```
So, what can I say? I just repeat my own words from comments of started post: 

> It's not the best approach. This way can be applied only for a small apps. When the app is becoming large, your viewmodel is becoming bigger and bigger. And one moment you can realize that the viewmodel is your data source in fact.

**One thing I'd like to add:** but generally there are a lot of situations when should use SharedViewModel. For instance, you have two screens that work with the same data (1st - for read, 2nd - for edit), it makes sense to use SharedViewModel in this case.

---
## Navigate with arguments
**Description:** we can pass a simple data like an argument in navigation route.

It looks something like this:

_NavGraph_

```kotlin
object NavDestinations {
    const val Screen1 = "Screen1"
    const val Screen2 = "Screen2"
}

@Composable
fun NavGraph(viewModel: MainViewModel, onLocaleChange: () -> Unit) {
    val navController = rememberNavController()
    NavHost(navController, NavDestinations.Screen1) {
        composable(NavDestinations.Screen1) {
            Screen1(navController)
        }
        composable("${NavDestinations.Screen2}/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")!!
            Screen2(navController)
        }
```

_Screen2ViewModel_
```kotlin
@HiltViewModel
class Screen2ViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getDataUseCase: GetDataUseCase
) : ViewModel() {
    private val id = savedStateHandle.get<Int>("id") ?: 0
    val data = getDataUseCase.getDataById(id)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = Data()
        )
}
```

_Screen1_

```kotlin
@Composable
fun Screen1(
    navController: NavController,
    viewModel: Screen1ViewModel = hiltViewModel()
) {
// some code
    Button(
        onClick = {
            navController.navigate(NavDestinations.Screen2 + "/${data.id}")
        }
    ) { Text("Click me") }
}
```

_Screen2_
```kotlin
@Composable
fun Screen2(
    navController: NavController,
    dataViewModel: Screen2ViewModel = hiltViewModel()
) {
    val data by dataViewModel.data.collectAsState()
// some code
}
```
More details you may see in [this article](https://sagar0-0.medium.com/pass-arguments-to-destinations-jetpack-compose-navigation-7c29d4c73ce5), for example.

It's an old approach and not actual now because we've got a [stable release](https://developer.android.com/jetpack/androidx/releases/navigation#2.8.5) of Navigation library in Dec 2024.

---

## Navigate with data
Ok, next we can send an object. [link to docs](https://developer.android.com/develop/ui/compose/navigation#nav-with-args)

I don't know should I copy code from Google docs or not. Okay, make a copy for collecting in one place.

_Serializable object_
```kotlin
@Serializable
data class Profile(val name: String)
```

_NavDestination_
```kotlin
// Pass only the user ID when navigating to a new destination as argument
navController.navigate(Profile(id = "user1234"))
```

_ViewModel_

```kotlin
class UserViewModel(
    savedStateHandle: SavedStateHandle,
    private val userInfoRepository: UserInfoRepository
) : ViewModel() {

    private val profile = savedStateHandle.toRoute<Profile>()

    // Fetch the relevant user information from the data layer,
    // ie. userInfoRepository, based on the passed userId argument
    private val userInfo: Flow<UserInfo> = userInfoRepository.getUserInfo(profile.id)

// …

}
```

So, you can send "primitive" types, string, recourse reference, parcelable, serializable end enum types. [link to docs](https://developer.android.com/guide/navigation/use-graph/pass-data#supported_argument_types)

**One thing I'd like to add:** I don't know how it works right now, but before the stable release it was a real challenge to send complex serializable object. It wasn't worth it.

**My advice:** if you need to send something, send a small piece of data such as "id" and get information from a normal data-source.

---

## Use repository in data layer

**The main concept:** Make a repository, inside of repository make a StateFlow with value. So, ViewModels subscribe to flow. Add a function with emit value in the repo. As a result we have a single data source and necessary amount ViewModels that don't depend on each other.

Quote by [Jihad Mahfouz](https://www.linkedin.com/in/jihad-mahfouz-017615209/?lipi=urn%3Ali%3Apage%3Ad_flagship3_pulse_read%3BMP%2FP%2FOafQ3iodAg3iWmI4Q%3D%3D)

> In large apps, data shouldn't be kept in memory. For that neither viewModels nor repositories should be used, we can pass a sort of reference like an ID as an argument to the next screen, and then look up the data for it

And you're absolutely right! So special for this approach I post an example from a real project (without NDA).

Okay, we have a large data what is keep in the database. Also we have two screens, each of them has its own ViewModel, obviously. On the 1st screen we have to show some kind of data from database, on the 2nd screen we choose necessary filters (lots of them) that we need to apply to data request.

It was something like this:

_FiltersRepositoryImpl_

```kotlin
class FiltersRepositoryImpl @Inject constructor(
    @DefaultDispatcher
    private val defaultDispatcher: CoroutineDispatcher
) : FiltersRepository {

    private val _filterStateFlow = MutableStateFlow(Filters(cond1, cond2))

    override fun getFilterStateFlow(): StateFlow<Filters> {
        return _filterStateFlow.asStateFlow()
    }

    override suspend fun updateFilter(filters: Filters) {
        withContext(defaultDispatcher) {
            _filterStateFlow.emit(filters)
        }
    }
}
```

_GetDataUseCaseImpl_

```kotlin
class GetDataUseCaseImpl @Inject constructor(
    private val dataRepository: DataRepository,
    @IoDispatcher
    private val ioDispatcher: CoroutineDispatcher,
// etc.
) : GetDataUseCase {

    override fun getDataListFlow(filters: Filters): Flow<List<Data>> {
        return dataRepository.getFilteredDataFlow(filters).map { dataDbViewList ->
            dataDbViewList.map { dataDbView -> mapToModel(dataDbView) }
        }.flowOn(ioDispatcher)
    }
// some code, mapper etc.
}
```

_FiltersScreenViewModel_

```kotlin
@HiltViewModel
class FiltersScreenViewModel @Inject constructor(
    private val filtersRepository: FiltersRepository
) : ViewModel() {
    var filters: Filters by mutableStateOf(Filters(cond1, cond2))

    init {
        viewModelScope.launch {
            filtersRepository.getFilterStateFlow().collectLatest {
                filters = it
            }
        }
    }

    fun applyFilters() {
        viewModelScope.launch {
            filtersRepository.updateFilter(filters.copy(someCond))
        }
    }
// some code
}
```

_DataScreenViewModel_

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
@HiltViewModel
class DataScreenViewModel @Inject constructor(
    private val getDataUseCase: GetDataUseCase,
    private val filtersRepository: FiltersRepository
) : ViewModel() {

    val dataListFlow: StateFlow<List<Data>> =
        filtersRepository.getFilterStateFlow().flatMapLatest { 
            getDataUseCase.getDataListFlow(it)
        }.stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )
// some code
}
```

As you see, sometimes using a repository for data exchange is justified.

---

So, that's all for now. Thanks for your attention :)
