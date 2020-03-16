# Beeline Android SDK

Integrate Beeline routing and navigation into your Android app

## Requirements

- Supports **Android 19 (4.4 KitKat)** and above
- Supports **armeabi-v7a**, **arm64-v8a**, **x86**, **x86_64**

- Written in **Kotlin** and **C++**
- Depends on **RxJava2**

Note: This guide has been written assuming the reader has a good understanding of RxJava2.

## Installation

Contact us for credentials tech@beeline.co


### Add Beeline SDK dependency

Register the Beeline maven repository in your project level `build.gradle` file:

```gradle
allprojects {

  repositories {

    // other repositories omitted

    maven {
      name = "GitHubPackages"
      url = uri("https://maven.pkg.github.com/RideBeeline/packages")
      credentials {
        username = 'username'
        password = 'password'
      }
    }
  }

}
```

In your module level `build.gradle` file:

```gradle
dependencies {
  implementation 'co.beeline:beeline-sdk:1.0.0'
}
```


### Specify permissions

The Beeline SDK requires the `INTERNET` and `ACCESS_FINE_LOCATION` permissions.

In your `AndroidManifest.xml`

```xml
<!-- Under manifest tag -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

We leave it to you to ensure you request the appropriate location permissions from your users.

See https://developer.android.com/training/permissions/requesting


### Initialise Beeline SDK

Before using any Beeline SDK components you must initialise the Beeline SDK.

We suggest placing this call in your `Application.onCreate()`.

```kotlin
BeelineSDK.initialise()
```

### Create a Beeline module

For convenience we have included a `BeelineModule` class that creates an instance of each of the classes you'll need for routing and navigation.

The `instance`, `key` and `group` parameters will be provided to you by Beeline.

You can also specify the distance unit to use for formatting distances, the default value is `DistanceUnit.METRIC`.

```kotlin
val beelineModule = BeelineModule(context, instance, key, group, distanceUnit = DistanceUnit.METRIC)

// RouteProvider for requesting a route from Beeline's routing API
val routeProvider = beelineModule.routeProvider

// Factory function for creating a NavigationController
val navigationController = beelineModule.navigationController(route)

// Factory function for creating a NavigationViewModel
val navigationViewModel = beelineModule.navigationViewModel(navigationController)
```

### Request a route

Start by requesting a route from the Beeline routing API

```kotlin
val parameters = RouteParameters(
  vehicle = Vehicle.Bicycle,
  start = Coordinate(51.50804, -0.12807),
  end = Coordinate(51.504500, -0.086500)
)

val disposable = beelineModule.routeProvider.route(parameters)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
        { routeResponse ->
            startNavigation(routeResponse.route)
        },
        { error ->
            Toast.makeText(this, error.localizedMessage, Toast.LENGTH_LONG).show()
        }
    )

// Remember to handle your Rx disposables

private fun startNavigation(route: TrackRoute) {
  val intent = Intent(this, NavigationActivity::class.java).apply {
      putExtra(NavigationActivity.EXTRA_ROUTE, route)
  }
  startActivity(intent)
}
```

### Display route on a map

The route response contains a `polyline` that you can use to display the route on a map.

### Create a navigation Activity

Create a new `Activity` and register it in your `AndroidManifest.xml` with the `theme` provided by Beeline.

```xml
<!-- Under application tag -->
<activity
  android:name="x.y.z.NavigationActivity"
  android:theme="@style/BeelineNavigationTheme" />
```

### Example navigation Activity implementation

```kotlin
class NavigationActivity : AppCompatActivity() {

    companion object {
        const val EXTRA_ROUTE = "route"
    }

    private lateinit var viewHolder: NavigationViewHolder
    private lateinit var navigationController: NavigationController

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.navigation)

        // inject or lookup as best suits your application
        val beelineModule: BeelineModule

        // get the route from the intent
        val route = intent.getParcelableExtra<TrackRoute>(EXTRA_ROUTE)!!

        // create a navigation controller for the route
        navigationController = beelineModule.navigationController(route)

        // create a view model for the navigation controller
        val navigationViewModel = beelineModule.navigationViewModel(navigationController)

        // create a view holder to bind the view to the view model
        viewHolder = NavigationViewHolder(findViewById(R.id.navigation), navigationViewModel)

        // implement the stop button
        viewHolder.stopButton.setOnClickListener {
            navigationController.stop()
            // show end of journey screen?
        }

        // implement the map button
        viewHolder.mapButton.setOnClickListener {
            // hide navigation and show map?
        }

        // start navigation
        navigationController.start()

        // Set the battery % and range
        viewHolder.batteryValueTextView.text = NumberFormat.getPercentInstance().format(0.95)
        viewHolder.rangeTextView.text = beelineModule.distanceFormatter.formatDistance(125_000.0).combined
    }

    override fun onStart() {
        super.onStart()
        viewHolder.onStart()
    }

    override fun onStop() {
        super.onStop()
        viewHolder.onStop()
    }
}

```

### Running in the background

If your app is moved to the background, location updates will stop unless you have a foreground `Service` running with the `android:foregroundServiceType="location"` attribute. For the best experience we recommend enabling location updates whilst the app is in the background, but this is not mandatory.

For more information see the Android documentation on accessing location in the background [here](https://developer.android.com/training/location/background#continue-user-initiated-action).
