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
        