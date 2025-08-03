**Why this approach?**
- **SharedPreferences**: Simple, reliable IPC method for MVP
- **Polling approach**: Easy to debug, won't crash app if service fails
- **Event parsing**: Straightforward string format for rapid development
- **Upgradeable**: Can easily switch to MethodChannel later for production

#### Step 11: Update MainActivity for Platform Communication
**File: `android/app/src/main/java/com/example/scrollsense/MainActivity.java`**
```java
package com.example.scrollsense;

import android.content.Intent;
import android.provider.Settings;
import io.flutter.embedding.android.FlutterActivity;
import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.plugin.common.MethodChannel;

public class MainActivity extends FlutterActivity {
    private static final String CHANNEL = "scrollsense/accessibility";
    
    @Override
    public void configureFlutterEngine(FlutterEngine flutterEngine) {
        super.configureFlutterEngine(flutterEngine);
        
        new MethodChannel(flutterEngine.getDartExecutor().getBinaryMessenger(), CHANNEL)
            .setMethodCallHandler((call, result) -> {
                switch (call.method) {
                    case "openAccessibilitySettings":
                        openAccessibilitySettings();
                        result.success(null);
                        break;
                    case "isAccessibilityEnabled":
                        result.success(isAccessibilityServiceEnabled());
                        break;
                    default:
                        result.notImplemented();
                        break;
                }
            });
    }
    
    private void openAccessibilitySettings() {
        Intent intent = new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
    
    private boolean isAccessibilityServiceEnabled() {
        // Check if our accessibility service is enabled
        String service = getPackageName() + "/" + ScrollAccessibilityService.class.getName();
        String enabledServices = Settings.Secure.getString(
            getContentResolver(),
            Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES
        );
        
        return enabledServices != null && enabledServices.contains(service);
    }
}
```

### Day 11-12: Database Service Implementation

#### Step 12: Create Database Service
**File: `lib/services/database_service.dart`**
```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import '../models/scroll_event.dart';
import '../models/scroll_session.dart';
import '../models/daily_stats.dart';
import '../config/constants.dart';

class DatabaseService {
  static final DatabaseService instance = DatabaseService._();
  DatabaseService._();
  
  static Database? _database;
  
  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  Future<Database> _initDatabase() async {
    String path = join(await getDatabasesPath(), AppConstants.databaseName);
    
    return await openDatabase(
      path,
      version: AppConstants.databaseVersion,
      onCreate: _createTables,
      onUpgrade: _onUpgrade,
    );
  }
  
  /// Create all database tables
  Future<void> _createTables(Database db, int version) async {
    // Scroll events table - raw data from accessibility service
    await db.execute('''
      CREATE TABLE scroll_events(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        package_name TEXT NOT NULL,
        timestamp INTEGER NOT NULL,
        scroll_distance REAL NOT NULL,
        scroll_x INTEGER NOT NULL,
        scroll_y INTEGER NOT NULL,
        created_at INTEGER DEFAULT (strftime('%s', 'now') * 1000)
      )
    ''');
    
    // Scroll sessions table - grouped meaningful interactions
    await db.execute('''
      CREATE TABLE scroll_sessions(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        package_name TEXT NOT NULL,
        start_time INTEGER NOT NULL,
        end_time INTEGER NOT NULL,
        total_distance REAL NOT NULL,
        event_count INTEGER NOT NULL,
        created_at INTEGER DEFAULT (strftime('%s', 'now') * 1000)
      )
    ''');
    
    // Daily aggregates table - for quick dashboard loading
    await db.execute('''
      CREATE TABLE daily_stats(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        date TEXT NOT NULL UNIQUE,
        total_distance REAL NOT NULL,
        total_time_seconds INTEGER NOT NULL,
        session_count INTEGER NOT NULL,
        top_app TEXT,
        created_at INTEGER DEFAULT (strftime('%s', 'now') * 1000)
      )
    ''');
    
    // App usage table - per-app statistics
    await db.execute('''
      CREATE TABLE app_usage(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        date TEXT NOT NULL,
        package_name TEXT NOT NULL,
        total_distance REAL NOT NULL,
        total_time_seconds INTEGER NOT NULL,
        session_count INTEGER NOT NULL,
        created_at INTEGER DEFAULT (strftime('%s', 'now') * 1000),
        UNIQUE(date, package_name)
      )
    ''');
    
    // Create indexes for better query performance
    await db.execute('CREATE INDEX idx_events_timestamp ON scroll_events(timestamp)');
    await db.execute('CREATE INDEX idx_events_package ON scroll_events(package_name)');
    await db.execute('CREATE INDEX idx_sessions_start_time ON scroll_sessions(start_time)');
    await db.execute('CREATE INDEX idx_daily_stats_date ON daily_stats(date)');
  }
  
  /// Handle database upgrades (for future versions)
  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    // Add upgrade logic here when we release new versions
    if (oldVersion < 2) {
      // Example future upgrade
      // await db.execute('ALTER TABLE scroll_events ADD COLUMN new_field TEXT');
    }
  }
  
  /// Insert a new scroll event
  Future<int> insertScrollEvent(ScrollEvent event) async {
    final db = await database;
    
    try {
      return await db.insert(
        'scroll_events',
        event.toMap(),
        conflictAlgorithm: ConflictAlgorithm.replace,
      );
    } catch (e) {
      print('Error inserting scroll event: $e');
      return -1;
    }
  }
  
  /// Insert multiple scroll events efficiently
  Future<void> insertScrollEventsBatch(List<ScrollEvent> events) async {
    final db = await database;
    
    final batch = db.batch();
    for (ScrollEvent event in events) {
      batch.insert('scroll_events', event.toMap());
    }
    
    try {
      await batch.commit(noResult: true);
    } catch (e) {
      print('Error batch inserting events: $e');
    }
  }
  
  /// Get scroll events for a specific time range
  Future<List<ScrollEvent>> getScrollEvents({
    DateTime? startTime,
    DateTime? endTime,
    String? packageName,
    int? limit,
  }) async {
    final db = await database;
    
    String whereClause = '';
    List<dynamic> whereArgs = [];
    
    if (startTime != null) {
      whereClause += 'timestamp >= ?';
      whereArgs.add(startTime.millisecondsSinceEpoch);
    }
    
    if (endTime != null) {
      if (whereClause.isNotEmpty) whereClause += ' AND ';
      whereClause += 'timestamp <= ?';
      whereArgs.add(endTime.millisecondsSinceEpoch);
    }
    
    if (packageName != null) {
      if (whereClause.isNotEmpty) whereClause += ' AND ';
      whereClause += 'package_name = ?';
      whereArgs.add(packageName);
    }
    
    try {
      final List<Map<String, dynamic>> maps = await db.query(
        'scroll_events',
        where: whereClause.isEmpty ? null : whereClause,
        whereArgs: whereArgs.isEmpty ? null : whereArgs,
        orderBy: 'timestamp DESC',
        limit: limit,
      );
      
      return maps.map((map) => ScrollEvent.fromMap(map)).toList();
    } catch (e) {
      print('Error getting scroll events: $e');
      return [];
    }
  }
  
  /// Insert a scroll session
  Future<int> insertScrollSession(ScrollSession session) async {
    final db = await database;
    
    try {
      return await db.insert(
        'scroll_sessions',
        session.toMap(),
        conflictAlgorithm: ConflictAlgorithm.replace,
      );
    } catch (e) {
      print('Error inserting scroll session: $e');
      return -1;
    }
  }
  
  /// Get today's scroll statistics
  Future<DailyStats?> getTodayStats() async {
    final today = DateTime.now();
    return await getDailyStats(today);
  }
  
  /// Get daily statistics for a specific date
  Future<DailyStats?> getDailyStats(DateTime date) async {
    final db = await database;
    final dateString = '${date.year}-${date.month.toString().padLeft(2, '0')}-${date.day.toString().padLeft(2, '0')}';
    
    try {
      // Get daily aggregate
      final dailyResults = await db.query(
        'daily_stats',
        where: 'date = ?',
        whereArgs: [dateString],
      );
      
      // Get app breakdown
      final appResults = await db.query(
        'app_usage',
        where: 'date = ?',
        whereArgs: [dateString],
        orderBy: 'total_distance DESC',
      );
      
      if (dailyResults.isEmpty) {
        // Calculate stats from raw events if no aggregate exists
        return await _calculateDailyStatsFromEvents(date);
      }
      
      final daily = dailyResults.first;
      final appDistances = <String, double>{};
      
      for (var app in appResults) {
        appDistances[app['package_name'] as String] = 
            (app['total_distance'] as num).toDouble();
      }
      
      return DailyStats(
        date: date,
        totalDistance: (daily['total_distance'] as num).toDouble(),
        totalTime: Duration(seconds: daily['total_time_seconds'] as int),
        sessionCount: daily['session_count'] as int,
        appDistances: appDistances,
      );
    } catch (e) {
      print('Error getting daily stats: $e');
      return null;
    }
  }
  
  /// Calculate daily stats from raw events (fallback method)
  Future<DailyStats?> _calculateDailyStatsFromEvents(DateTime date) async {
    final startOfDay = DateTime(date.year, date.month, date.day);
    final endOfDay = startOfDay.add(const Duration(days: 1)).subtract(const Duration(microseconds: 1));
    
    final events = await getScrollEvents(
      startTime: startOfDay,
      endTime: endOfDay,
    );
    
    if (events.isEmpty) return null;
    
    double totalDistance = 0;
    final appDistances = <String, double>{};
    
    for (var event in events) {
      totalDistance += event.scrollDistance;
      appDistances[event.packageName] = 
          (appDistances[event.packageName] ?? 0) + event.scrollDistance;
    }
    
    return DailyStats(
      date: date,
      totalDistance: totalDistance,
      totalTime: const Duration(hours: 0), // TODO: Calculate from sessions
      sessionCount: 0, // TODO: Calculate from sessions
      appDistances: appDistances,
    );
  }
  
  /// Get weekly statistics (last 7 days)
  Future<List<DailyStats>> getWeeklyStats() async {
    final today = DateTime.now();
    final weekAgo = today.subtract(const Duration(days: 7));
    
    final stats = <DailyStats>[];
    
    for (int i = 0; i < 7; i++) {
      final date = weekAgo.add(Duration(days: i));
      final dayStats = await getDailyStats(date);
      if (dayStats != null) {
        stats.add(dayStats);
      }
    }
    
    return stats;
  }
  
  /// Clean up old data (privacy-first approach)
  Future<void> cleanupOldData({int daysToKeep = 30}) async {
    final db = await database;
    final cutoffTime = DateTime.now()
        .subtract(Duration(days: daysToKeep))
        .millisecondsSinceEpoch;
    
    try {
      // Delete old scroll events
      await db.delete(
        'scroll_events',
        where: 'timestamp < ?',
        whereArgs: [cutoffTime],
      );
      
      // Delete old sessions
      await db.delete(
        'scroll_sessions',
        where: 'start_time < ?',
        whereArgs: [cutoffTime],
      );
      
      print('Cleaned up data older than $daysToKeep days');
    } catch (e) {
      print('Error cleaning up old data: $e');
    }
  }
  
  /// Delete all user data (privacy compliance)
  Future<void> deleteAllData() async {
    final db = await database;
    
    try {
      await db.delete('scroll_events');
      await db.delete('scroll_sessions');
      await db.delete('daily_stats');
      await db.delete('app_usage');
      
      print('All user data deleted');
    } catch (e) {
      print('Error deleting all data: $e');
    }
  }
  
  /// Get database size for storage monitoring
  Future<int> getDatabaseSize() async {
    final db = await database;
    final path = db.path;
    
    try {
      final file = await File(path).stat();
      return file.size;
    } catch (e) {
      print('Error getting database size: $e');
      return 0;
    }
  }
}
```

**Why this database design?**
- **Normalized structure**: Separate tables for events, sessions, and aggregates
- **Privacy-first**: Built-in data cleanup and deletion methods
- **Performance optimized**: Indexes on frequently queried columns
- **Scalable**: Batch operations for handling high-frequency scroll events
- **Upgradeable**: Version management for future database changes

**Why this background processing approach?**
- **Battery efficient**: Uses WorkManager for intelligent scheduling
- **Crash resistant**: Each task is isolated and recoverable
- **Privacy compliant**: Automatic data cleanup prevents indefinite storage
- **Scalable**: Processes data in batches to handle high scroll volumes
- **User-transparent**: All processing happens in background

### Day 15-16: Permissions and User Onboarding

#### Step 15: Permission Handler Service
**File: `lib/utils/permissions.dart`**
```dart
import 'dart:io';
import 'package:flutter/services.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:device_info_plus/device_info_plus.dart';

class PermissionManager {
  static final PermissionManager instance = PermissionManager._();
  PermissionManager._();
  
  static const MethodChannel _channel = MethodChannel('scrollsense/accessibility');
  
  /// Check if accessibility service is enabled
  Future<bool> isAccessibilityEnabled() async {
    try {
      final bool result = await _channel.invokeMethod('isAccessibilityEnabled');
      return result;
    } catch (e) {
      print('Error checking accessibility permission: $e');
      return false;
    }
  }
  
  /// Open accessibility settings for user to enable service
  Future<void> openAccessibilitySettings() async {
    try {
      await _channel.invokeMethod('openAccessibilitySettings');
    } catch (e) {
      print('Error opening accessibility settings: $e');
    }
  }
  
  /// Check notification permission (Android 13+)
  Future<bool> isNotificationEnabled() async {
    if (Platform.isAndroid) {
      final androidInfo = await DeviceInfoPlugin().androidInfo;
      
      // Android 13+ requires explicit notification permission
      if (androidInfo.version.sdkInt >= 33) {
        final status = await Permission.notification.status;
        return status.isGranted;
      }
    }
    
    // For older Android versions, notifications are enabled by default
    return true;
  }
  
  /// Request notification permission
  Future<bool> requestNotificationPermission() async {
    if (Platform.isAndroid) {
      final androidInfo = await DeviceInfoPlugin().androidInfo;
      
      if (androidInfo.version.sdkInt >= 33) {
        final status = await Permission.notification.request();
        return status.isGranted;
      }
    }
    
    return true;
  }
  
  /// Check if app can display over other apps (for floating widgets - future feature)
  Future<bool> canDrawOverlays() async {
    try {
      final bool result = await _channel.invokeMethod('canDrawOverlays');
      return result;
    } catch (e) {
      print('Error checking overlay permission: $e');
      return false;
    }
  }
  
  /// Get comprehensive permission status for onboarding
  Future<PermissionStatus> getPermissionStatus() async {
    final accessibilityEnabled = await isAccessibilityEnabled();
    final

---

## Phase 3: Data Management (Week 3)

### Day 13-14: Scroll Session Processing

#### Step 13: Create Scroll Calculator Service
**File: `lib/services/scroll_calculator.dart`**
```dart
import 'dart:math';
import '../models/scroll_event.dart';
import '../models/scroll_session.dart';
import '../config/constants.dart';

class ScrollCalculator {
  static final ScrollCalculator instance = ScrollCalculator._();
  ScrollCalculator._();
  
  /// Convert pixels to real-world distance (meters)
  double pixelsToMeters(double pixels) {
    return pixels / AppConstants.pixelsPerMeter;
  }
  
  /// Convert pixels to feet for US users
  double pixelsToFeet(double pixels) {
    return pixelsToMeters(pixels) * 3.28084;
  }
  
  /// Calculate total scroll distance using Pythagorean theorem
  double calculateScrollDistance(int scrollX, int scrollY) {
    return sqrt(pow(scrollX.abs(), 2) + pow(scrollY.abs(), 2));
  }
  
  /// Determine if events should be grouped into a session
  bool shouldGroupEvents(ScrollEvent event1, ScrollEvent event2) {
    // Same app check
    if (event1.packageName != event2.packageName) {
      return false;
    }
    
    // Time proximity check
    final timeDiff = event2.timestamp.difference(event1.timestamp);
    if (timeDiff > AppConstants.sessionTimeout) {
      return false;
    }
    
    // Distance threshold check (ignore very small scrolls)
    if (event2.scrollDistance < AppConstants.minScrollDistance) {
      return false;
    }
    
    return true;
  }
  
  /// Group scroll events into meaningful sessions
  List<ScrollSession> createSessionsFromEvents(List<ScrollEvent> events) {
    if (events.isEmpty) return [];
    
    // Sort events by timestamp
    events.sort((a, b) => a.timestamp.compareTo(b.timestamp));
    
    final sessions = <ScrollSession>[];
    List<ScrollEvent> currentSessionEvents = [events.first];
    
    for (int i = 1; i < events.length; i++) {
      if (shouldGroupEvents(currentSessionEvents.last, events[i])) {
        // Add to current session
        currentSessionEvents.add(events[i]);
      } else {
        // End current session and start new one
        if (currentSessionEvents.isNotEmpty) {
          sessions.add(_createSessionFromEvents(currentSessionEvents));
        }
        currentSessionEvents = [events[i]];
      }
    }
    
    // Don't forget the last session
    if (currentSessionEvents.isNotEmpty) {
      sessions.add(_createSessionFromEvents(currentSessionEvents));
    }
    
    return sessions;
  }
  
  /// Create a session from a group of events
  ScrollSession _createSessionFromEvents(List<ScrollEvent> events) {
    if (events.isEmpty) {
      throw ArgumentError('Cannot create session from empty events');
    }
    
    events.sort((a, b) => a.timestamp.compareTo(b.timestamp));
    
    final startTime = events.first.timestamp;
    final endTime = events.last.timestamp;
    final packageName = events.first.packageName;
    
    double totalDistance = 0;
    for (var event in events) {
      totalDistance += event.scrollDistance;
    }
    
    return ScrollSession(
      packageName: packageName,
      startTime: startTime,
      endTime: endTime,
      totalDistance: totalDistance,
      eventCount: events.length,
    );
  }
  
  /// Calculate scroll speed (pixels per second)
  double calculateScrollSpeed(List<ScrollEvent> events) {
    if (events.length < 2) return 0;
    
    events.sort((a, b) => a.timestamp.compareTo(b.timestamp));
    
    final totalTime = events.last.timestamp.difference(events.first.timestamp);
    if (totalTime.inSeconds == 0) return 0;
    
    double totalDistance = 0;
    for (var event in events) {
      totalDistance += event.scrollDistance;
    }
    
    return totalDistance / totalTime.inSeconds;
  }
  
  /// Get fun comparison for distance
  String getFunComparison(double pixels) {
    final meters = pixelsToMeters(pixels);
    
    // Find the best comparison from our constants
    double closestDistance = 0;
    String comparison = '';
    
    for (var entry in AppConstants.distanceComparisons.entries) {
      if (meters >= entry.key && entry.key > closestDistance) {
        closestDistance = entry.key;
        comparison = entry.value;
      }
    }
    
    if (comparison.isEmpty) {
      // For very small distances
      if (meters < 1) {
        final cm = (meters * 100).toStringAsFixed(0);
        return "You scrolled ${cm}cm - about the length of a pencil!";
      } else {
        return "You scrolled ${meters.toStringAsFixed(1)}m today!";
      }
    }
    
    return "You scrolled ${meters.toStringAsFixed(1)}m - that's like $comparison!";
  }
  
  /// Calculate daily scroll goal progress
  double calculateGoalProgress(double dailyDistance, double goalDistance) {
    if (goalDistance <= 0) return 0;
    return (dailyDistance / goalDistance).clamp(0.0, 1.0);
  }
  
  /// Detect scroll patterns (for future features)
  ScrollPattern analyzeScrollPattern(List<ScrollEvent> events) {
    if (events.isEmpty) return ScrollPattern.none;
    
    final speeds = events.map((e) => e.scrollDistance).toList();
    final avgSpeed = speeds.reduce((a, b) => a + b) / speeds.length;
    
    // Simple pattern detection
    if (avgSpeed > 1000) {
      return ScrollPattern.fastScrolling; // Doom scrolling
    } else if (avgSpeed < 100) {
      return ScrollPattern.slowBrowsing; // Reading/browsing
    } else {
      return ScrollPattern.normalScrolling;
    }
  }
}

enum ScrollPattern {
  none,
  slowBrowsing,
  normalScrolling,
  fastScrolling, // Potential doom scrolling
}
```

#### Step 14: Background Data Processing Service
**File: `lib/services/background_processor.dart`**
```dart
import 'dart:async';
import 'package:workmanager/workmanager.dart';
import 'database_service.dart';
import 'scroll_calculator.dart';
import '../models/scroll_event.dart';
import '../config/constants.dart';

class BackgroundProcessor {
  static final BackgroundProcessor instance = BackgroundProcessor._();
  BackgroundProcessor._();
  
  static const String sessionProcessingTask = 'session_processing';
  static const String dailyAggregationTask = 'daily_aggregation';
  static const String dataCleanupTask = 'data_cleanup';
  
  /// Initialize background processing
  Future<void> initialize() async {
    await Workmanager().initialize(
      callbackDispatcher,
      isInDebugMode: false, // Set to true during development
    );
    
    // Schedule periodic tasks
    await _schedulePeriodicTasks();
  }
  
  /// Schedule all background tasks
  Future<void> _schedulePeriodicTasks() async {
    // Process scroll events into sessions every 5 minutes
    await Workmanager().registerPeriodicTask(
      sessionProcessingTask,
      sessionProcessingTask,
      frequency: const Duration(minutes: 5),
      constraints: Constraints(
        networkType: NetworkType.not_required,
        requiresBatteryNotLow: true,
      ),
    );
    
    // Aggregate daily statistics once per day
    await Workmanager().registerPeriodicTask(
      dailyAggregationTask,
      dailyAggregationTask,
      frequency: const Duration(hours: 24),
      initialDelay: const Duration(hours: 1),
    );
    
    // Clean up old data weekly
    await Workmanager().registerPeriodicTask(
      dataCleanupTask,
      dataCleanupTask,
      frequency: const Duration(days: 7),
      initialDelay: const Duration(hours: 2),
    );
  }
  
  /// Process unprocessed scroll events into sessions
  Future<void> processScrollEventsIntoSessions() async {
    try {
      final db = DatabaseService.instance;
      
      // Get recent events that haven't been processed into sessions
      final recentEvents = await db.getScrollEvents(
        startTime: DateTime.now().subtract(const Duration(hours: 2)),
        limit: 1000, // Process in batches
      );
      
      if (recentEvents.isEmpty) return;
      
      // Group events into sessions
      final sessions = ScrollCalculator.instance.createSessionsFromEvents(recentEvents);
      
      // Insert sessions into database
      for (var session in sessions) {
        await db.insertScrollSession(session);
      }
      
      print('Processed ${recentEvents.length} events into ${sessions.length} sessions');
    } catch (e) {
      print('Error processing scroll events: $e');
    }
  }
  
  /// Aggregate daily statistics
  Future<void> aggregateDailyStatistics() async {
    try {
      final db = DatabaseService.instance;
      final yesterday = DateTime.now().subtract(const Duration(days: 1));
      
      // Check if we already have aggregated data for yesterday
      final existingStats = await db.getDailyStats(yesterday);
      if (existingStats != null) {
        print('Daily stats already exist for ${yesterday.toIso8601String()}');
        return;
      }
      
      // Calculate stats from raw events
      final stats = await db._calculateDailyStatsFromEvents(yesterday);
      if (stats != null) {
        // Store aggregated statistics
        await _storeDailyStats(stats);
        print('Aggregated daily stats for ${yesterday.toIso8601String()}');
      }
    } catch (e) {
      print('Error aggregating daily statistics: $e');
    }
  }
  
  /// Store daily statistics in database
  Future<void> _storeDailyStats(DailyStats stats) async {
    final db = await DatabaseService.instance.database;
    final dateString = '${stats.date.year}-${stats.date.month.toString().padLeft(2, '0')}-${stats.date.day.toString().padLeft(2, '0')}';
    
    // Insert daily aggregate
    await db.insert(
      'daily_stats',
      {
        'date': dateString,
        'total_distance': stats.totalDistance,
        'total_time_seconds': stats.totalTime.inSeconds,
        'session_count': stats.sessionCount,
        'top_app': stats.topApps.isNotEmpty ? stats.topApps.first.key : null,
      },
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
    
    // Insert per-app statistics
    for (var entry in stats.appDistances.entries) {
      await db.insert(
        'app_usage',
        {
          'date': dateString,
          'package_name': entry.key,
          'total_distance': entry.value,
          'total_time_seconds': 0, // TODO: Calculate from sessions
          'session_count': 0, // TODO: Calculate from sessions
        },
        conflictAlgorithm: ConflictAlgorithm.replace,
      );
    }
  }
  
  /// Clean up old data for privacy
  Future<void> performDataCleanup() async {
    try {
      await DatabaseService.instance.cleanupOldData(daysToKeep: 30);
      print('Data cleanup completed');
    } catch (e) {
      print('Error during data cleanup: $e');
    }
  }
}

/// Background task callback dispatcher
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    try {
      switch (task) {
        case BackgroundProcessor.sessionProcessingTask:
          await BackgroundProcessor.instance.processScrollEventsIntoSessions();
          break;
        case BackgroundProcessor.dailyAggregationTask:
          await BackgroundProcessor.instance.aggregateDailyStatistics();
          break;
        case BackgroundProcessor.dataCleanupTask:
          await BackgroundProcessor.instance.performDataCleanup();
          break;
        default:
          print('Unknown background task: $task');
          return false;
      }
      return true;
    } catch (e) {
      print('Background task $task failed: $e');
      return false;
    }
  });
}
```