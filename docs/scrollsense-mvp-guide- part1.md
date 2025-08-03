# ScrollSense MVP: Complete Flutter Development Guide

## Table of Contents
1. [Project Overview & Architecture](#project-overview--architecture)
2. [Development Environment Setup](#development-environment-setup)
3. [Project Structure Explained](#project-structure-explained)
4. [Phase 1: Flutter Foundation (Week 1)](#phase-1-flutter-foundation-week-1)
5. [Phase 2: Android Integration (Week 2)](#phase-2-android-integration-week-2)
6. [Phase 3: Data Management (Week 3)](#phase-3-data-management-week-3)
7. [Phase 4: UI Development (Week 4)](#phase-4-ui-development-week-4)
8. [Phase 5: MVP Polish (Week 5)](#phase-5-mvp-polish-week-5)
9. [Testing & Debugging Guide](#testing--debugging-guide)
10. [MVP Launch Checklist](#mvp-launch-checklist)

---

## Project Overview & Architecture

### What We're Building
ScrollSense tracks how much users scroll on their phones across all apps, converting scroll distance to real-world measurements (like "You scrolled the height of the Eiffel Tower today!").

### Core Architecture
```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   User Scrolls  │────▶│  Accessibility   │────▶│  Flutter App    │
│   in Any App    │     │     Service      │     │  (Processing)   │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                           │
                        ┌──────────────────┐     ┌─────────▼────────┐
                        │   UI Dashboard   │◀────│  Local Database  │
                        │  (Data Display)  │     │    (SQLite)      │
                        └──────────────────┘     └──────────────────┘
```

### Why This Architecture?
- **Accessibility Service**: Only way to detect scrolls across ALL apps on Android
- **Flutter App**: Handles UI, data processing, and user interactions
- **Local Database**: Privacy-first approach - all data stays on device
- **Platform Channels**: Bridge between Android native code and Flutter

---

## Development Environment Setup

### Step 1: Install Flutter (30 minutes)
```bash
# For Windows:
# 1. Download Flutter SDK from https://flutter.dev/docs/get-started/install/windows
# 2. Extract to C:\flutter (avoid spaces in path)
# 3. Add C:\flutter\bin to PATH

# For Mac:
brew install flutter

# For Linux:
sudo snap install flutter --classic

# Verify installation:
flutter doctor
```

### Step 2: Install Android Studio (45 minutes)
```bash
# 1. Download from https://developer.android.com/studio
# 2. During installation, ensure these are checked:
#    - Android SDK
#    - Android SDK Platform
#    - Android Virtual Device (AVD)

# 3. Open Android Studio
# 4. Go to SDK Manager (Tools > SDK Manager)
# 5. Install:
#    - Android SDK Platform 33
#    - Android SDK Build-Tools
#    - Android SDK Platform-Tools
#    - Android Emulator
```

### Step 3: Setup IDE (15 minutes)
```bash
# Install Flutter/Dart plugins in Android Studio:
# 1. File > Settings > Plugins
# 2. Search and install:
#    - Flutter
#    - Dart

# Alternative: Use VS Code
# 1. Install VS Code
# 2. Install extensions:
#    - Flutter
#    - Dart
#    - Awesome Flutter Snippets
```

### Step 4: Create Android Virtual Device (AVD)
```bash
# In Android Studio:
# 1. Tools > AVD Manager
# 2. Create Virtual Device
# 3. Choose: Pixel 4 (good balance of features)
# 4. System Image: Android 11.0 (API 30)
# 5. Finish and launch emulator
```

---

## Project Structure Explained

### Complete Project Structure
```
scrollsense/
├── android/                          # Android-specific code
│   ├── app/
│   │   ├── src/
│   │   │   └── main/
│   │   │       ├── java/             # Native Android code
│   │   │       │   └── com/example/scrollsense/
│   │   │       │       ├── MainActivity.java
│   │   │       │       └── ScrollAccessibilityService.java
│   │   │       ├── res/
│   │   │       │   └── xml/
│   │   │       │       └── accessibility_service_config.xml
│   │   │       └── AndroidManifest.xml
│   │   └── build.gradle
│   └── build.gradle
│
├── lib/                              # Flutter/Dart code
│   ├── main.dart                     # App entry point
│   ├── config/
│   │   └── constants.dart            # App-wide constants
│   ├── models/                       # Data structures
│   │   ├── scroll_event.dart
│   │   ├── scroll_session.dart
│   │   └── daily_stats.dart
│   ├── services/                     # Business logic
│   │   ├── accessibility_service.dart
│   │   ├── database_service.dart
│   │   ├── scroll_calculator.dart
│   │   └── notification_service.dart
│   ├── screens/                      # Full page views
│   │   ├── splash_screen.dart
│   │   ├── onboarding_screen.dart
│   │   ├── home_screen.dart
│   │   ├── stats_screen.dart
│   │   └── settings_screen.dart
│   ├── widgets/                      # Reusable UI components
│   │   ├── stats_card.dart
│   │   ├── scroll_chart.dart
│   │   └── app_usage_list.dart
│   └── utils/                        # Helper functions
│       ├── formatters.dart
│       └── permissions.dart
│
├── assets/                           # Images, fonts, etc.
│   └── images/
│       └── logo.png
│
├── test/                             # Unit and widget tests
│   └── widget_test.dart
│
└── pubspec.yaml                      # Dependencies & metadata
```

### Why This Structure?
- **Separation of Concerns**: Each folder has a specific purpose
- **Scalability**: Easy to add new features without cluttering
- **Maintainability**: Clear where to find/add specific functionality
- **Testing**: Mirrors lib/ structure for easy test organization

---

## Phase 1: Flutter Foundation (Week 1)

### Day 1-2: Project Creation and Basic Setup

#### Step 1: Create the Flutter Project
```bash
# Create project
flutter create scrollsense
cd scrollsense

# Test it runs
flutter run
```

#### Step 2: Configure pubspec.yaml
**File: `pubspec.yaml`**
```yaml
name: scrollsense
description: Track your scroll distance across all apps
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  
  # Local database for storing scroll data
  sqflite: ^2.3.0
  path: ^1.8.3
  
  # For app settings and preferences
  shared_preferences: ^2.2.0
  
  # Beautiful charts for data visualization
  fl_chart: ^0.63.0
  
  # Handle Android permissions
  permission_handler: ^11.0.0
  
  # Background task scheduling
  workmanager: ^0.5.1
  
  # Get information about installed apps
  device_apps: ^2.2.0
  
  # Push notifications for break reminders
  flutter_local_notifications: ^16.1.0
  
  # UI enhancements
  google_fonts: ^6.1.0
  
  # State management (simple and effective)
  provider: ^6.1.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  
  assets:
    - assets/images/
```

**Why these dependencies?**
- **sqflite**: Local SQL database for privacy-first data storage
- **fl_chart**: Creates beautiful, animated charts for scroll data
- **provider**: Simple state management perfect for MVP
- **workmanager**: Handles background processing efficiently
- **shared_preferences**: Stores user settings persistently

#### Step 3: Create App Entry Point
**File: `lib/main.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'screens/splash_screen.dart';
import 'screens/onboarding_screen.dart';
import 'screens/home_screen.dart';
import 'services/database_service.dart';

void main() async {
  // Ensure Flutter binding is initialized before accessing native features
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize database
  await DatabaseService.instance.database;
  
  runApp(const ScrollSenseApp());
}

class ScrollSenseApp extends StatelessWidget {
  const ScrollSenseApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ScrollSense',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        // Modern color scheme
        primarySwatch: Colors.blue,
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF2196F3),
          brightness: Brightness.light,
        ),
      ),
      // Start with splash screen to check onboarding status
      home: const SplashScreen(),
      routes: {
        '/onboarding': (context) => const OnboardingScreen(),
        '/home': (context) => const HomeScreen(),
      },
    );
  }
}
```

**Why this structure?**
- **WidgetsFlutterBinding**: Required for native platform interactions
- **async main**: Allows initialization of services before app starts
- **Material3**: Modern design system for better UI
- **Named routes**: Clean navigation management

### Day 3-4: Core Models and Constants

#### Step 4: Define Data Models
**File: `lib/models/scroll_event.dart`**
```dart
/// Represents a single scroll event captured from any app
class ScrollEvent {
  final int? id;
  final String packageName;     // Which app (com.instagram.android)
  final DateTime timestamp;      // When the scroll happened
  final double scrollDistance;   // Distance in pixels
  final int scrollX;            // Horizontal scroll component
  final int scrollY;            // Vertical scroll component
  
  ScrollEvent({
    this.id,
    required this.packageName,
    required this.timestamp,
    required this.scrollDistance,
    required this.scrollX,
    required this.scrollY,
  });
  
  // Convert to Map for database storage
  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'package_name': packageName,
      'timestamp': timestamp.millisecondsSinceEpoch,
      'scroll_distance': scrollDistance,
      'scroll_x': scrollX,
      'scroll_y': scrollY,
    };
  }
  
  // Create from database Map
  factory ScrollEvent.fromMap(Map<String, dynamic> map) {
    return ScrollEvent(
      id: map['id'],
      packageName: map['package_name'],
      timestamp: DateTime.fromMillisecondsSinceEpoch(map['timestamp']),
      scrollDistance: map['scroll_distance'],
      scrollX: map['scroll_x'],
      scrollY: map['scroll_y'],
    );
  }
}
```

**File: `lib/models/scroll_session.dart`**
```dart
/// Groups scroll events into meaningful sessions
class ScrollSession {
  final int? id;
  final String packageName;
  final DateTime startTime;
  final DateTime endTime;
  final double totalDistance;    // Total pixels scrolled
  final int eventCount;          // Number of scroll events
  
  ScrollSession({
    this.id,
    required this.packageName,
    required this.startTime,
    required this.endTime,
    required this.totalDistance,
    required this.eventCount,
  });
  
  // Calculate session duration
  Duration get duration => endTime.difference(startTime);
  
  // Average scroll speed (pixels per second)
  double get averageSpeed => 
    duration.inSeconds > 0 ? totalDistance / duration.inSeconds : 0;
  
  // Convert pixels to real-world distance (meters)
  // Average phone has ~450 PPI, so ~17700 pixels per meter
  double get distanceInMeters => totalDistance / 17700;
  
  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'package_name': packageName,
      'start_time': startTime.millisecondsSinceEpoch,
      'end_time': endTime.millisecondsSinceEpoch,
      'total_distance': totalDistance,
      'event_count': eventCount,
    };
  }
  
  factory ScrollSession.fromMap(Map<String, dynamic> map) {
    return ScrollSession(
      id: map['id'],
      packageName: map['package_name'],
      startTime: DateTime.fromMillisecondsSinceEpoch(map['start_time']),
      endTime: DateTime.fromMillisecondsSinceEpoch(map['end_time']),
      totalDistance: map['total_distance'].toDouble(),
      eventCount: map['event_count'],
    );
  }
}
```

**File: `lib/models/daily_stats.dart`**
```dart
/// Aggregated statistics for a single day
class DailyStats {
  final DateTime date;
  final double totalDistance;
  final Duration totalTime;
  final int sessionCount;
  final Map<String, double> appDistances;  // Distance per app
  
  DailyStats({
    required this.date,
    required this.totalDistance,
    required this.totalTime,
    required this.sessionCount,
    required this.appDistances,
  });
  
  // Fun comparisons for user engagement
  String get funComparison {
    final meters = totalDistance / 17700;
    
    if (meters < 100) {
      return "You scrolled ${meters.toStringAsFixed(1)}m - about the length of a football field!";
    } else if (meters < 300) {
      return "You scrolled ${meters.toStringAsFixed(1)}m - that's the height of the Eiffel Tower!";
    } else if (meters < 1000) {
      return "You scrolled ${meters.toStringAsFixed(1)}m - almost 1 kilometer!";
    } else {
      final km = meters / 1000;
      return "You scrolled ${km.toStringAsFixed(1)}km - that's a serious workout for your thumb!";
    }
  }
  
  // Get top 3 most scrolled apps
  List<MapEntry<String, double>> get topApps {
    final sorted = appDistances.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));
    return sorted.take(3).toList();
  }
}
```

**Why these models?**
- **ScrollEvent**: Raw data from accessibility service
- **ScrollSession**: Groups events into meaningful time periods
- **DailyStats**: Provides insights users actually care about
- **Real-world comparisons**: Makes abstract pixel counts relatable

#### Step 5: App Constants
**File: `lib/config/constants.dart`**
```dart
class AppConstants {
  // Session detection
  static const Duration sessionTimeout = Duration(minutes: 2);
  static const double minScrollDistance = 10.0;  // Ignore tiny scrolls
  
  // Pixel to real-world conversions
  static const double pixelsPerMeter = 17700;  // Based on average phone PPI
  
  // Notification settings
  static const int breakReminderMinutes = 30;
  static const String notificationChannelId = 'scroll_break_reminder';
  static const String notificationChannelName = 'Scroll Break Reminders';
  
  // Database
  static const String databaseName = 'scrollsense.db';
  static const int databaseVersion = 1;
  
  // Shared preferences keys
  static const String keyOnboardingComplete = 'onboarding_complete';
  static const String keyNotificationsEnabled = 'notifications_enabled';
  static const String keyBreakInterval = 'break_interval_minutes';
  static const String keyDailyGoal = 'daily_scroll_goal_meters';
  
  // App info
  static const String appName = 'ScrollSense';
  static const String appTagline = 'Discover how far you scroll';
  
  // Fun distance comparisons
  static const Map<double, String> distanceComparisons = {
    100: "Length of a football field",
    300: "Height of the Eiffel Tower",
    443: "Height of the Empire State Building",
    828: "Height of Burj Khalifa",
    1000: "1 kilometer!",
    8848: "Height of Mount Everest",
  };
}
```

### Day 5-6: Basic UI Screens

#### Step 6: Splash Screen
**File: `lib/screens/splash_screen.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../config/constants.dart';

class SplashScreen extends StatefulWidget {
  const SplashScreen({Key? key}) : super(key: key);

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  @override
  void initState() {
    super.initState();
    _navigateToNextScreen();
  }
  
  Future<void> _navigateToNextScreen() async {
    // Show splash for at least 2 seconds for branding
    await Future.delayed(const Duration(seconds: 2));
    
    // Check if user has completed onboarding
    final prefs = await SharedPreferences.getInstance();
    final hasCompletedOnboarding = 
        prefs.getBool(AppConstants.keyOnboardingComplete) ?? false;
    
    if (!mounted) return;
    
    // Navigate based on onboarding status
    Navigator.pushReplacementNamed(
      context,
      hasCompletedOnboarding ? '/home' : '/onboarding',
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.primary,
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // App icon
            Container(
              width: 120,
              height: 120,
              decoration: BoxDecoration(
                color: Colors.white,
                borderRadius: BorderRadius.circular(30),
                boxShadow: [
                  BoxShadow(
                    color: Colors.black.withOpacity(0.2),
                    blurRadius: 20,
                    offset: const Offset(0, 10),
                  ),
                ],
              ),
              child: Icon(
                Icons.swap_vert_rounded,
                size: 80,
                color: Theme.of(context).colorScheme.primary,
              ),
            ),
            const SizedBox(height: 30),
            // App name
            const Text(
              'ScrollSense',
              style: TextStyle(
                color: Colors.white,
                fontSize: 32,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 10),
            // Tagline
            const Text(
              'Discover how far you scroll',
              style: TextStyle(
                color: Colors.white70,
                fontSize: 16,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Why this splash screen?**
- **Branding**: Shows app identity immediately
- **Navigation logic**: Determines where user should go
- **Smooth UX**: Gives time for background initialization

---

## Phase 2: Android Integration (Week 2)

### Day 7-8: Accessibility Service Setup

#### Step 7: Create Android Accessibility Service
**File: `android/app/src/main/java/com/example/scrollsense/ScrollAccessibilityService.java`**
```java
package com.example.scrollsense;

import android.accessibilityservice.AccessibilityService;
import android.accessibilityservice.AccessibilityServiceInfo;
import android.view.accessibility.AccessibilityEvent;
import android.view.accessibility.AccessibilityNodeInfo;
import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.embedding.engine.dart.DartExecutor;
import io.flutter.plugin.common.MethodChannel;
import java.util.HashMap;
import java.util.Map;

public class ScrollAccessibilityService extends AccessibilityService {
    private static final String CHANNEL = "scrollsense/accessibility";
    private MethodChannel methodChannel;
    private long lastEventTime = 0;
    private static final long EVENT_THRESHOLD_MS = 50; // Debounce events
    
    @Override
    protected void onServiceConnected() {
        super.onServiceConnected();
        
        // Configure service
        AccessibilityServiceInfo info = new AccessibilityServiceInfo();
        info.eventTypes = AccessibilityEvent.TYPE_VIEW_SCROLLED;
        info.feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC;
        info.notificationTimeout = 100;
        info.flags = AccessibilityServiceInfo.DEFAULT;
        setServiceInfo(info);
        
        // Setup Flutter method channel
        setupFlutterEngine();
    }
    
    private void setupFlutterEngine() {
        // This is a simplified version - in production, you'd need proper engine management
        // For MVP, we'll communicate through SharedPreferences
    }
    
    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        if (event.getEventType() != AccessibilityEvent.TYPE_VIEW_SCROLLED) {
            return;
        }
        
        // Debounce rapid scroll events
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastEventTime < EVENT_THRESHOLD_MS) {
            return;
        }
        lastEventTime = currentTime;
        
        // Extract scroll information
        String packageName = event.getPackageName() != null ? 
            event.getPackageName().toString() : "unknown";
        
        // Get scroll details
        int scrollX = event.getScrollX();
        int scrollY = event.getScrollY();
        
        // Calculate scroll distance (Pythagorean theorem)
        double scrollDistance = Math.sqrt(scrollX * scrollX + scrollY * scrollY);
        
        // For MVP, we'll store events in SharedPreferences
        // In production, use proper IPC or database
        storeScrollEvent(packageName, scrollDistance, scrollX, scrollY, currentTime);
    }
    
    private void storeScrollEvent(String packageName, double distance, 
                                 int scrollX, int scrollY, long timestamp) {
        // Store in SharedPreferences for Flutter to read
        // This is simplified for MVP - production would use better IPC
        android.content.SharedPreferences prefs = 
            getSharedPreferences("scroll_events", MODE_PRIVATE);
        android.content.SharedPreferences.Editor editor = prefs.edit();
        
        // Create unique key for this event
        String key = "event_" + timestamp;
        String value = packageName + "," + distance + "," + 
                      scrollX + "," + scrollY + "," + timestamp;
        
        editor.putString(key, value);
        editor.apply();
    }
    
    @Override
    public void onInterrupt() {
        // Handle service interruption
    }
}
```

**Why this implementation?**
- **Event filtering**: Only tracks scroll events for efficiency
- **Debouncing**: Prevents overwhelming the system with rapid events
- **Distance calculation**: Uses Pythagorean theorem for accurate measurement
- **SharedPreferences**: Simple IPC for MVP (upgrade to proper channels later)

#### Step 8: Configure Accessibility Service
**File: `android/app/src/main/res/xml/accessibility_service_config.xml`**
```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeViewScrolled"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault"
    android:canRetrieveWindowContent="true"
    android:description="@string/accessibility_service_description"
    android:notificationTimeout="100"
    android:packageNames=""
    android:settingsActivity=".MainActivity" />
```

**File: `android/app/src/main/res/values/strings.xml`**
```xml
<resources>
    <string name="app_name">ScrollSense</string>
    <string name="accessibility_service_description">
        ScrollSense needs accessibility access to track scroll distance across all your apps. 
        This helps you understand your scrolling habits and digital wellness. 
        Your data stays private on your device.
    </string>
</resources>
```

#### Step 9: Update Android Manifest
**File: `android/app/src/main/AndroidManifest.xml`**
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    
    <!-- Permissions -->
    <uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE" />
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    
    <application
        android:label="scrollsense"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        
        <!-- Main Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">
            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme" />
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <!-- Accessibility Service -->
        <service
            android:name=".ScrollAccessibilityService"
            android:exported="true"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_service_config" />
        </service>
        
        <!-- Flutter meta-data -->
        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>
</manifest>
```

### Day 9-10: Flutter-Android Communication

#### Step 10: Create Flutter Accessibility Service Bridge
**File: `lib/services/accessibility_service.dart`**
```dart
import 'package:flutter/services.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:async';
import '../models/scroll_event.dart';

class AccessibilityService {
  static const MethodChannel _channel = 
      MethodChannel('scrollsense/accessibility');
  
  // Singleton pattern for service access
  static final AccessibilityService instance = AccessibilityService._();
  AccessibilityService._();
  
  // Stream controller for real-time scroll events
  final _scrollEventController = StreamController<ScrollEvent>.broadcast();
  Stream<ScrollEvent> get scrollEventStream => _scrollEventController.stream;
  
  // Timer for polling SharedPreferences (MVP approach)
  Timer? _pollingTimer;
  
  /// Check if accessibility service is enabled
  Future<bool> isAccessibilityEnabled() async {
    try {
      // For MVP, we check if our service is running
      // In production, implement proper platform channel
      final prefs = await SharedPreferences.getInstance();
      final lastEventTime = prefs.getInt('last_event_time') ?? 0;
      final currentTime = DateTime.now().millisecondsSinceEpoch;
      
      // If we received an event in the last 5 seconds, service is running
      return (currentTime - lastEventTime) < 5000;
    } catch (e) {
      print('Error checking accessibility: $e');
      return false;
    }
  }
  
  /// Open accessibility settings
  Future<void> openAccessibilitySettings() async {
    try {
      await _channel.invokeMethod('openAccessibilitySettings');
    } catch (e) {
      print('Error opening settings: $e');
    }
  }
  
  /// Start monitoring for scroll events
  void startMonitoring() {
    // Poll SharedPreferences for new events (MVP approach)
    _pollingTimer?.cancel();
    _pollingTimer = Timer.periodic(const Duration(seconds: 1), (_) {
      _checkForNewEvents();
    });
  }
  
  /// Stop monitoring
  void stopMonitoring() {
    _pollingTimer?.cancel();
  }
  
  /// Check SharedPreferences for new scroll events
  Future<void> _checkForNewEvents() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final keys = prefs.getKeys().where((key) => key.startsWith('event_'));
      
      for (String key in keys) {
        final eventData = prefs.getString(key);
        if (eventData != null) {
          // Parse event data
          final parts = eventData.split(',');
          if (parts.length >= 5) {
            final event = ScrollEvent(
              packageName: parts[0],
              scrollDistance: double.parse(parts[1]),
              scrollX: int.parse(parts[2]),
              scrollY: int.parse(parts[3]),
              timestamp: DateTime.fromMillisecondsSinceEpoch(
                int.parse(parts[4])
              ),
            );
            
            // Emit event
            _scrollEventController.add(event);
            
            // Remove processed event
            prefs.remove(key);
          }
        }
      }
    } catch (e) {
      print('Error checking events: $e');
    }
  }
  
  /// Clean up resources
  void dispose() {
    _pollingTimer?.cancel();
    _scrollEventController.close();
  }
}
```

**Why this approach