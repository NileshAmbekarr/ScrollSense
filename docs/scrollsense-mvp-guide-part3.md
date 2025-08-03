# ScrollSense MVP Guide - Part 2 (Continued from Week 3)

## Phase 3: Data Management (Week 3) - Continued

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
    final notificationsEnabled = await isNotificationEnabled();
    
    if (accessibilityEnabled && notificationsEnabled) {
      return PermissionStatus.allGranted;
    } else if (accessibilityEnabled) {
      return PermissionStatus.accessibilityOnly;
    } else if (notificationsEnabled) {
      return PermissionStatus.notificationsOnly;
    } else {
      return PermissionStatus.noneGranted;
    }
  }
  
  /// Check if device is whitelisted for battery optimization
  Future<bool> isBatteryOptimizationDisabled() async {
    try {
      final bool result = await _channel.invokeMethod('isBatteryOptimizationDisabled');
      return result;
    } catch (e) {
      print('Error checking battery optimization: $e');
      return false;
    }
  }
  
  /// Request to disable battery optimization for better background performance
  Future<void> requestDisableBatteryOptimization() async {
    try {
      await _channel.invokeMethod('requestDisableBatteryOptimization');
    } catch (e) {
      print('Error requesting battery optimization disable: $e');
    }
  }
}

enum PermissionStatus {
  noneGranted,
  accessibilityOnly,
  notificationsOnly,
  allGranted,
}
```

**Why this permission approach?**
- **Progressive permissions**: Request only what's needed, when needed
- **Clear explanations**: Users understand why each permission is required
- **Fallback handling**: App works even with limited permissions
- **Future-proof**: Handles different Android versions gracefully

#### Step 16: Comprehensive Onboarding Screen
**File: `lib/screens/onboarding_screen.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../config/constants.dart';
import '../utils/permissions.dart';
import '../widgets/onboarding_page.dart';

class OnboardingScreen extends StatefulWidget {
  const OnboardingScreen({Key? key}) : super(key: key);

  @override
  State<OnboardingScreen> createState() => _OnboardingScreenState();
}

class _OnboardingScreenState extends State<OnboardingScreen> {
  final PageController _pageController = PageController();
  int _currentPage = 0;
  bool _isLoading = false;
  
  final List<OnboardingPageData> _pages = [
    OnboardingPageData(
      title: 'Welcome to ScrollSense',
      subtitle: 'Discover how much you really scroll',
      description: 'Ever wondered how far you scroll each day? ScrollSense tracks your scrolling across all apps and shows you fascinating insights about your digital habits.',
      icon: Icons.swap_vert_rounded,
      color: Colors.blue,
    ),
    OnboardingPageData(
      title: 'Your Privacy Matters',
      subtitle: 'All data stays on your device',
      description: 'ScrollSense is completely privacy-first. We never send your data anywhere - everything stays safely on your phone. You can delete all data anytime.',
      icon: Icons.security_rounded,
      color: Colors.green,
    ),
    OnboardingPageData(
      title: 'Fun Comparisons',
      subtitle: 'Like climbing the Eiffel Tower!',
      description: 'We convert your scroll distance into fun real-world comparisons. You might be surprised to learn you scroll the height of a skyscraper each day!',
      icon: Icons.emoji_events_rounded,
      color: Colors.orange,
    ),
    OnboardingPageData(
      title: 'Enable Accessibility',
      subtitle: 'Required to track scrolling',
      description: 'To track scrolling across all your apps, ScrollSense needs accessibility permission. This is the only way Android allows apps to detect scrolling system-wide.',
      icon: Icons.accessibility_rounded,
      color: Colors.purple,
      isPermissionPage: true,
      permissionType: PermissionType.accessibility,
    ),
    OnboardingPageData(
      title: 'Enable Notifications',
      subtitle: 'Get gentle scroll break reminders',
      description: 'Optional: Get friendly notifications to take breaks from scrolling. You can customize or disable these anytime in settings.',
      icon: Icons.notifications_rounded,
      color: Colors.teal,
      isPermissionPage: true,
      permissionType: PermissionType.notifications,
    ),
  ];
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            // Progress indicator
            Container(
              padding: const EdgeInsets.all(20),
              child: Row(
                children: [
                  // Skip button
                  TextButton(
                    onPressed: _currentPage < _pages.length - 2 ? _skipOnboarding : null,
                    child: Text(
                      'Skip',
                      style: TextStyle(
                        color: _currentPage < _pages.length - 2 
                            ? Colors.grey[600] 
                            : Colors.transparent,
                      ),
                    ),
                  ),
                  const Spacer(),
                  // Page indicators
                  Row(
                    children: List.generate(
                      _pages.length,
                      (index) => Container(
                        margin: const EdgeInsets.symmetric(horizontal: 4),
                        width: 8,
                        height: 8,
                        decoration: BoxDecoration(
                          color: index == _currentPage 
                              ? Theme.of(context).primaryColor 
                              : Colors.grey[300],
                          borderRadius: BorderRadius.circular(4),
                        ),
                      ),
                    ),
                  ),
                ],
              ),
            ),
            
            // Pages
            Expanded(
              child: PageView.builder(
                controller: _pageController,
                onPageChanged: (index) {
                  setState(() {
                    _currentPage = index;
                  });
                },
                itemCount: _pages.length,
                itemBuilder: (context, index) {
                  return OnboardingPage(
                    data: _pages[index],
                    onPermissionAction: _handlePermissionAction,
                  );
                },
              ),
            ),
            
            // Navigation buttons
            Container(
              padding: const EdgeInsets.all(20),
              child: Row(
                children: [
                  // Back button
                  if (_currentPage > 0)
                    TextButton(
                      onPressed: () {
                        _pageController.previousPage(
                          duration: const Duration(milliseconds: 300),
                          curve: Curves.easeInOut,
                        );
                      },
                      child: const Text('Back'),
                    )
                  else
                    const SizedBox(width: 80),
                  
                  const Spacer(),
                  
                  // Next/Finish button
                  _isLoading
                      ? const CircularProgressIndicator()
                      : ElevatedButton(
                          onPressed: _handleNextButton,
                          style: ElevatedButton.styleFrom(
                            padding: const EdgeInsets.symmetric(
                              horizontal: 32,
                              vertical: 12,
                            ),
                          ),
                          child: Text(
                            _currentPage == _pages.length - 1 
                                ? 'Get Started' 
                                : 'Next',
                          ),
                        ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Future<void> _handleNextButton() async {
    if (_currentPage == _pages.length - 1) {
      // Last page - finish onboarding
      await _finishOnboarding();
    } else {
      // Move to next page
      _pageController.nextPage(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
      );
    }
  }
  
  Future<void> _handlePermissionAction(PermissionType type) async {
    setState(() {
      _isLoading = true;
    });
    
    try {
      switch (type) {
        case PermissionType.accessibility:
          await PermissionManager.instance.openAccessibilitySettings();
          break;
        case PermissionType.notifications:
          await PermissionManager.instance.requestNotificationPermission();
          break;
      }
      
      // Give user time to enable permission, then check status
      await Future.delayed(const Duration(seconds: 2));
      
      // Auto-advance if permission was granted
      final status = await PermissionManager.instance.getPermissionStatus();
      if (type == PermissionType.accessibility && 
          await PermissionManager.instance.isAccessibilityEnabled()) {
        _pageController.nextPage(
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeInOut,
        );
      }
    } catch (e) {
      print('Error handling permission: $e');
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }
  
  Future<void> _skipOnboarding() async {
    // Jump to last page (which handles completion)
    _pageController.animateToPage(
      _pages.length - 1,
      duration: const Duration(milliseconds: 500),
      curve: Curves.easeInOut,
    );
  }
  
  Future<void> _finishOnboarding() async {
    setState(() {
      _isLoading = true;
    });
    
    try {
      // Mark onboarding as complete
      final prefs = await SharedPreferences.getInstance();
      await prefs.setBool(AppConstants.keyOnboardingComplete, true);
      
      // Set default settings
      await prefs.setBool(AppConstants.keyNotificationsEnabled, true);
      await prefs.setInt(AppConstants.keyBreakInterval, 30);
      await prefs.setDouble(AppConstants.keyDailyGoal, 1000.0); // 1km default goal
      
      // Navigate to home screen
      if (mounted) {
        Navigator.pushReplacementNamed(context, '/home');
      }
    } catch (e) {
      print('Error finishing onboarding: $e');
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('Error completing setup. Please try again.'),
        ),
      );
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }
  
  @override
  void dispose() {
    _pageController.dispose();
    super.dispose();
  }
}

class OnboardingPageData {
  final String title;
  final String subtitle;
  final String description;
  final IconData icon;
  final Color color;
  final bool isPermissionPage;
  final PermissionType? permissionType;
  
  OnboardingPageData({
    required this.title,
    required this.subtitle,
    required this.description,
    required this.icon,
    required this.color,
    this.isPermissionPage = false,
    this.permissionType,
  });
}

enum PermissionType {
  accessibility,
  notifications,
}
```

#### Step 17: Reusable Onboarding Page Widget
**File: `lib/widgets/onboarding_page.dart`**
```dart
import 'package:flutter/material.dart';
import '../screens/onboarding_screen.dart';
import '../utils/permissions.dart';

class OnboardingPage extends StatefulWidget {
  final OnboardingPageData data;
  final Function(PermissionType)? onPermissionAction;
  
  const OnboardingPage({
    Key? key,
    required this.data,
    this.onPermissionAction,
  }) : super(key: key);

  @override
  State<OnboardingPage> createState() => _OnboardingPageState();
}

class _OnboardingPageState extends State<OnboardingPage> 
    with TickerProviderStateMixin {
  late AnimationController _animationController;
  late Animation<double> _fadeAnimation;
  late Animation<Offset> _slideAnimation;
  
  bool _permissionGranted = false;
  bool _checkingPermission = false;
  
  @override
  void initState() {
    super.initState();
    
    _animationController = AnimationController(
      duration: const Duration(milliseconds: 800),
      vsync: this,
    );
    
    _fadeAnimation = Tween<double>(
      begin: 0.0,
      end: 1.0,
    ).animate(CurvedAnimation(
      parent: _animationController,
      curve: Curves.easeInOut,
    ));
    
    _slideAnimation = Tween<Offset>(
      begin: const Offset(0, 0.3),
      end: Offset.zero,
    ).animate(CurvedAnimation(
      parent: _animationController,
      curve: Curves.easeOutCubic,
    ));
    
    _animationController.forward();
    
    // Check permission status for permission pages
    if (widget.data.isPermissionPage) {
      _checkPermissionStatus();
    }
  }
  
  Future<void> _checkPermissionStatus() async {
    if (!widget.data.isPermissionPage) return;
    
    setState(() {
      _checkingPermission = true;
    });
    
    bool granted = false;
    
    switch (widget.data.permissionType) {
      case PermissionType.accessibility:
        granted = await PermissionManager.instance.isAccessibilityEnabled();
        break;
      case PermissionType.notifications:
        granted = await PermissionManager.instance.isNotificationEnabled();
        break;
      case null:
        break;
    }
    
    setState(() {
      _permissionGranted = granted;
      _checkingPermission = false;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: _fadeAnimation,
      child: SlideTransition(
        position: _slideAnimation,
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 32, vertical: 20),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              // Icon
              Container(
                width: 120,
                height: 120,
                decoration: BoxDecoration(
                  color: widget.data.color.withOpacity(0.1),
                  borderRadius: BorderRadius.circular(60),
                ),
                child: Icon(
                  widget.data.icon,
                  size: 60,
                  color: widget.data.color,
                ),
              ),
              
              const SizedBox(height: 40),
              
              // Title
              Text(
                widget.data.title,
                style: const TextStyle(
                  fontSize: 28,
                  fontWeight: FontWeight.bold,
                ),
                textAlign: TextAlign.center,
              ),
              
              const SizedBox(height: 16),
              
              // Subtitle
              Text(
                widget.data.subtitle,
                style: TextStyle(
                  fontSize: 18,
                  color: widget.data.color,
                  fontWeight: FontWeight.w500,
                ),
                textAlign: TextAlign.center,
              ),
              
              const SizedBox(height: 24),
              
              // Description
              Text(
                widget.data.description,
                style: TextStyle(
                  fontSize: 16,
                  color: Colors.grey[700],
                  height: 1.5,
                ),
                textAlign: TextAlign.center,
              ),
              
              const SizedBox(height: 40),
              
              // Permission-specific content
              if (widget.data.isPermissionPage) ...[
                _buildPermissionSection(),
              ],
            ],
          ),
        ),
      ),
    );
  }
  
  Widget _buildPermissionSection() {
    if (_checkingPermission) {
      return const Column(
        children: [
          CircularProgressIndicator(),
          SizedBox(height: 16),
          Text('Checking permission status...'),
        ],
      );
    }
    
    if (_permissionGranted) {
      return Container(
        padding: const EdgeInsets.all(16),
        decoration: BoxDecoration(
          color: Colors.green.withOpacity(0.1),
          borderRadius: BorderRadius.circular(12),
          border: Border.all(color: Colors.green.withOpacity(0.3)),
        ),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.check_circle,
              color: Colors.green[700],
              size: 24,
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Text(
                'Permission granted! You\'re all set.',
                style: TextStyle(
                  color: Colors.green[700],
                  fontWeight: FontWeight.w500,
                ),
              ),
            ),
          ],
        ),
      );
    }
    
    // Permission not granted - show action button
    return Column(
      children: [
        Container(
          padding: const EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: Colors.orange.withOpacity(0.1),
            borderRadius: BorderRadius.circular(12),
            border: Border.all(color: Colors.orange.withOpacity(0.3)),
          ),
          child: Row(
            children: [
              Icon(
                Icons.warning_amber_rounded,
                color: Colors.orange[700],
                size: 24,
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Text(
                  _getPermissionWarningText(),
                  style: TextStyle(
                    color: Colors.orange[700],
                    fontWeight: FontWeight.w500,
                  ),
                ),
              ),
            ],
          ),
        ),
        
        const SizedBox(height: 20),
        
        ElevatedButton.icon(
          onPressed: () async {
            if (widget.onPermissionAction != null && 
                widget.data.permissionType != null) {
              await widget.onPermissionAction!(widget.data.permissionType!);
              // Recheck permission status after action
              await Future.delayed(const Duration(milliseconds: 500));
              _checkPermissionStatus();
            }
          },
          icon: Icon(_getPermissionButtonIcon()),
          label: Text(_getPermissionButtonText()),
          style: ElevatedButton.styleFrom(
            backgroundColor: widget.data.color,
            foregroundColor: Colors.white,
            padding: const EdgeInsets.symmetric(
              horizontal: 24,
              vertical: 12,
            ),
          ),
        ),
        
        const SizedBox(height: 16),
        
        Text(
          _getPermissionHelpText(),
          style: TextStyle(
            fontSize: 14,
            color: Colors.grey[600],
          ),
          textAlign: TextAlign.center,
        ),
      ],
    );
  }
  
  String _getPermissionWarningText() {
    switch (widget.data.permissionType) {
      case PermissionType.accessibility:
        return 'Accessibility permission is required for ScrollSense to work.';
      case PermissionType.notifications:
        return 'Enable notifications for helpful scroll break reminders.';
      case null:
        return '';
    }
  }
  
  IconData _getPermissionButtonIcon() {
    switch (widget.data.permissionType) {
      case PermissionType.accessibility:
        return Icons.settings_accessibility;
      case PermissionType.notifications:
        return Icons.notifications_active;
      case null:
        return Icons.settings;
    }
  }
  
  String _getPermissionButtonText() {
    switch (widget.data.permissionType) {
      case PermissionType.accessibility:
        return 'Open Settings';
      case PermissionType.notifications:
        return 'Enable Notifications';
      case null:
        return 'Enable';
    }
  }
  
  String _getPermissionHelpText() {
    switch (widget.data.permissionType) {
      case PermissionType.accessibility:
        return 'Look for "ScrollSense" in the accessibility services list and toggle it on.';
      case PermissionType.notifications:
        return 'You can always change notification settings later in the app.';
      case null:
        return '';
    }
  }
  
  @override
  void dispose() {
    _animationController.dispose();
    super.dispose();
  }
}
```

**Why this onboarding approach?**
- **Educational**: Users understand what the app does before granting permissions
- **Privacy-transparent**: Clearly explains data stays on device
- **Progressive disclosure**: Introduces concepts gradually
- **Permission context**: Explains why each permission is needed
- **User-friendly**: Beautiful animations and clear visual feedback
- **Flexible**: Easy to add or remove onboarding steps

---

## Phase 4: UI Development (Week 4)

### Day 17-18: Home Screen and Dashboard

#### Step 18: Main Home Screen with Real-time Stats
**File: `lib/screens/home_screen.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../services/database_service.dart';
import '../services/accessibility_service.dart';
import '../services/scroll_calculator.dart';
import '../models/daily_stats.dart';
import '../models/scroll_event.dart';
import '../widgets/stats_card.dart';
import '../widgets/scroll_progress_ring.dart';
import '../widgets/today_summary_card.dart';
import '../widgets/quick_stats_row.dart';
import '../utils/formatters.dart';
import 'stats_screen.dart';
import 'settings_screen.dart';
import 'dart:async';

class HomeScreen extends StatefulWidget {
  const HomeScreen({Key? key}) : super(key: key);

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> with WidgetsBindingObserver {
  final AccessibilityService _accessibilityService = AccessibilityService.instance;
  final DatabaseService _databaseService = DatabaseService.instance;
  final ScrollCalculator _scrollCalculator = ScrollCalculator.instance;
  
  DailyStats? _todayStats;
  bool _isLoading = true;
  bool _accessibilityEnabled = false;
  StreamSubscription<ScrollEvent>? _scrollEventSubscription;
  double _realTimeDistance = 0;
  
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _initializeData();
    _startListeningToScrollEvents();
  }
  
  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _scrollEventSubscription?.cancel();
    _accessibilityService.stopMonitoring();
    super.dispose();
  }
  
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      // Refresh data when app comes back to foreground
      _checkAccessibilityStatus();
      _loadTodayStats();
    }
  }
  
  Future<void> _initializeData() async {
    await _checkAccessibilityStatus();
    await _loadTodayStats();
    setState(() {
      _isLoading = false;
    });
  }
  
  Future<void> _checkAccessibilityStatus() async {
    final enabled = await _accessibilityService.isAccessibilityEnabled();
    setState(() {
      _accessibilityEnabled = enabled;
    });
    
    if (enabled) {
      _accessibilityService.startMonitoring();
    }
  }
  
  Future<void> _loadTodayStats() async {
    try {
      final stats = await _databaseService.getTodayStats();
      setState(() {
        _todayStats = stats;
        _realTimeDistance = stats?.totalDistance ?? 0;
      });
    } catch (e) {
      print('Error loading today stats: $e');
    }
  }
  
  void _startListeningToScrollEvents() {
    _scrollEventSubscription = _accessibilityService.scrollEventStream.listen(
      (event) {
        // Update real-time distance
        setState(() {
          _realTimeDistance += event.scrollDistance;
        });
        
        // Store event in database
        _databaseService.insertScrollEvent(event);
      },
      onError: (error) {
        print('Error in scroll event stream: $error');
      },
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey[50],
      appBar: AppBar(
        title: const Text(
          'ScrollSense',
          style: TextStyle(fontWeight: FontWeight.bold),
        ),
        backgroundColor: Colors.transparent,
        elevation: 0,
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(builder: (context) => const SettingsScreen()),
              );
            },
          ),
        ],
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : _buildBody(),
    );
  }
  
  Widget _buildBody() {
    if (!_accessibilityEnabled) {
      return _buildAccessibilityRequiredView();
    }
    
    return RefreshIndicator(
      onRefresh: _loadTodayStats,
      child: SingleChildScrollView(
        physics: const AlwaysScrollableScrollPhysics(),
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Today's progress ring
            ScrollProgressRing(
              currentDistance: _realTimeDistance,
              goalDistance: 1000, // TODO: Get from settings
              size: 200,
            ),
            
            const SizedBox(height: 24),
            
            // Today's summary
            TodaySummaryCard(
              stats: _todayStats,
              realTimeDistance: _realTimeDistance,
            ),
            
            const SizedBox(height: 16),
            
            // Quick stats row
            QuickStatsRow(
              stats: _todayStats,
              realTimeDistance: _realTimeDistance,
            ),
            
            const SizedBox(height: 24),
            
            // Section header
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                const Text(
                  'Your Stats',
                  style: TextStyle(
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                TextButton(
                  onPressed: () {
                    Navigator.push(
                      context,
                      MaterialPageRoute(builder: (context) => const StatsScreen()),
                    );
                  },
                  child: const Text('View All'),
                ),
              ],
            ),
            
            const SizedBox(height: 16),
            
            // Stats cards
            _buildStatsCards(),
            
            const SizedBox(height: 32),
            
            // Fun fact card
            if (_todayStats != null) _buildFunFactCard(),
          ],
        ),
      ),
    );
  }
  
  Widget _buildAccessibilityRequiredView() {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.accessibility_rounded,
              size: 80,
              color: Colors.grey[400],
            ),
            const SizedBox(height: 24),
            const Text(
              'Accessibility Permission Required',
              style: TextStyle(
                fontSize: 24,
                fontWeight: FontWeight.bold,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 16),
            Text(
              'ScrollSense needs accessibility permission to track your scrolling across all apps. This is completely private - all data stays on your device.',
              style: TextStyle(
                fontSize: 16,
                color: Colors.grey[600],
                height: 1.5,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 32),
            ElevatedButton.icon(
              onPressed: () async {
                await _accessibilityService.openAccessibilitySettings();