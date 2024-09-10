

# Background Location Tracking in Flutter

This project demonstrates how to implement **background location tracking** using `flutter_background_service`, `Geolocator`, `Dio`, `flutter_local_notifications`, and `permission_handler`. This setup is ideal for applications that require constant location updates even when the app is in the background.

## Getting Started

### Prerequisites

Before you begin, ensure that you have the following dependencies added to your `pubspec.yaml` file:

```yaml
dependencies:
  dio: ^5.0.0
  geolocator: ^9.0.0
  permission_handler: ^10.2.0
  flutter_local_notifications: ^9.0.0
  flutter_background_service: ^2.4.6
  flutter_background_service_android: ^2.4.6
  get_it: ^7.2.0  # Optional for dependency injection
```

After adding these dependencies, run:
```bash
flutter pub get
```

### Project Setup

#### Step 1: Create `LocationServiceController`

In your project, create a file named `location_service_controller.dart` and add the following code to handle location tracking, permission requests, and notifications.

```dart
import 'dart:async';
import 'package:dio/dio.dart';
import 'package:flutter_background_service/flutter_background_service.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:geolocator/geolocator.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:flutter_background_service_android/flutter_background_service_android.dart';

@pragma('vm:entry-point')
void onStart(ServiceInstance service) async {
  print("onStart");

  if (service is AndroidServiceInstance) {
    service.setAsForegroundService();

    service.on('stopService').listen((event) {
      service.stopSelf();
    });

    FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
        FlutterLocalNotificationsPlugin();

    flutterLocalNotificationsPlugin.show(
      888,
      'Location Tracking',
      'Tracking location in the background.',
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'my_foreground',
          'Location Tracking',
          channelDescription: 'Background location tracking.',
          importance: Importance.low,
        ),
      ),
    );
  }

  Dio dio = Dio();
  int counter = 1;

  Timer.periodic(const Duration(seconds: 5), (timer) async {
    await sendLocation(dio, counter);
    counter++;
  });

  service.on('stopService').listen((event) {
    timer.cancel();
    service.stopSelf();
  });
}

Future<void> sendLocation(Dio dio, int counter) async {
  Position position = await Geolocator.getCurrentPosition(
    desiredAccuracy: LocationAccuracy.high,
  );

  var headers = {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Authorization': 'Bearer YOUR_TOKEN_HERE',
  };

  var data = {
    'lat': position.latitude.toString(),
    'long': position.longitude.toString(),
    'driver_id': "1",
    'order_id': '1',
  };

  try {
    var response = await dio.post(
      'https://your-api-endpoint.com/track-location',
      data: data,
      options: Options(headers: headers),
    );

    if (response.statusCode == 200) {
      print('Location sent: ${response.data}');
    } else {
      print('Error: ${response.statusMessage}');
    }
  } catch (e) {
    print('Request failed: $e');
  }
}
```

### Step 2: Update `AndroidManifest.xml`

To properly enable background location tracking, you need to modify the `AndroidManifest.xml` file. This involves adding the necessary permissions, services, and declaring the appropriate namespaces.

#### Step-by-Step Instructions:

1. **Add Permissions**: These permissions are needed to access the device's location and use background services.

    In your `android/app/src/main/AndroidManifest.xml`, inside the `<manifest>` tag (at the top of the file), add the following permissions:

    ```xml
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    ```

    These permissions ensure that your app can access the internet, get the location (both fine and coarse), and run location services in the background.

2. **Declare the `tools` Namespace**: To modify the service configuration in Android, you'll need to use the `tools` namespace. This allows you to override certain attributes.

    In the `<manifest>` tag (right after the `xmlns:android`), add the following line to declare the `tools` namespace:

    ```xml
    xmlns:tools="http://schemas.android.com/tools"
    ```

    This is necessary because later, we will be using `tools:replace="android:exported"` to override the default exported value of the service.

3. **Add the Service Declaration**: Background location tracking requires a foreground service. This service needs to be declared inside the `<application>` tag.

    Add the following code inside the `<application>` tag (after the `<activity>` tag):

    ```xml
    <service
        android:name="id.flutter.flutter_background_service.BackgroundService"
        android:exported="false"
        android:foregroundServiceType="location"
        tools:replace="android:exported" />
    ```

    This declaration enables the background service for location tracking.

4. **Check the `MainActivity` Declaration**: Ensure that your `MainActivity` is properly declared in the `<application>` tag, like this:

    ```xml
    <activity
        android:name=".MainActivity"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    ```

#### Final Example of `AndroidManifest.xml`

After completing the above steps, your `AndroidManifest.xml` should look like this:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.yourapp">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

    <application
        android:label="YourApp"
        android:icon="@mipmap/ic_launcher">

        <!-- MainActivity -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Background Service for Location Tracking -->
        <service
            android:name="id.flutter.flutter_background_service.BackgroundService"
            android:exported="false"
            android:foregroundServiceType="location"
            tools:replace="android:exported" />
    </application>
</manifest>


#### Step 3: Initialization in `main.dart`

Add the following initialization logic to your `main.dart`:

```dart
import 'package:flutter/material.dart';
import 'location_service_controller.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:get_it/get_it.dart';

GetIt locator = GetIt.instance;

void setupLocator() {
  locator.registerSingleton<LocationServiceController>(
    LocationServiceController(
      dio: Dio(),
      flutterLocalNotificationsPlugin: FlutterLocalNotificationsPlugin(),
    ),
  );
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  setupLocator();

  final LocationServiceController locationServiceController = locator<LocationServiceController>();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: MyHomePage(),
    );
  }
}
```

#### Step 4: Request Permissions

Ensure the app properly requests location permissions:

```dart
Future<void> requestLocationPermissions() async {
  var status = await Permission.locationAlways.request();
  if (status.isGranted) {
    print("Location permission granted.");
  } else {
    print("Location permission denied.");
  }
}
```

---

## Common Issues and Fixes

### 1. **Invalid Package Name**
   **Error**: `attribute 'package' in <manifest> tag is not a valid Android package name.`
   
   **Fix**: Ensure the package name does not contain hyphens. Update it to something like `com.example.yourapp`.

### 2. **Kotlin Compilation Error**
   **Error**: `Execution failed for task ':app:compileDebugKotlin'.`
   
   **Fix**: Ensure your `MainActivity.kt` file is formatted correctly:
```kotlin
package com.example.yourapp

import io.flutter.embedding.android.FlutterActivity

class MainActivity: FlutterActivity() {
}
```

### 3. **Permissions Denied**
   **Error**: `Location permission denied.`
   
   **Fix**: Make sure `ACCESS_BACKGROUND_LOCATION` is requested for Android 10 and above.

### 4. **Gradle Version Conflict**
   **Error**: `A problem occurred evaluating project ':app'.`
   
   **Fix**: Update your `android/build.gradle`:
```gradle
buildscript {
    ext.kotlin_version = '1.7.10'
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath 'com.android.tools.build:gradle:7.1.2'
    }
}
```

---

## Running the Project

To run the project, use the following commands:

```bash
flutter clean
flutter pub get
flutter run
```

Test background tracking by using Android Studio's background execution testing tools.

---

## Conclusion

By following the steps outlined in this guide, you will be able to implement **background location tracking** in a Flutter app. For more details or to contribute to this project, feel free to open an issue or a pull request.
