# ScrollSense MVP Guide - Part 4 (Statistics & Settings)

## Phase 4: UI Development (Week 4) - Continued

### Day 19-20: Statistics Screen and Data Visualization

#### Step 24: Comprehensive Statistics Screen
**File: `lib/screens/stats_screen.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:fl_chart/fl_chart.dart';
import '../services/database_service.dart';
import '../services/scroll_calculator.dart';
import '../models/daily_stats.dart';
import '../widgets/scroll_chart.dart';
import '../widgets/app_usage_list.dart';
import '../utils/formatters.dart';
import 'dart:async';

class StatsScreen extends StatefulWidget {
  const StatsScreen({Key? key}) : super(key: key);

  @override
  State<StatsScreen> createState() => _StatsScreenState();
}

class _StatsScreenState extends State<StatsScreen> 
    with SingleTickerProviderStateMixin {
  
  late TabController _tabController;
  final DatabaseService _databaseService = DatabaseService.instance;
  
  List<DailyStats> _weeklyStats = [];
  DailyStats? _todayStats;
  bool _isLoading = true;
  
  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 3, vsync: this);
    _loadStatistics();
  }
  
  Future<void> _loadStatistics() async {
    setState(() {
      _isLoading = true;
    });
    
    try {
      final [weeklyStats, todayStats] = await Future.wait([
        _databaseService.getWeeklyStats(),
        _databaseService.getTodayStats(),
      ]);
      
      setState(() {
        _weeklyStats = weeklyStats as List<DailyStats>;
        _todayStats = todayStats as DailyStats?;
        _isLoading = false;
      });
    } catch (e) {
      print('Error loading statistics: $e');
      setState(() {
        _isLoading = false;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey[50],
      appBar: AppBar(
        title: const Text(
          'Your Statistics',
          style: TextStyle(fontWeight: FontWeight.bold),
        ),
        backgroundColor: Colors.transparent,
        elevation: 0,
        bottom: TabBar(
          controller: _tabController,
          tabs: const [
            Tab(text: 'Today', icon: Icon(Icons.today)),
            Tab(text: 'Week', icon: Icon(Icons.calendar_view_week)),
            Tab(text: 'Apps', icon: Icon(Icons.apps)),
          ],
        ),
        actions: [
          IconButton(
            icon: const Icon(Icons.share),
            onPressed: _shareStats,
          ),
        ],
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : TabBarView(
              controller: _tabController,
              children: [
                _buildTodayTab(),
                _buildWeekTab(),
                _buildAppsTab(),
              ],
            ),
    );
  }
  
  Widget _buildTodayTab() {
    if (_todayStats == null) {
      return _buildEmptyState('No data for today yet', 'Start scrolling to see your stats!');
    }
    
    return RefreshIndicator(
      onRefresh: _loadStatistics,
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Today's overview card
            _buildTodayOverviewCard(),
            
            const SizedBox(height: 24),
            
            // Hourly breakdown chart
            _buildSectionHeader('Activity Throughout Day'),
            const SizedBox(height: 12),
            ScrollChart(
              data: _generateHourlyData(),
              title: 'Scroll Activity by Hour',
              type: ChartType.line,
            ),
            
            const SizedBox(height: 24),
            
            // Speed analysis
            _buildSectionHeader('Scroll Speed Analysis'),
            const SizedBox(height: 12),
            _buildSpeedAnalysisCard(),
            
            const SizedBox(height: 24),
            
            // Session breakdown
            _buildSectionHeader('Today\'s Sessions'),
            const SizedBox(height: 12),
            _buildSessionsList(),
          ],
        ),
      ),
    );
  }
  
  Widget _buildWeekTab() {
    if (_weeklyStats.isEmpty) {
      return _buildEmptyState('No weekly data yet', 'Use the app for a few days to see trends!');
    }
    
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Weekly summary cards
          _buildWeeklySummaryCards(),
          
          const SizedBox(height: 24),
          
          // Weekly trend chart
          _buildSectionHeader('Weekly Trend'),
          const SizedBox(height: 12),
          ScrollChart(
            data: _weeklyStats.map((stat) => ChartData(
              x: stat.date.day.toDouble(),
              y: ScrollCalculator.instance.pixelsToMeters(stat.totalDistance),
              label: '${stat.date.day}',
            )).toList(),
            title: 'Distance Scrolled (meters)',
            type: ChartType.bar,
          ),
          
          const SizedBox(height: 24),
          
          // Day-by-day breakdown
          _buildSectionHeader('Daily Breakdown'),
          const SizedBox(height: 12),
          _buildWeeklyBreakdown(),
          
          const SizedBox(height: 24),
          
          // Weekly insights
          _buildWeeklyInsights(),
        ],
      ),
    );
  }
  
  Widget _buildAppsTab() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // App usage summary
          _buildSectionHeader('App Usage Today'),
          const SizedBox(height: 12),
          
          if (_todayStats?.appDistances.isNotEmpty ?? false) ...[
            AppUsageList(
              appUsage: _todayStats!.appDistances,
              showDetails: true,
            ),
            
            const SizedBox(height: 24),
            
            // Top apps chart
            _buildSectionHeader('Top Apps Distribution'),
            const SizedBox(height: 12),
            ScrollChart(
              data: _todayStats!.appDistances.entries.take(5).map((entry) => 
                ChartData(
                  x: 0,
                  y: ScrollCalculator.instance.pixelsToMeters(entry.value),
                  label: AppFormatters.getAppDisplayName(entry.key),
                )
              ).toList(),
              title: 'Distance by App (meters)',
              type: ChartType.pie,
            ),
          ] else ...[
            _buildEmptyState('No app data yet', 'Start scrolling in different apps!'),
          ],
        ],
      ),
    );
  }
  
  Widget _buildTodayOverviewCard() {
    final totalMeters = ScrollCalculator.instance.pixelsToMeters(_todayStats!.totalDistance);
    final comparison = ScrollCalculator.instance.getFunComparison(_todayStats!.totalDistance);
    
    return Card(
      elevation: 4,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      child: Container(
        padding: const EdgeInsets.all(20),
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(16),
          gradient: LinearGradient(
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
            colors: [Colors.blue[400]!, Colors.purple[400]!],
          ),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              'Today\'s Achievement',
              style: TextStyle(
                color: Colors.white,
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        '${totalMeters.toStringAsFixed(0)}m',
                        style: const TextStyle(
                          color: Colors.white,
                          fontSize: 32,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(
                        comparison,
                        style: const TextStyle(
                          color: Colors.white70,
                          fontSize: 14,
                        ),
                      ),
                    ],
                  ),
                ),
                Container(
                  padding: const EdgeInsets.all(12),
                  decoration: BoxDecoration(
                    color: Colors.white.withOpacity(0.2),
                    borderRadius: BorderRadius.circular(12),
                  ),
                  child: const Icon(
                    Icons.emoji_events,
                    color: Colors.white,
                    size: 32,
                  ),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildWeeklySummaryCards() {
    final totalDistance = _weeklyStats.fold<double>(
      0, (sum, stat) => sum + stat.totalDistance,
    );
    final totalSessions = _weeklyStats.fold<int>(
      0, (sum, stat) => sum + stat.sessionCount,
    );
    final avgDaily = totalDistance / _weeklyStats.length;
    
    return Row(
      children: [
        Expanded(
          child: _buildSummaryCard(
            'Total Distance',
            AppFormatters.formatDistance(
              ScrollCalculator.instance.pixelsToMeters(totalDistance),
            ),
            Icons.straighten,
            Colors.blue,
          ),
        ),
        const SizedBox(width: 12),
        Expanded(
          child: _buildSummaryCard(
            'Total Sessions',
            totalSessions.toString(),
            Icons.timeline,
            Colors.green,
          ),
        ),
        const SizedBox(width: 12),
        Expanded(
          child: _buildSummaryCard(
            'Daily Average',
            AppFormatters.formatDistance(
              ScrollCalculator.instance.pixelsToMeters(avgDaily),
            ),
            Icons.trending_up,
            Colors.orange,
          ),
        ),
      ],
    );
  }
  
  Widget _buildSummaryCard(String title, String value, IconData icon, Color color) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Icon(icon, color: color, size: 24),
            const SizedBox(height: 8),
            Text(
              value,
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
                color: color,
              ),
            ),
            const SizedBox(height: 4),
            Text(
              title,
              style: TextStyle(
                fontSize: 12,
                color: Colors.grey[600],
              ),
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildSpeedAnalysisCard() {
    // Calculate speed metrics from today's data
    final avgSpeed = _calculateAverageSpeed();
    final peakSpeed = _calculatePeakSpeed();
    
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(Icons.speed, color: Colors.teal[600]),
                const SizedBox(width: 8),
                const Text(
                  'Scroll Speed Analysis',
                  style: TextStyle(fontWeight: FontWeight.w600),
                ),
              ],
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                Expanded(
                  child: _buildSpeedMetric('Average', avgSpeed, Colors.teal),
                ),
                Expanded(
                  child: _buildSpeedMetric('Peak', peakSpeed, Colors.red),
                ),
              ],
            ),
            const SizedBox(height: 12),
            Text(
              _getSpeedInsight(avgSpeed),
              style: TextStyle(
                fontSize: 14,
                color: Colors.grey[700],
                fontStyle: FontStyle.italic,
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildSpeedMetric(String label, double speed, Color color) {
    return Container(
      padding: const EdgeInsets.all(12),
      margin: const EdgeInsets.symmetric(horizontal: 4),
      decoration: BoxDecoration(
        color: color.withOpacity(0.1),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Column(
        children: [
          Text(
            '${speed.toStringAsFixed(1)} m/s',
            style: TextStyle(
              fontSize: 16,
              fontWeight: FontWeight.bold,
              color: color,
            ),
          ),
          Text(
            label,
            style: TextStyle(
              fontSize: 12,
              color: Colors.grey[600],
            ),
          ),
        ],
      ),
    );
  }
  
  Widget _buildSessionsList() {
    // This would show individual scroll sessions from today
    // For MVP, we'll show a simplified version
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(Icons.list_alt, color: Colors.indigo[600]),
                const SizedBox(width: 8),
                const Text(
                  'Session Summary',
                  style: TextStyle(fontWeight: FontWeight.w600),
                ),
              ],
            ),
            const SizedBox(height: 16),
            Text(
              'You had ${_todayStats?.sessionCount ?? 0} scroll sessions today.',
              style: const TextStyle(fontSize: 16),
            ),
            const SizedBox(height: 8),
            if ((_todayStats?.sessionCount ?? 0) > 0) ...[
              Text(
                'Average session: ${(_todayStats!.totalDistance / _todayStats!.sessionCount / 17700).toStringAsFixed(1)}m',
                style: TextStyle(
                  fontSize: 14,
                  color: Colors.grey[700],
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }
  
  Widget _buildWeeklyBreakdown() {
    return Column(
      children: _weeklyStats.map((stat) {
        final meters = ScrollCalculator.instance.pixelsToMeters(stat.totalDistance);
        final dayName = _getDayName(stat.date.weekday);
        
        return Card(
          margin: const EdgeInsets.only(bottom: 8),
          child: ListTile(
            leading: CircleAvatar(
              backgroundColor: _getDayColor(stat.date.weekday),
              child: Text(
                dayName.substring(0, 1),
                style: const TextStyle(
                  color: Colors.white,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
            title: Text(dayName),
            subtitle: Text('${stat.sessionCount} sessions'),
            trailing: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Text(
                  '${meters.toStringAsFixed(0)}m',
                  style: const TextStyle(
                    fontWeight: FontWeight.bold,
                    fontSize: 16,
                  ),
                ),
                Text(
                  AppFormatters.formatDuration(stat.totalTime),
                  style: TextStyle(
                    fontSize: 12,
                    color: Colors.grey[600],
                  ),
                ),
              ],
            ),
          ),
        );
      }).toList(),
    );
  }
  
  Widget _buildWeeklyInsights() {
    final insights = _generateWeeklyInsights();
    
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(Icons.lightbulb_outline, color: Colors.amber[600]),
                const SizedBox(width: 8),
                const Text(
                  'Weekly Insights',
                  style: TextStyle(fontWeight: FontWeight.w600),
                ),
              ],
            ),
            const SizedBox(height: 16),
            ...insights.map((insight) => Padding(
              padding: const EdgeInsets.only(bottom: 8),
              child: Row(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text('• ', style: TextStyle(color: Colors.amber[600])),
                  Expanded(
                    child: Text(
                      insight,
                      style: const TextStyle(fontSize: 14, height: 1.4),
                    ),
                  ),
                ],
              ),
            )),
          ],
        ),
      ),
    );
  }
  
  Widget _buildSectionHeader(String title) {
    return Text(
      title,
      style: const TextStyle(
        fontSize: 18,
        fontWeight: FontWeight.bold,
        color: Colors.black87,
      ),
    );
  }
  
  Widget _buildEmptyState(String title, String message) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.timeline,
              size: 64,
              color: Colors.grey[400],
            ),
            const SizedBox(height: 16),
            Text(
              title,
              style: const TextStyle(
                fontSize: 20,
                fontWeight: FontWeight.w600,
                color: Colors.grey,
              ),
            ),
            const SizedBox(height: 8),
            Text(
              message,
              style: TextStyle(
                fontSize: 14,
                color: Colors.grey[600],
              ),
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }
  
  // Helper methods
  List<ChartData> _generateHourlyData() {
    // Generate sample hourly data - in production, calculate from actual events
    return List.generate(24, (hour) {
      final activity = _getHourlyActivity(hour);
      return ChartData(
        x: hour.toDouble(),
        y: activity,
        label: '${hour.toString().padLeft(2, '0')}:00',
      );
    });
  }
  
  double _getHourlyActivity(int hour) {
    // Sample activity pattern - replace with real data
    if (hour >= 8 && hour <= 10) return 150; // Morning peak
    if (hour >= 12 && hour <= 14) return 120; // Lunch break
    if (hour >= 18 && hour <= 22) return 200; // Evening peak
    if (hour >= 23 || hour <= 6) return 20; // Night/early morning
    return 80; // Regular activity
  }
  
  double _calculateAverageSpeed() {
    if (_todayStats?.totalTime.inSeconds == 0) return 0;
    return ScrollCalculator.instance.pixelsToMeters(_todayStats!.totalDistance) / 
           _todayStats!.totalTime.inSeconds;
  }
  
  double _calculatePeakSpeed() {
    // In production, calculate from individual events
    return _calculateAverageSpeed() * 2.5; // Estimate
  }
  
  String _getSpeedInsight(double avgSpeed) {
    if (avgSpeed > 5) {
      return 'You\'re a fast scroller! Consider taking breaks to avoid fatigue.';
    } else if (avgSpeed > 2) {
      return 'You scroll at a moderate pace - perfect for reading content.';
    } else {
      return 'You take your time scrolling - great for thoughtful reading!';
    }
  }
  
  String _getDayName(int weekday) {
    const days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];
    return days[weekday - 1];
  }
  
  Color _getDayColor(int weekday) {
    const colors = [
      Colors.red, Colors.orange, Colors.yellow, 
      Colors.green, Colors.blue, Colors.indigo, Colors.purple,
    ];
    return colors[weekday - 1];
  }
  
  List<String> _generateWeeklyInsights() {
    final insights = <String>[];
    
    if (_weeklyStats.isEmpty) return insights;
    
    // Find most active day
    final mostActiveDay = _weeklyStats.reduce((a, b) => 
      a.totalDistance > b.totalDistance ? a : b);
    insights.add('Your most active day was ${_getDayName(mostActiveDay.date.weekday)} '
        'with ${ScrollCalculator.instance.pixelsToMeters(mostActiveDay.totalDistance).toStringAsFixed(0)}m scrolled.');
    
    // Calculate trend
    if (_weeklyStats.length >= 2) {
      final firstHalf = _weeklyStats.take(_weeklyStats.length ~/ 2)
          .fold<double>(0, (sum, stat) => sum + stat.totalDistance);
      final secondHalf = _weeklyStats.skip(_weeklyStats.length ~/ 2)
          .fold<double>(0, (sum, stat) => sum + stat.totalDistance);
      
      if (secondHalf > firstHalf) {
        insights.add('Your scrolling activity increased towards the end of the week.');
      } else {
        insights.add('You scrolled less towards the end of the week - great self-control!');
      }
    }
    
    return insights;
  }
  
  void _shareStats() {
    // Implement sharing functionality
    final totalMeters = ScrollCalculator.instance.pixelsToMeters(_todayStats?.totalDistance ?? 0);
    final shareText = 'I scrolled ${totalMeters.toStringAsFixed(0)}m today with ScrollSense! '
        '${ScrollCalculator.instance.getFunComparison(_todayStats?.totalDistance ?? 0)}';
    
    // Use share_plus package in production
    print('Share: $shareText');
  }
  
  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }
}

// Chart data model
class ChartData {
  final double x;
  final double y;
  final String label;
  
  ChartData({required this.x, required this.y, required this.label});
}
```

#### Step 25: Create Reusable Chart Widget
**File: `lib/widgets/scroll_chart.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:fl_chart/fl_chart.dart';
import '../screens/stats_screen.dart';

enum ChartType { line, bar, pie }

class ScrollChart extends StatelessWidget {
  final List<ChartData> data;
  final String title;
  final ChartType type;
  final Color primaryColor;
  final Color backgroundColor;
  
  const ScrollChart({
    Key? key,
    required this.data,
    required this.title,
    this.type = ChartType.line,
    this.primaryColor = Colors.blue,
    this.backgroundColor = Colors.white,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              title,
              style: const TextStyle(
                fontSize: 16,
                fontWeight: FontWeight.w600,
              ),
            ),
            const SizedBox(height: 16),
            SizedBox(
              height: 200,
              child: _buildChart(),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildChart() {
    switch (type) {
      case ChartType.line:
        return _buildLineChart();
      case ChartType.bar:
        return _buildBarChart();
      case ChartType.pie:
        return _buildPieChart();
    }
  }
  
  Widget _buildLineChart() {
    return LineChart(
      LineChartData(
        lineBarsData: [
          LineChartBarData(
            spots: data.map((d) => FlSpot(d.x, d.y)).toList(),
            isCurved: true,
            color: primaryColor,
            barWidth: 3,
            isStrokeCapRound: true,
            belowBarData: BarAreaData(
              show: true,
              color: primaryColor.withOpacity(0.1),
            ),
            dotData: FlDotData(
              show: true,
              getDotPainter: (spot, percent, barData, index) {
                return FlDotCirclePainter(
                  radius: 4,
                  color: primaryColor,
                  strokeWidth: 2,
                  strokeColor: Colors.white,
                );
              },
            ),
          ),
        ],
        titlesData: FlTitlesData(
          show: true,
          topTitles: AxisTitles(sideTitles: SideTitles(showTitles: false)),
          rightTitles: AxisTitles(sideTitles: SideTitles(showTitles: false)),
          bottomTitles: AxisTitles(
            sideTitles: SideTitles(
              showTitles: true,
              reservedSize: 30,
              interval: data.length > 10 ? 2 : 1,
              getTitlesWidget: (value, meta) {
                final index = value.toInt();
                if (index >= 0 && index < data.length) {
                  return Padding(
                    padding: const EdgeInsets.only(top: 8),
                    child: Text(
                      data[index].label,
                      style: const TextStyle(
                        fontSize: 10,
                        color: Colors.grey,
                      ),
                    ),
                  );
                }
                return const Text('');
              },
            ),
          ),
          leftTitles: AxisTitles(
            sideTitles: SideTitles(
              showTitles: true,
              reservedSize: 40,
              getTitlesWidget: (value, meta) {
                return Text(
                  value.toStringAsFixed(0),
                  style: const TextStyle(
                    fontSize: 10,
                    color: Colors.grey,
                  ),
                );
              },
            ),
          ),
        ),
        gridData: FlGridData(
          show: true,
          drawVerticalLine: false,
          horizontalInterval: _calculateInterval(),
          getDrawingHorizontalLine: (value) {
            return FlLine(
              color: Colors.grey.withOpacity(0.3),
              strokeWidth: 1,
            );
          },
        ),
        borderData: FlBorderData(
          show: true,
          border: Border(
            bottom: BorderSide(color: Colors.grey.withOpacity(0.3)),
            left: BorderSide(color: Colors.grey.withOpacity(0.3)),
          ),
        ),
        barTouchData: BarTouchData(
          touchTooltipData: BarTouchTooltipData(
            tooltipBgColor: primaryColor.withOpacity(0.9),
            getTooltipItem: (group, groupIndex, rod, rodIndex) {
              final dataPoint = data[groupIndex];
              return BarTooltipItem(
                '${dataPoint.label}\n${dataPoint.y.toStringAsFixed(1)}m',
                const TextStyle(
                  color: Colors.white,
                  fontWeight: FontWeight.bold,
                ),
              );
            },
          ),
        ),
      ),
    );
  }
  
  Widget _buildPieChart() {
    final colors = [
      primaryColor,
      primaryColor.withOpacity(0.8),
      primaryColor.withOpacity(0.6),
      primaryColor.withOpacity(0.4),
      primaryColor.withOpacity(0.2),
    ];
    
    return PieChart(
      PieChartData(
        sections: data.asMap().entries.map((entry) {
          final index = entry.key;
          final dataPoint = entry.value;
          final total = data.fold<double>(0, (sum, d) => sum + d.y);
          final percentage = (dataPoint.y / total * 100);
          
          return PieChartSectionData(
            color: colors[index % colors.length],
            value: dataPoint.y,
            title: '${percentage.toStringAsFixed(1)}%',
            radius: 80,
            titleStyle: const TextStyle(
              fontSize: 12,
              fontWeight: FontWeight.bold,
              color: Colors.white,
            ),
          );
        }).toList(),
        sectionsSpace: 2,
        centerSpaceRadius: 40,
        pieTouchData: PieTouchData(
          touchCallback: (FlTouchEvent event, pieTouchResponse) {
            // Handle touch events if needed
          },
        ),
      ),
    );
  }
  
  double _calculateInterval() {
    if (data.isEmpty) return 1;
    
    final maxValue = data.map((e) => e.y).reduce((a, b) => a > b ? a : b);
    
    if (maxValue <= 10) return 2;
    if (maxValue <= 50) return 10;
    if (maxValue <= 100) return 20;
    if (maxValue <= 500) return 50;
    return 100;
  }
}
```

#### Step 26: Create App Usage List Widget
**File: `lib/widgets/app_usage_list.dart`**
```dart
import 'package:flutter/material.dart';
import '../services/scroll_calculator.dart';
import '../utils/formatters.dart';

class AppUsageList extends StatelessWidget {
  final Map<String, double> appUsage;
  final bool showDetails;
  final int maxItems;
  
  const AppUsageList({
    Key? key,
    required this.appUsage,
    this.showDetails = false,
    this.maxItems = 10,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    if (appUsage.isEmpty) {
      return _buildEmptyState();
    }
    
    // Sort apps by usage (distance scrolled)
    final sortedApps = appUsage.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));
    
    final displayApps = sortedApps.take(maxItems).toList();
    final totalDistance = appUsage.values.fold<double>(0, (sum, distance) => sum + distance);
    
    return Column(
      children: [
        ...displayApps.map((entry) => _buildAppUsageItem(
          entry.key,
          entry.value,
          totalDistance,
          context,
        )),
        
        if (sortedApps.length > maxItems) ...[
          const SizedBox(height: 8),
          TextButton(
            onPressed: () {
              _showAllApps(context, sortedApps);
            },
            child: Text('View all ${sortedApps.length} apps'),
          ),
        ],
      ],
    );
  }
  
  Widget _buildAppUsageItem(
    String packageName,
    double distance,
    double totalDistance,
    BuildContext context,
  ) {
    final percentage = (distance / totalDistance * 100);
    final meters = ScrollCalculator.instance.pixelsToMeters(distance);
    final appName = AppFormatters.getAppDisplayName(packageName);
    final emoji = AppFormatters.getAppEmoji(packageName);
    
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Row(
          children: [
            // App icon/emoji
            Container(
              width: 40,
              height: 40,
              decoration: BoxDecoration(
                color: _getAppColor(packageName).withOpacity(0.1),
                borderRadius: BorderRadius.circular(8),
              ),
              child: Center(
                child: Text(
                  emoji,
                  style: const TextStyle(fontSize: 20),
                ),
              ),
            ),
            
            const SizedBox(width: 12),
            
            // App info
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    appName,
                    style: const TextStyle(
                      fontWeight: FontWeight.w600,
                      fontSize: 16,
                    ),
                  ),
                  const SizedBox(height: 4),
                  Text(
                    '${meters.toStringAsFixed(0)}m • ${percentage.toStringAsFixed(1)}%',
                    style: TextStyle(
                      color: Colors.grey[600],
                      fontSize: 14,
                    ),
                  ),
                  if (showDetails) ...[
                    const SizedBox(height: 8),
                    // Progress bar
                    LinearProgressIndicator(
                      value: percentage / 100,
                      backgroundColor: Colors.grey[200],
                      valueColor: AlwaysStoppedAnimation<Color>(
                        _getAppColor(packageName),
                      ),
                    ),
                  ],
                ],
              ),
            ),
            
            // Distance value
            Column(
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Text(
                  '${meters.toStringAsFixed(0)}m',
                  style: TextStyle(
                    fontWeight: FontWeight.bold,
                    fontSize: 16,
                    color: _getAppColor(packageName),
                  ),
                ),
                if (showDetails) ...[
                  const SizedBox(height: 4),
                  Text(
                    _getFunComparison(distance),
                    style: TextStyle(
                      fontSize: 12,
                      color: Colors.grey[600],
                    ),
                  ),
                ],
              ],
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildEmptyState() {
    return Container(
      padding: const EdgeInsets.all(32),
      child: Column(
        children: [
          Icon(
            Icons.apps,
            size: 64,
            color: Colors.grey[400],
          ),
          const SizedBox(height: 16),
          const Text(
            'No app usage data yet',
            style: TextStyle(
              fontSize: 18,
              fontWeight: FontWeight.w600,
              color: Colors.grey,
            ),
          ),
          const SizedBox(height: 8),
          Text(
            'Start scrolling in different apps to see your usage breakdown',
            style: TextStyle(
              fontSize: 14,
              color: Colors.grey[600],
            ),
            textAlign: TextAlign.center,
          ),
        ],
      ),
    );
  }
  
  Color _getAppColor(String packageName) {
    // Assign colors based on app type/name
    if (packageName.contains('instagram')) return Colors.purple;
    if (packageName.contains('tiktok')) return Colors.black;
    if (packageName.contains('twitter')) return Colors.blue;
    if (packageName.contains('facebook')) return Colors.indigo;
    if (packageName.contains('youtube')) return Colors.red;
    if (packageName.contains('whatsapp')) return Colors.green;
    if (packageName.contains('snapchat')) return Colors.yellow;
    if (packageName.contains('linkedin')) return Colors.blue[700]!;
    if (packageName.contains('reddit')) return Colors.orange;
    if (packageName.contains('pinterest')) return Colors.red[300]!;
    
    // Default colors based on hash of package name
    final colors = [
      Colors.blue, Colors.green, Colors.orange, Colors.purple,
      Colors.teal, Colors.indigo, Colors.pink, Colors.cyan,
    ];
    return colors[packageName.hashCode % colors.length];
  }
  
  String _getFunComparison(double distance) {
    final meters = ScrollCalculator.instance.pixelsToMeters(distance);
    
    if (meters < 10) return 'A few steps';
    if (meters < 50) return 'Half a pool';
    if (meters < 100) return 'Football field';
    if (meters < 300) return 'City block';
    if (meters < 1000) return 'Multiple blocks';
    
    return 'Marathon distance!';
  }
  
  void _showAllApps(BuildContext context, List<MapEntry<String, double>> apps) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => DraggableScrollableSheet(
        initialChildSize: 0.7,
        maxChildSize: 0.9,
        minChildSize: 0.5,
        builder: (context, scrollController) => Column(
          children: [
            Container(
              padding: const EdgeInsets.all(16),
              child: Row(
                children: [
                  const Text(
                    'All Apps',
                    style: TextStyle(
                      fontSize: 20,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const Spacer(),
                  IconButton(
                    onPressed: () => Navigator.pop(context),
                    icon: const Icon(Icons.close),
                  ),
                ],
              ),
            ),
            Expanded(
              child: ListView.builder(
                controller: scrollController,
                padding: const EdgeInsets.symmetric(horizontal: 16),
                itemCount: apps.length,
                itemBuilder: (context, index) {
                  final app = apps[index];
                  final totalDistance = appUsage.values.fold<double>(
                    0, (sum, distance) => sum + distance);
                  
                  return _buildAppUsageItem(
                    app.key,
                    app.value,
                    totalDistance,
                    context,
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Day 21-22: Settings Screen and User Preferences

#### Step 27: Comprehensive Settings Screen
**File: `lib/screens/settings_screen.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../config/constants.dart';
import '../services/database_service.dart';
import '../services/accessibility_service.dart';
import '../utils/permissions.dart';
import '../widgets/settings_section.dart';
import '../widgets/settings_tile.dart';

class SettingsScreen extends StatefulWidget {
  const SettingsScreen({Key? key}) : super(key: key);

  @override
  State<SettingsScreen> createState() => _SettingsScreenState();
}

class _SettingsScreenState extends State<SettingsScreen> {
  bool _notificationsEnabled = true;
  int _breakInterval = 30;
  double _dailyGoal = 1000.0;
  bool _accessibilityEnabled = false;
  bool _isLoading = true;
  
  @override
  void initState() {
    super.initState();
    _loadSettings();
  }
  
  Future<void> _loadSettings() async {
    setState(() {
      _isLoading = true;
    });
    
    try {
      final prefs = await SharedPreferences.getInstance();
      final accessibilityEnabled = await PermissionManager.instance.isAccessibilityEnabled();
      
      setState(() {
        _notificationsEnabled = prefs.getBool(AppConstants.keyNotificationsEnabled) ?? true;
        _breakInterval = prefs.getInt(AppConstants.keyBreakInterval) ?? 30;
        _dailyGoal = prefs.getDouble(AppConstants.keyDailyGoal) ?? 1000.0;
        _accessibilityEnabled = accessibilityEnabled;
        _isLoading = false;
      });
    } catch (e) {
      print('Error loading settings: $e');
      setState(() {
        _isLoading = false;
      });
    }
  }
  
  Future<void> _saveSettings() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setBool(AppConstants.keyNotificationsEnabled, _notificationsEnabled);
      await prefs.setInt(AppConstants.keyBreakInterval, _breakInterval);
      await prefs.setDouble(AppConstants.keyDailyGoal, _dailyGoal);
    } catch (e) {
      print('Error saving settings: $e');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey[50],
      appBar: AppBar(
        title: const Text(
          'Settings',
          style: TextStyle(fontWeight: FontWeight.bold),
        ),
        backgroundColor: Colors.transparent,
        elevation: 0,
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : SingleChildScrollView(
              padding: const EdgeInsets.all(16),
              child: Column(
                children: [
                  // Permissions Section
                  SettingsSection(
                    title: 'Permissions',
                    children: [
                      SettingsTile(
                        title: 'Accessibility Service',
                        subtitle: _accessibilityEnabled 
                            ? 'Enabled - ScrollSense can track scrolling'
                            : 'Required to track scrolling across apps',
                        leading: Icon(
                          Icons.accessibility,
                          color: _accessibilityEnabled ? Colors.green : Colors.orange,
                        ),
                        trailing: _accessibilityEnabled
                            ? const Icon(Icons.check_circle, color: Colors.green)
                            : const Icon(Icons.warning, color: Colors.orange),
                        onTap: _accessibilityEnabled ? null : _openAccessibilitySettings,
                      ),
                      SettingsTile(
                        title: 'Notifications',
                        subtitle: 'Get scroll break reminders',
                        leading: const Icon(Icons.notifications),
                        trailing: Switch(
                          value: _notificationsEnabled,
                          onChanged: (value) {
                            setState(() {
                              _notificationsEnabled = value;
                            });
                            _saveSettings();
                          },
                        ),
                      ),
                    ],
                  ),
                  
                  const SizedBox(height: 24),
                  
                  // Goals & Reminders Section
                  SettingsSection(
                    title: 'Goals & Reminders',
                    children: [
                      SettingsTile(
                        title: 'Daily Scroll Goal',
                        subtitle: '${_dailyGoal.toStringAsFixed(0)} meters per day',
                        leading: const Icon(Icons.flag),
                        onTap: () => _showGoalDialog(),
                      ),
                      SettingsTile(
                        title: 'Break Reminder Interval',
                        subtitle: 'Every ${_breakInterval} minutes',
                        leading: const Icon(Icons.schedule),
                        trailing: _notificationsEnabled 
                            ? null 
                            : const Icon(Icons.notifications_off, color: Colors.grey),
                        onTap: _notificationsEnabled ? () => _showIntervalDialog() : null,
                      ),
                    ],
                  ),
                  
                  const SizedBox(height: 24),
                  
                  // Data & Privacy Section
                  SettingsSection(
                    title: 'Data & Privacy',
                    children: [
                      SettingsTile(
                        title: 'Data Storage',
                        subtitle: 'All data stays on your device',
                        leading: const Icon(Icons.storage),
                        trailing: const Icon(Icons.info_outline),
                        onTap: () => _showDataInfoDialog(),
                      ),
                      SettingsTile(
                        title: 'Export Data',
                        subtitle: 'Download your scroll data',
                        leading: const Icon(Icons.download),
                        onTap: () => _exportData(),
                      ),
                      SettingsTile(
                        title: 'Clear All Data',
                        subtitle: 'Permanently delete all stored data',
                        leading: const Icon(Icons.delete_forever, color: Colors.red),
                        textColor: Colors.red,
                        onTap: () => _showDeleteDataDialog(),
                      ),
                    ],
                  ),
                  
                  const SizedBox(height: 24),
                  
                  // About Section
                  SettingsSection(
                    title: 'About',
                    children: [
                      SettingsTile(
                        title: 'Version',
                        subtitle: '1.0.0',
                        leading: const Icon(Icons.info),
                      ),
                      SettingsTile(
                        title: 'Privacy Policy',
                        subtitle: 'How we protect your data',
                        leading: const Icon(Icons.policy),
                        onTap: () => _showPrivacyPolicy(),
                      ),
                      SettingsTile(
                        title: 'Contact Support',
                        subtitle: 'Get help or report issues',
                        leading: const Icon(Icons.support),
                        onTap: () => _contactSupport(),
                      ),
                    ],
                  ),
                  
                  const SizedBox(height: 32),
                  
                  // Debug Section (only in debug mode)
                  if (const bool.fromEnvironment('dart.vm.product') == false) ...[
                    SettingsSection(
                      title: 'Debug',
                      children: [
                        SettingsTile(
                          title: 'Database Size',
                          subtitle: 'Check storage usage',
                          leading: const Icon(Icons.bug_report),
                          onTap: () => _showDatabaseSize(),
                        ),
                        SettingsTile(
                          title: 'Test Notification',
                          subtitle: 'Send a test notification',
                          leading: const Icon(Icons.notifications_active),
                          onTap: () => _testNotification(),
                        ),
                      ],
                    ),
                  ],
                ],
              ),
            ),
    );
  }
  
  Future<void> _openAccessibilitySettings() async {
    await PermissionManager.instance.openAccessibilitySettings();
    
    // Check status after a delay
    await Future.delayed(const Duration(seconds: 2));
    final enabled = await PermissionManager.instance.isAccessibilityEnabled();
    setState(() {
      _accessibilityEnabled = enabled;
    });
  }
  
  void _showGoalDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Set Daily Goal'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('Current goal: ${_dailyGoal.toStringAsFixed(0)}m per day'),
            const SizedBox(height: 16),
            Slider(
              value: _dailyGoal,
              min: 100,
              max: 5000,
              divisions: 49,
              label: '${_dailyGoal.toStringAsFixed(0)}m',
              onChanged: (value) {
                setState(() {
                  _dailyGoal = value;
                });
              },
            ),
            Text(
              _getGoalDescription(_dailyGoal),
              style: TextStyle(
                fontSize: 12,
                color: Colors.grey[600],
              ),
              textAlign: TextAlign.center,
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              _saveSettings();
              Navigator.pop(context);
            },
            child: const Text('Save'),
          ),
        ],
      ),
    );
  }
  
  void _showIntervalDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Break Reminder Interval'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text('How often should we remind you to take a break?'),
            const SizedBox(height: 16),
            ...[ 15, 30, 45, 60, 90, 120].map((minutes) => RadioListTile<int>(
              title: Text('Every $minutes minutes'),
              value: minutes,
              groupValue: _breakInterval,
              onChanged: (value) {
                setState(() {
                  _breakInterval = value!;
                });
              },
            )),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              _saveSettings();
              Navigator.pop(context);
            },
            child: const Text('Save'),
          ),
        ],
      ),
    );
  }
  
  void _showDataInfoDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Data & Privacy'),
        content: const Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Privacy-First Approach',
              style: TextStyle(fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            Text('• All your data stays on your device'),
            Text('• We never send data to external servers'),
            Text('• You can delete all data anytime'),
            Text('• No tracking or analytics'),
            SizedBox(height: 16),
            Text(
              'What We Store',
              style: TextStyle(fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            Text('• Scroll events (distance, time, app)'),
            Text('• Daily statistics'),
            Text('• App preferences'),
            SizedBox(height: 16),
            Text(
              'Data is automatically cleaned up after 30 days to save storage.',
              style: TextStyle(fontStyle: FontStyle.italic),
            ),
          ],
        ),
        actions: [
          ElevatedButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Got it'),
          ),
        ],
      ),
    );
  }
  
  void _showDeleteDataDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Clear All Data'),
        content: const Text(
          'This will permanently delete all your scroll data, statistics, and preferences. This action cannot be undone.\n\nAre you sure you want to continue?',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () async {
              Navigator.pop(context);
              await _deleteAllData();
            },
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.red,
              foregroundColor: Colors.white,
            ),
            child: const Text('Delete All'),
          ),
        ],
      ),
    );
  }
  
  Future<void> _deleteAllData() async {
    try {
      // Show loading
      showDialog(
        context: context,
        barrierDismissible: false,
        builder: (context) => const AlertDialog(
          content: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              CircularProgressIndicator(),
              SizedBox(height: 16),
              Text('Deleting all data...'),
            ],
          ),
        ),
      );
      
      // Delete database data
      await DatabaseService.instance.deleteAllData();
      
      // Clear preferences
      final prefs = await SharedPreferences.getInstance();
      await prefs.clear();
      
      // Reset to defaults
      setState(() {
        _notificationsEnabled = true;
        _breakInterval = 30;
        _dailyGoal = 1000.0;
      });
      
      Navigator.pop(context); // Close loading dialog
      
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('All data has been deleted'),
          backgroundColor: Colors.green,
        ),
      );
    } catch (e) {
      Navigator.pop(context); // Close loading dialog
      
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Error deleting data: $e'),
          backgroundColor: Colors.red,
        ),
      );
    }
  }
  
  Future<void> _exportData() async {
    // For MVP, show info about export feature
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Export Data'),
        content: const Text(
          'Data export feature will be available in a future update. '
          'Currently, all your data is stored locally on your device for privacy.',
        ),
        actions: [
          ElevatedButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('OK'),
          ),
        ],
      ),
    );
  }
  
  void _showPrivacyPolicy() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Privacy Policy'),
        content: const SingleChildScrollView(
          child: Text(
            'ScrollSense Privacy Policy\n\n'
            '1. Data Collection\n'
            'ScrollSense collects scroll activity data from your device to provide usage statistics. This includes:\n'
            '• Scroll distance and speed\n'
            '• App usage patterns\n'
            '• Session duration\n\n'
            '2. Data Storage\n'
            'All data is stored locally on your device. We do not transmit, share, or store your data on external servers.\n\n'
            '3. Data Usage\n'
            'Data is used solely to provide you with insights about your scrolling habits.\n\n'
            '4. Data Deletion\n'
            'You can delete all data at any time through the app settings.\n\n'
            '5. Permissions\n'
            'Accessibility permission is required to detect scrolling across apps.\n\n'
            'Last updated: ${DateTime.now().year}',
          ),
        ),
        actions: [
          ElevatedButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Close'),
          ),
        ],
      ),
    );
  }
  
  void _contactSupport() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Contact Support'),
        content: const Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Need help or found a bug?'),
            SizedBox(height: 16),
            Text('📧 Email: support@scrollsense.app'),
            SizedBox(height: 8),
            Text('🐛 Report issues on GitHub'),
            SizedBox(height: 16),
            Text(
              'Include your device model and Android version when reporting issues.',
              style: TextStyle(fontSize: 12, fontStyle: FontStyle.italic),
            ),
          ],
        ),
        actions: [
          ElevatedButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Close'),
          ),
        ],
      ),
    );
  }
  
  Future<void> _showDatabaseSize() async {
    try {
      final size = await DatabaseService.instance.getDatabaseSize();
      final sizeInMB = size / (1024 * 1024);
      
      if (!mounted) return;
      
      showDialog(
        context: context,
        builder: (context) => AlertDialog(
          title: const Text('Database Size'),
          content: Text(
            'Current database size: ${sizeInMB.toStringAsFixed(2)} MB\n\n'
            'This includes all your scroll events, sessions, and statistics.',
          ),
          actions: [
            ElevatedButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('OK'),
            ),
          ],
        ),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error getting database size: $e')),
      );
    }
  }
  
  void _testNotification() {
    
        gridData: FlGridData(
          show: true,
          drawVerticalLine: false,
          horizontalInterval: _calculateInterval(),
          getDrawingHorizontalLine: (value) {
            return FlLine(
              color: Colors.grey.withOpacity(0.3),
              strokeWidth: 1,
            );
          },
        ),
        borderData: FlBorderData(
          show: true,
          border: Border(
            bottom: BorderSide(color: Colors.grey.withOpacity(0.3)),
            left: BorderSide(color: Colors.grey.withOpacity(0.3)),
          ),
        ),
        lineTouchData: LineTouchData(
          touchTooltipData: LineTouchTooltipData(
            tooltipBgColor: primaryColor.withOpacity(0.9),
            getTooltipItems: (List<LineBarSpot> touchedBarSpots) {
              return touchedBarSpots.map((barSpot) {
                final dataPoint = data[barSpot.x.toInt()];
                return LineTooltipItem(
                  '${dataPoint.label}\n${dataPoint.y.toStringAsFixed(1)}m',
                  const TextStyle(
                    color: Colors.white,
                    fontWeight: FontWeight.bold,
                  ),
                );
              }).toList();
            },
          ),
        ),
      ),
    );
  }
  
  Widget _buildBarChart() {
    return BarChart(
      BarChartData(
        barGroups: data.asMap().entries.map((entry) {
          return BarChartGroupData(
            x: entry.key,
            barRods: [
              BarChartRodData(
                toY: entry.value.y,
                color: primaryColor,
                width: 16,
                borderRadius: const BorderRadius.only(
                  topLeft: Radius.circular(4),
                  topRight: Radius.circular(4),
                ),
              ),
            ],
          );
        }).toList(),
        titlesData: FlTitlesData(
          show: true,
          topTitles: AxisTitles(sideTitles: SideTitles(showTitles: false)),
          rightTitles: AxisTitles(sideTitles: SideTitles(showTitles: false)),
          bottomTitles: AxisTitles(
            sideTitles: SideTitles(
              showTitles: true,
              getTitlesWidget: (value, meta) {
                final index = value.toInt();
                if (index >= 0 && index < data.length) {
                  return Padding(
                    padding: const EdgeInsets.only(top: 8),
                    child: Text(
                      data[index].label,
                      style: const TextStyle(
                        fontSize: 10,
                        color: Colors.grey,
                      ),
                    ),
                  );
                }
                return const Text('');
              },
            ),
          ),
          leftTitles: AxisTitles(
            sideTitles: SideTitles(
              showTitles: true,
              reservedSize: 40,
              getTitlesWidget: (value, meta) {
                return Text(
                  value.toStringAsFixed(0),
                  style: const TextStyle(
                    fontSize: 10,
                    color: Colors.grey,
                  ),
                );
              },
            ),
          ),
        ),
        