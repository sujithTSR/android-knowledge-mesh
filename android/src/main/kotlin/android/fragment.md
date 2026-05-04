# Fragment

## Fragment lifecycle methods:

| Method            | Called When…                                                        | Use Case                                                                   |
|-------------------|---------------------------------------------------------------------|----------------------------------------------------------------------------|
| `onAttach()`      | Fragment is associated with its host activity                       | Use this to get `Context` or communicate with the activity (via interface) | 
| `onCreate()`      | Fragment is being created (non-UI logic)                            | Initialize non-UI resources (e.g., ViewModels, arguments)                  | 
| `onCreateView()`  | UI layout is being created                                          | Inflate layout and initialize root views                                   |
| `onViewCreated()` | View is created, fully accessible                                   | Setup UI logic, bind views, observe LiveData, etc.                         | 
| `onStart()`       | Fragment is visible (UI is on screen)                               | Start animations, audio, or visible-related tasks                          |
| `onResume()`      | Fragment is fully interactive and in foreground                     | Start live actions (camera, sensors, gestures, listeners)                  | 
| `onPause()`       | Fragment is partially hidden or another fragment/activity is on top | Pause ongoing tasks, save UI state                                         | 
| `onStop()`        | Fragment is fully hidden                                            | Stop animations, unregister listeners                                      | 
| `onDestroyView()` | Fragment’s view hierarchy is removed                                | Clean up views, avoid memory leaks (set binding/view = null)               | 
| `onDestroy()`     | Fragment instance is about to be destroyed                          | Clean up remaining resources                                               | 
| `onDetach()`      | Fragment is disassociated from Activity                             | Final cleanup, reference to Activity should be cleared                     | 


## Activity And Fragment lifecycle calls as per launch

**When you run the app**
 
- MainActivity: onCreate
- SampleFragment: onAttach
- SampleFragment: onCreate
- SampleFragment: onCreateView
- SampleFragment: onViewCreated
- MainActivity: onStart
- SampleFragment: onStart
- MainActivity: onResume
- SampleFragment: onResume

**And during exit:**

- MainActivity: onPause
- SampleFragment: onPause
- MainActivity: onStop
- SampleFragment: onStop
- SampleFragment: onDestroyView
- SampleFragment: onDestroy
- SampleFragment: onDetach
- MainActivity: onDestroy

## Fragment lifecycle vs Fragment view lifecycle

A Fragment has two related but different lifecycles:

1. **Fragment lifecycle**
2. **Fragment view lifecycle**

This distinction is one of the most important Fragment interview topics.

### Why are there two lifecycles?

A Fragment object can stay alive even after its view is destroyed.

This usually happens when the Fragment is placed on the back stack.

Example:

```text
Fragment instance still exists
But Fragment view is destroyed
```

That means anything connected to the UI should be tied to the **view lifecycle**, not just the Fragment lifecycle.

### Fragment lifecycle

The Fragment lifecycle belongs to the Fragment object itself.

```kotlin
fragment.lifecycle
```

It starts when the Fragment is attached/created and ends when the Fragment is destroyed/detached.

Typical methods:

```text
onAttach()
onCreate()
onCreateView()
onViewCreated()
onStart()
onResume()
onPause()
onStop()
onDestroyView()
onDestroy()
onDetach()
```

### Fragment view lifecycle

The view lifecycle belongs only to the Fragment's UI/view hierarchy.

```kotlin
viewLifecycleOwner.lifecycle
```

It starts after `onCreateView()` and ends at `onDestroyView()`.

Important point:

```text
onDestroyView() destroys the Fragment's view,
but the Fragment instance may still remain alive.
```

### Why does this matter?

If you observe data using the Fragment lifecycle, your observer may continue even after the view is destroyed.

This can cause:

- Memory leaks
- Crashes
- Duplicate collectors
- Updating a destroyed view
- Invalid binding access

### Wrong approach

```kotlin
lifecycleScope.launch {
    viewModel.state.collect { state ->
        binding.title.text = state.title
    }
}
```

Problem:

```text
This is tied to the Fragment lifecycle, not the view lifecycle.
The Fragment may remain alive even when the view is destroyed.
```

### Better approach

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { state ->
            binding.title.text = state.title
        }
    }
}
```

This ensures collection happens only when the Fragment view is at least `STARTED`.

### View binding cleanup

A common Fragment pattern:

```kotlin
private var _binding: FragmentSampleBinding? = null
private val binding get() = _binding!!

@Override
fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View {
    _binding = FragmentSampleBinding.inflate(inflater, container, false)
    return binding.root
}

@Override
fun onDestroyView() {
    super.onDestroyView()
    _binding = null
}
```

The binding is cleared in `onDestroyView()` because the view is destroyed there.

### Interview answer

> A Fragment has its own lifecycle, but its view has a separate lifecycle. The Fragment can outlive its view, especially when it is on the back stack. So UI-related work like binding access, Flow collection, LiveData observation, adapters, and listeners should be tied to `viewLifecycleOwner`, not just the Fragment lifecycle.

---

## `onCreateView()` vs `onViewCreated()`

Both methods are related to Fragment UI creation, but they are used for different purposes.

### `onCreateView()`

Used to inflate and return the Fragment's root view.

```kotlin
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View {
    _binding = FragmentSampleBinding.inflate(inflater, container, false)
    return binding.root
}
```

Keep this method lightweight.

Use it mainly for:

- Inflating layout
- Creating binding
- Returning the root view

### `onViewCreated()`

Called after the view has been created.

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)

    setupRecyclerView()
    setupClickListeners()
    observeViewModel()
}
```

Use it for:

- Setting click listeners
- Observing ViewModel state
- Setting up RecyclerView
- Binding UI data
- Navigation click handling

### Interview answer

> I keep `onCreateView()` mostly for inflating and returning the view. I use `onViewCreated()` for UI setup because the view hierarchy is fully created and safely accessible there.

---

## FragmentManager and FragmentTransaction

`FragmentManager` manages Fragment operations.

`FragmentTransaction` defines what change should happen.

Example:

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.container, HomeFragment())
    .addToBackStack(null)
    .commit()
```

### Common FragmentTransaction operations

| Operation | Meaning |
|---|---|
| `add()` | Adds a Fragment without removing the existing one |
| `replace()` | Removes the current Fragment and adds a new one |
| `remove()` | Removes a Fragment |
| `hide()` | Hides a Fragment but keeps it added |
| `show()` | Shows a hidden Fragment |
| `detach()` | Destroys the Fragment view but keeps Fragment state |
| `attach()` | Recreates the view for a detached Fragment |

### `add()` vs `replace()`

`add()` keeps the existing Fragment and adds another one.

`replace()` removes the existing Fragment and adds a new one.

Interview point:

```text
replace() is similar to remove() + add().
```

### `commit()` vs `commitNow()` vs `commitAllowingStateLoss()`

| Method | Meaning |
|---|---|
| `commit()` | Schedules the transaction asynchronously |
| `commitNow()` | Executes the transaction immediately |
| `commitAllowingStateLoss()` | Allows commit after state is saved, but transaction may be lost |

### Important note

`commitNow()` cannot be used with `addToBackStack()`.

### Interview answer

> I use `commit()` for normal Fragment transactions. I avoid `commitAllowingStateLoss()` unless the UI change is temporary and non-critical because it may be lost during Activity recreation.

---

## Fragment back stack

The back stack controls how Fragments are restored when the user presses back.

Example:

```text
A -> B -> C
```

If B and C are added to the back stack, pressing back from C returns to B.

### What happens when a Fragment goes to back stack?

Usually, the Fragment's view can be destroyed:

```text
onPause()
onStop()
onDestroyView()
```

But the Fragment instance may remain alive.

These may not be called immediately:

```text
onDestroy()
onDetach()
```

### Interview answer

> When a Fragment is added to the back stack, its view may be destroyed but the Fragment instance can remain. That is why view references and binding should be cleared in `onDestroyView()`.

---

## Fragment arguments and Safe Args

Avoid passing data through Fragment constructors.

### Wrong approach

```kotlin
class DetailFragment(private val id: String) : Fragment()
```

Problem:

```text
Android may recreate the Fragment using the default constructor.
Constructor arguments can be lost during recreation.
```

### Better approach using arguments

```kotlin
class DetailFragment : Fragment() {

    companion object {
        fun newInstance(id: String): DetailFragment {
            return DetailFragment().apply {
                arguments = bundleOf("id" to id)
            }
        }
    }
}
```

### Better approach using Safe Args

```kotlin
class DetailFragment : Fragment(R.layout.fragment_detail) {
    private val args: DetailFragmentArgs by navArgs()
}
```

### Interview answer

> Fragment data should be passed using arguments or Safe Args because the system can recreate Fragments during configuration changes or process death.

---

## Fragment Result API

Used for one-time communication between Fragments.

### Sender Fragment

```kotlin
setFragmentResult(
    "requestKey",
    bundleOf("selectedId" to selectedId)
)
```

### Receiver Fragment

```kotlin
setFragmentResultListener("requestKey") { _, bundle ->
    val selectedId = bundle.getString("selectedId")
}
```

### When to use

Use Fragment Result API when one Fragment needs to send a one-time result back to another Fragment.

Examples:

- Selecting an item
- Returning from a bottom sheet
- Confirming an action
- Picking a filter

### Interview answer

> I use Fragment Result API for one-time Fragment-to-Fragment communication. For shared screen state, I prefer a shared ViewModel scoped to the Activity, parent Fragment, or navigation graph.

---

## Fragment communication patterns

| Communication | Preferred approach |
|---|---|
| Fragment to Activity | Activity-scoped ViewModel or interface |
| Fragment to Fragment | Shared ViewModel, Fragment Result API, Navigation result |
| Parent Fragment to Child Fragment | Parent-scoped ViewModel or `childFragmentManager` |
| Child Fragment to Parent Fragment | Fragment Result API or parent-scoped ViewModel |
| Fragment to ViewModel | Direct ViewModel state/events |

### Interview answer

> For shared state, I use a shared ViewModel scoped to the right owner. For one-time results, I use Fragment Result API. I avoid tightly coupling Fragments directly to each other.

---

## `childFragmentManager` vs `parentFragmentManager`

### `childFragmentManager`

Used when a Fragment hosts another Fragment.

```kotlin
childFragmentManager.beginTransaction()
    .replace(R.id.childContainer, ChildFragment())
    .commit()
```

### `parentFragmentManager`

Used to interact with the FragmentManager that added the current Fragment.

```kotlin
parentFragmentManager.beginTransaction()
    .replace(R.id.container, AnotherFragment())
    .commit()
```

### Interview answer

> If a Fragment is hosting nested Fragments, I use `childFragmentManager`. If I need to interact with the manager that owns the current Fragment, I use `parentFragmentManager`.

---

## DialogFragment

`DialogFragment` is used for dialogs that should be lifecycle-aware and managed by FragmentManager.

```kotlin
class ConfirmDialogFragment : DialogFragment()
```

### Why use DialogFragment?

| Benefit | Why it matters |
|---|---|
| Lifecycle-aware | Handles configuration changes better than raw dialogs |
| FragmentManager-managed | Can participate in Fragment state handling |
| Reusable | Can be reused across screens |
| Back handling | Works better with Fragment navigation |

### Interview answer

> I prefer `DialogFragment` when the dialog is part of the app flow because it is lifecycle-aware and managed by FragmentManager.

---

## ViewPager2 with Fragments

`ViewPager2` uses `FragmentStateAdapter` for Fragment pages.

```kotlin
class SamplePagerAdapter(fragment: Fragment) : FragmentStateAdapter(fragment) {

    override fun getItemCount(): Int = 3

    override fun createFragment(position: Int): Fragment {
        return when (position) {
            0 -> FirstFragment()
            1 -> SecondFragment()
            else -> ThirdFragment()
        }
    }
}
```

Important property:

```kotlin
viewPager.offscreenPageLimit = 1
```

This controls how many pages are retained on either side of the current page.

### Interview answer

> ViewPager2 uses RecyclerView internally, and FragmentStateAdapter manages Fragment pages efficiently. I keep page state in ViewModels and avoid assuming all pages are always resumed.

---

## Memory leaks in Fragments

Most Fragment leaks happen because the Fragment outlives its view.

### Common leak sources

| Leak source | Fix |
|---|---|
| Binding not cleared | Clear binding in `onDestroyView()` |
| Observing with Fragment lifecycle | Use `viewLifecycleOwner` |
| Long-running collectors | Use `repeatOnLifecycle()` |
| Adapter holding view references | Clear adapter when needed |
| Listener/callback not removed | Remove in `onDestroyView()` |
| Holding Activity reference | Avoid storing Activity/context unnecessarily |

### Interview answer

> I prevent Fragment leaks by tying UI work to `viewLifecycleOwner`, clearing binding in `onDestroyView()`, removing listeners, and avoiding long-lived references to views or Activity context.

---

## Configuration changes

During configuration change, such as rotation, Activity and Fragment views are recreated.

Typical flow:

```text
onPause()
onStop()
onDestroyView()
onDestroy()
onDetach()
onAttach()
onCreate()
onCreateView()
onViewCreated()
onStart()
onResume()
```

But ViewModel survives configuration changes.

```kotlin
private val viewModel: SampleViewModel by viewModels()
```

### Interview answer

> Configuration changes recreate Activity and Fragment views, but ViewModel survives. I keep UI state in ViewModel and restore small view state using saved instance state or SavedStateHandle where needed.

---

## Process death and SavedStateHandle

ViewModel survives configuration changes, but it does not survive process death.

For process death recovery, use:

- Fragment arguments
- `SavedStateHandle`
- `onSaveInstanceState()`
- Room
- DataStore
- Backend reload

Example:

```kotlin
class DetailViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    val itemId: String = savedStateHandle["itemId"] ?: ""
}
```

### Interview answer

> ViewModel is enough for configuration changes, but not for process death. For process death, I rely on Fragment arguments, SavedStateHandle, persistent storage, or reloading from source of truth.

---

## Jetpack Navigation Component

Navigation Component helps manage Fragment navigation using a navigation graph.

Common APIs:

```kotlin
findNavController().navigate(R.id.action_home_to_detail)
findNavController().popBackStack()
```

### Important concepts

| Concept | Meaning |
|---|---|
| Navigation graph | Defines app navigation flow |
| Safe Args | Type-safe argument passing |
| Nested graphs | Useful for feature-level flows |
| Deep links | Opens a specific destination |
| NavGraph-scoped ViewModel | Shared ViewModel within a flow |
| Multiple back stacks | Useful for bottom navigation |

### Interview answer

> In larger apps, I prefer Navigation Component with feature-level graphs, Safe Args, and graph-scoped ViewModels instead of manually managing every FragmentTransaction.

---

## RecyclerView state restoration in Fragments

RecyclerView state can be restored too early before data is loaded.

Use:

```kotlin
adapter.stateRestorationPolicy =
    RecyclerView.Adapter.StateRestorationPolicy.PREVENT_WHEN_EMPTY
```

This prevents RecyclerView from restoring scroll position when the adapter is empty.

### Interview answer

> For RecyclerView inside Fragments, I use `PREVENT_WHEN_EMPTY` so scroll state is restored only after data is available.

---

## `setMaxLifecycle()`

`setMaxLifecycle()` is used to restrict the maximum lifecycle state a Fragment can reach.

Example:

```kotlin
transaction.setMaxLifecycle(fragment, Lifecycle.State.STARTED)
```

This means the Fragment can move up to `STARTED`, but it will not move to `RESUMED`.

So the Fragment can reach:

```text
onCreate()
onCreateView()
onViewCreated()
onStart()
```

But it will not reach:

```text
onResume()
```

### Why restrict a Fragment lifecycle?

Sometimes multiple Fragments are attached, but only one Fragment should be fully active.

Example: `ViewPager2`

```text
Page 1 - STARTED
Page 2 - RESUMED   ← currently selected page
Page 3 - STARTED
```

The side pages may already be created or kept nearby, but they should not run full foreground work.

Examples of work that should usually happen only in the `RESUMED` Fragment:

- Camera usage
- Sensor updates
- Location updates
- Heavy animations
- Expensive collectors
- Active playback

Only the currently selected Fragment should be `RESUMED`.

### Common use cases

- `ViewPager2`
- Bottom navigation with multiple attached Fragments
- Custom tab navigation
- Screens where multiple Fragments are visible but only one should be interactive

### Interview answer

> `setMaxLifecycle()` restricts the maximum lifecycle state a Fragment can reach. For example, if we set the max lifecycle to `STARTED`, the Fragment can be visible but it will not become `RESUMED`. This is useful in ViewPager2 or custom tab flows where multiple Fragments are attached, but only the selected Fragment should be fully active.

---

## FragmentFactory

`FragmentFactory` is the correct way to create Fragments with constructor injection.

```kotlin
class MyFragmentFactory(
    private val dependency: SomeDependency
) : FragmentFactory() {

    override fun instantiate(
        classLoader: ClassLoader,
        className: String
    ): Fragment {
        return when (loadFragmentClass(classLoader, className)) {
            DetailFragment::class.java -> DetailFragment(dependency)
            else -> super.instantiate(classLoader, className)
        }
    }
}
```

### Interview answer

> Normally, I avoid custom constructors in Fragments and pass runtime data through arguments. If constructor injection is needed, FragmentFactory is the correct mechanism.

---

## Fragment testing

Useful tools:

| Test area | Tool |
|---|---|
| Fragment UI | FragmentScenario |
| Navigation | TestNavHostController |
| ViewModel state | Fake repository |
| Flow collection | Turbine and `runTest` |
| UI behaviour | Espresso or Compose testing |

Example:

```kotlin
launchFragmentInContainer<SampleFragment>()
```

### Interview answer

> I test Fragments in isolation using FragmentScenario, inject fake dependencies, and verify navigation using TestNavHostController.

---

## Compose inside Fragment

Compose can be hosted inside a Fragment using `ComposeView`.

```kotlin
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View {
    return ComposeView(requireContext()).apply {
        setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
        )

        setContent {
            MyScreen()
        }
    }
}
```

### Interview answer

> When using Compose inside Fragments, I set the composition strategy so the composition is disposed with the Fragment view lifecycle. This avoids leaking Compose UI after `onDestroyView()`.

---

## Fragment interview checklist

- Fragment lifecycle vs view lifecycle
- `viewLifecycleOwner`
- `onCreateView()` vs `onViewCreated()`
- FragmentManager and FragmentTransaction
- `add()` vs `replace()`
- Back stack behaviour
- `commit()` vs `commitNow()` vs `commitAllowingStateLoss()`
- Fragment arguments and Safe Args
- Fragment Result API
- Shared ViewModel communication
- `childFragmentManager` vs `parentFragmentManager`
- DialogFragment
- ViewPager2 with `FragmentStateAdapter`
- Memory leaks in Fragments
- Configuration changes
- Process death and `SavedStateHandle`
- Jetpack Navigation Component
- RecyclerView state restoration
- `setMaxLifecycle()`
- FragmentFactory
- Fragment testing
- Compose inside Fragment

## Top 5 Fragment topics to master deeply

1. Fragment lifecycle vs view lifecycle
2. Back stack and FragmentTransaction behaviour
3. ViewModel, SavedStateHandle, arguments, and process death
4. Memory leaks with binding, observers, adapters, and listeners
5. Navigation Component and communication between Fragments
