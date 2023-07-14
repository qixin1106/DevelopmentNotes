# NavigationBar组件

* [NavigationBar](#NavigationBar)
* [NavigationRail](#NavigationRail)


# NavigationBar
该组件用于制作类似iOS的底部Tabbar

![截屏2023-07-14 15.15.20](media/%E6%88%AA%E5%B1%8F2023-07-14%2015.15.20.png)

```dart
class _MyHomePageState extends State<MyHomePage> {

  /// 记录当前页面
  int currentPageIndex = 0;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          title: const Text('Title'),
        ),
        // 配置Tabbar
        bottomNavigationBar: _createTabBar(),
        // 配置对应的4个页面
        body: _createPages()[currentPageIndex]);
  }

  /// 创建四个页面
  List<Widget> _createPages() {
    List<Widget> list = [];
    for (var i = 0; i < 4; i++) {
      list.add(Center(child: Text("page$i")));
    }
    return list;
  }

  /// 创建底部Tabbar 
  NavigationBar _createTabBar() {
    return NavigationBar(
      animationDuration: const Duration(milliseconds: 250),
      // 选中后指示器颜色
      indicatorColor: Colors.green,
      
      // 背景颜色
      backgroundColor: Colors.white,
      
      // 阴影, 如果此处不设置为0,那么背景颜色不纯.
      elevation: 0,
      
      // 切换index回调
      onDestinationSelected: (index) {
        setState(() {
          currentPageIndex = index;
        });
      },
      
      // 当前选择的index
      selectedIndex: currentPageIndex,
      
      // 配置对应Item
      destinations: const <NavigationDestination>[
        NavigationDestination(
          icon: Icon(Icons.home),
          selectedIcon: Icon(
            Icons.home,
            color: Colors.white,
          ),
          label: 'home',
        ),
        NavigationDestination(
          icon: Icon(Icons.explore),
          selectedIcon: Icon(
            Icons.explore,
            color: Colors.white,
          ),
          label: 'explore',
        ),
        NavigationDestination(
          icon: Icon(Icons.person),
          selectedIcon: Icon(
            Icons.person,
            color: Colors.white,
          ),
          label: 'person',
        ),
        NavigationDestination(
          icon: Icon(Icons.settings),
          selectedIcon: Icon(
            Icons.settings,
            color: Colors.white,
          ),
          label: 'settings',
        ),
      ],
    );
  }
}
```





# NavigationRail

用于适配桌面端的布局样式.

![截屏2023-07-14 15.43.11](media/%E6%88%AA%E5%B1%8F2023-07-14%2015.43.11.png)

```dart
class _MyHomePageState extends State<MyHomePage> {
  /// 记录当前页面
  int currentPageIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Row(
      children: [
        // 创建侧边Tabbar
        _createSlideTabbar(),
        // 分隔线
        const VerticalDivider(thickness: 1, width: 1),
        // page显示
        Expanded(child: _createPages()[currentPageIndex]),
      ],
    ));
  }

  /// 创建大屏设备的侧边Tabbar
  NavigationRail _createSlideTabbar() {
    return NavigationRail(
      labelType: NavigationRailLabelType.all,
      indicatorColor: Colors.green,
      backgroundColor: Colors.white,
      onDestinationSelected: (index) {
        setState(() {
          currentPageIndex = index;
        });
      },
      selectedIndex: currentPageIndex,
      destinations: const [
        NavigationRailDestination(
          icon: Icon(Icons.home),
          selectedIcon: Icon(
            Icons.home,
            color: Colors.white,
          ),
          label: Text("home"),
        ),
        NavigationRailDestination(
          icon: Icon(Icons.explore),
          selectedIcon: Icon(
            Icons.explore,
            color: Colors.white,
          ),
          label: Text("explore"),
        ),
        NavigationRailDestination(
          icon: Icon(Icons.person),
          selectedIcon: Icon(
            Icons.person,
            color: Colors.white,
          ),
          label: Text("person"),
        ),
        NavigationRailDestination(
          icon: Icon(Icons.settings),
          selectedIcon: Icon(
            Icons.settings,
            color: Colors.white,
          ),
          label: Text("settings"),
        ),
      ],
    );
  }

  /// 创建四个页面
  List<Widget> _createPages() {
    List<Widget> list = [];
    for (var i = 0; i < 4; i++) {
      list.add(Container(
        color: const Color.fromARGB(255, 228, 228, 228),
        child: Center(child: Text("page$i")),
      ));
    }
    return list;
  }
}
```