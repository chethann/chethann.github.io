---
title: ViewModels with hilt and compose navigation
date: 2024-11-07 12:00:00 +0000
categories: [android, compose]
tags: [viewmodel, hilt, compose navigation]
---

# ViewModels with hilt and compose navigation

As per the official documentation, hilt is the recommended solution for dependency injection in Android apps, and works seamlessly with Compose. Hilt also integrates with the Navigation Compose library and gives a developer-friendly API to create ViewModels in Compose projects. One can also check https://developer.android.com/jetpack/compose/libraries#hilt-navigation

We can use ***hiltViewModel()*** function to get an instance of a ViewModel which is annotated with ***@HiltViewModel***. Ex:

```kotlin
val viewModel = hiltViewModel<MyViewModel>()
```

This piece of code gives a ViewModel scoped to the current backStackEntry of the navigation. NavBackStackEntry implements ViewModelStoreOwner and the hiltViewModel method accepts the ViewModelStoreOwner as an optional param. The default value is set to LocalViewModelStoreOwner's (a *CompositionLocal*) current value, which points to the current NavBackStackEntry.

In most cases just calling hiltViewModel method suffices our development needs. However, there are obvious exceptions when we want a ViewModel to be scoped to something else apart from the current backStackEntry of the navigation graph.


## Scopping ViewModel to the entire route / graph


One of the simplest such use cases is to have a ViewModel scoped to the entire route (or sub-graph), not just the current destination, in such cases, we can pass the backStack entry of the route instead of the current destination's backStackEntry. Below is a sample code to do so.

```kotlin
fun NavGraphBuilder.loginGraph(navController: NavController) {
    navigation(
        startDestination = "login/phoneNumber",
        route = "login"
    ) {

        composable("login/phoneNumber") {
            val parentEntry = remember(it) {
                navController.getBackStackEntry("login")
            }
            val loginViewModel: LoginViewModel = hiltViewModel(parentEntry)
            LoginMobileNumberScreen(navController = navController, loginViewModel)
        }
        composable("login/otp") {
            val parentEntry = remember(it) {
                navController.getBackStackEntry("login")
            }
            val loginViewModel: LoginViewModel = hiltViewModel(parentEntry)
            LoginOTPScreen(navController = navController, loginViewModel)
        }
        composable("login/error") {
            val parentEntry = remember(it) {
                navController.getBackStackEntry("login")
            }
            val loginViewModel: LoginViewModel = hiltViewModel(parentEntry)
            LoginErrorScreen(navController = navController, loginViewModel = loginViewModel)
        }

    }
}
```

In the above code, all three screens (Mobile Number collection, OTP submission and error) in the loginGraph share a common ViewModel. This can be convenient when we have to share the UI states / fetched data with all the screens within a graph.

One can pass any navBackStack entry to get a viewModel scoped to it. One of the use cases can be sharing a ViewModel between a screen and its next screen. This one is convenient, especially in cases where we have a screen and a bottom sheet in that screen and we model bottom sheets as bottomSheet (We found this to be a more scalable way instead of using ModelBottomSheet) which can be navigated to just like any other destinations. We can pass previousBackStackEntry in this case in bottomSheet to get the same ViewModel instance at that of the main screen.

```kotlin
fun NavGraphBuilder.bottomSheetGraph(navController: NavController) {
    bottomSheet("bottomsheet/qwerty") {
        val sharedViewModel: SharedViewModel = hiltViewModel(navController.previousBackStackEntry!!)
        BottomSheetView(sharedViewModel, navController)
    }
}
```


**Note**: *We need to ensure we navigate to this destination (bottomsheet) only from a host screen.*

## Multiple ViewModels of the the same type within a scope

***hiltViewModel*** method gives the same instance of the ViewModel (of a particular type) if it is called multiple times within a screen, for obvious reasons already discussed. There can be cases where we want to have multiple ViewModels of the same type within a screen. One of the cases where this comes in handy is when we have different tabs that user can select and we want to have the logic of fetching data for each of the tabs in a separate ViewModel. Also, if the user switches back to a tab for which the data is fetched previously, we want to show that immediately.

This can be achieved by introducing a unique key for each instance of ViewModel we want to create. Unfortunately, hilt doesn't expose such a method to developers right now. However, we can quickly write our own version of hiltViewModel method that accepts a key.

Code for the same:
```kotlin
/**
 * A wrapper to get hilt VMs by key. This can be used when we need different instance of the same ViewModel within the same NavBackStackEntry
 * Most of the logic is copied from [androidx.hilt.navigation.compose.hiltViewModel]
 * and [androidx.lifecycle.viewmodel.compose.viewModel]
 */

@Composable
inline fun <reified VM : ViewModel> hiltViewModelForKey(key: String, navBackStackEntry: NavBackStackEntry): VM {
    return viewModel(
        key = key,
        viewModelStoreOwner = navBackStackEntry,
        modelClass = VM::class.java,
        factory = HiltViewModelFactory(
            context = LocalContext.current,
            navBackStackEntry = navBackStackEntry
        )
    )
}
```

Here we take advantage of the fact that ViewModel method already supports this way of passing a key to get different instances of ViewModels within a scope based on the key.

One can even combine both the techniques of passing a key and passing previousBackStackEntry in case the UI has multiple tabs and we need to show a bottom sheet on click of some element in that particular tab. In this case, we need to pass the key to the bottom sheet as a navigation parameter.

Code sample:
```kotlin
bottomSheet("bottomsheet/qwerty") {
        val key = it.arguments?.getString("key") ?: ""
        val sharedViewModel: SharedViewModel = hiltViewModelForKey(
            key = key,
            navBackStackEntry = navController.previousBackStackEntry!!
        )
        BottomSheetView(sharedViewModel, navController)
    }
```
You can access a ViewModel from any backStackEntry this was, but accessing a backStackEntry deep inside the current navigation stack is not recommended. Also, we need to make sure BackStackEntry we are trying to access is not empty. Only then we can use this approach.

In some rare cases, if we need a ViewModel scoped at the activity level (can be useful if we want to share the ViewModel for activity and some usecase inside of the compose destinations) then we can create it at the activity level and this can be passed to NavHost down to all the destinations where it is needed.

If we need a ViewModel scoped to the entire NavHost, we can create it in NavHost and pass it down to all the sub-graphs or destinations.

```kotlin
NavHost(
            navController = rememberNavController(),
            startDestination = "startDestination"
        ) {
            val commonViewModel: CommonViewModel = hiltViewModel()
            myGraph(navController, commonViewModel)
        }
```

**Conclusion**: We can use different ways like the above to create ViewModels when using hilt and compose navigation to fit our use case.