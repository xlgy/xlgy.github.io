---
layout:     post
title:      Flutter CustomScrollView 的使用 及 常用的Sliver系列组件
subtitle:   Flutter CustomScrollView 的使用 及 常用的Sliver系列组件
date:       2023-05-06
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - flutter
---
## CustomScrollView简介
CustomScrollView是可以使用Sliver来自定义滚动模型（效果）的组件。它可以包含多种滚动模型。包括header，footer，CustomScrollView可以实现把多个彼此独立的可滑动widget组合起来。

## CustomScrollView 一"码"当先

```
  Widget mCustomScrollView() {
    return CustomScrollView(
      slivers: [
        SliverAppBar(
          floating: true,
          title: Text('CustomScrollView Demo'),
          expandedHeight: 300,
          flexibleSpace: Image.network(
            "https://img2.baidu.com/it/u=1977503081,1869926726&fm=253&fmt=auto&app=138&f=JPEG?w=889&h=500",
            fit: BoxFit.cover,
          ),
        ),
        SliverList(
            delegate: SliverChildListDelegate([
          Container(
            height: 80,
            color: Colors.primaries[0],
          ),
          Container(
            height: 80,
            color: Colors.primaries[1],
          ),
          Container(
            height: 80,
            color: Colors.primaries[2],
          ),
          Container(
            height: 80,
            color: Colors.primaries[3],
          ),
          Container(
            height: 80,
            color: Colors.primaries[4],
          ),
        ])),
        SliverAppBar(
          pinned: false,
          expandedHeight: 250.0,
          backgroundColor: Colors.blue,
          flexibleSpace: FlexibleSpaceBar(
            background: Image.network(
              "https://img2.baidu.com/it/u=1977503081,1869926726&fm=253&fmt=auto&app=138&f=JPEG?w=889&h=500",
              fit: BoxFit.cover,
            ),
          ),
        ),
      ],
    );
  }
```

效果：

![](https://images.xiaozhuanlan.com/photo/2022/8eb7c779b33e051f6653dc9d06d576a6.png)

**CustomScrollView其实就是一个Sliver的载体，里面添加的子widget都必须是Sliver!!!接下来讲重点介绍Sliver**


## Sliver的概念
Flutter中提出一个Sliver（中文为“薄片”的意思）概念，如果一个可滚动组件支持Sliver模型，那么该滚动可以将子组件分成好多个“薄片”（Sliver），只有当Sliver出现在视口中时才会去构建它，这种模型也称为“基于Sliver的延迟构建模型”。可滚动组件中有很多都支持基于Sliver的延迟构建模型，如ListView、GridView，但是也有不支持该模
的，如SingleChildScrollView。

## SliverList和SliverGrid
SliverList只有一个属性：delegate，类型是SliverChildDelegate。SliverChildDelegate是一个抽象类，不能直接使用。Flutter中定义好了两个继承于SliverChildDelegate的类对象，可以直接用，分别是：SliverChildListDelegate和SliverChildBuilderDelegate。

### SliverChildListDelegate
先来看SliverChildListDelegate，声明如下：

```
SliverChildListDelegate(
    this.children, {
    ...
  })
```

只有一个必填属性children，是一个类型为Widget的List集合。其他属性几乎不用，暂时忽略。跟ListView构造函数相同，会将所有的子组件一次性的全部渲染出来。

代码示例：

```
  Widget mSliverList(){
    return SliverList(
        delegate:SliverChildListDelegate([
          Container(
            height: 80,
            color: Colors.primaries[0],
          ),
          Container(
            height: 80,
            color: Colors.primaries[1],
          ),
          Container(
            height: 80,
            color: Colors.primaries[2],
          ),
          Container(
            height: 80,
            color: Colors.primaries[3],
          ),
          Container(
            height: 80,
            color: Colors.primaries[4],
          ),
        ]));
  }
```

### SliverChildBuilderDelegate
SliverChildBuilderDelegate则跟ListView.build构造函数类似，需要时才会创建，提高了性能。

```
const SliverChildBuilderDelegate(
    this.builder, {
    this.childCount,
    ...
  })
```

主要参数是builder，是一个返回值为Widget的函数，原型如下：

```
Widget Function(BuildContext context, int index)
```

代码示例：

```
  Widget mSliverList2() {
    return SliverList(
      delegate: SliverChildBuilderDelegate((BuildContext context, int index) {
        return Container(
          color: Colors.red,
          alignment: Alignment.center,
          child: Text("$index"),
        );
      }, childCount: 20),
    );
  }
```

### SliverFixedExtentList
SliverFixedExtentList是固定item高度的SliverList，只是比SliverList多了一个参数itemExtent来设置item高度，其用法跟SliverList一致。

代码示例:

```
Widget mSliverFixedExtentList() {
    return SliverFixedExtentList(
      delegate: SliverChildBuilderDelegate((BuildContext context, int index) {
        return Container(
          color: Colors.red,
          alignment: Alignment.center,
          child: Text("$index"),
        );
      }, childCount: 20),
      itemExtent: 50,
    );
  }
```

### SliverGrid
SliverGrid有两个属性：delegate和gridDelegate。其中delegate跟上面的用法一样，gridDelegate的类型是SliverGridDelegate。
SliverGridDelegate是一个抽象类，定义了子控件Layout相关的接口。Flutter中提供了两个SliverGridDelegate的子类SliverGridDelegateWithFixedCrossAxisCount和SliverGridDelegateWithMaxCrossAxisExtent，可以直接使用。下面分别介绍一下。

#### SliverGridDelegateWithFixedCrossAxisCount

该类实现了一个横轴方向上固定子控件数量的layout的算法，构造函数为：

```
SliverGridDelegateWithFixedCrossAxisCount({
  @required double crossAxisCount, 
  double mainAxisSpacing = 0.0,
  double crossAxisSpacing = 0.0,
  double childAspectRatio = 1.0,
})
```

- crossAxisCount：横轴子元素的数量。此属性值确定后子元素在横轴的长度就确定了，即横轴长度除以crossAxisCount的商。
- mainAxisSpacing：主轴方向的间距。
- crossAxisSpacing：横轴方向子元素的间距。
- childAspectRatio：子元素在横轴长度和主轴长度的比例。由于crossAxisCount指定后，子元素横轴长度就确定了，然后通过此参数值就可以确定子元素在竖轴的长度。

**子元素的大小是通过crossAxisCount和childAspectRatio两个参数决定的。**

#### SliverGridDelegateWithMaxCrossAxisExtent

该类实现了一个横轴方向上子元素为固定最大长度的layout算法，其构造函数为：

```
SliverGridDelegateWithMaxCrossAxisExtent({
  double maxCrossAxisExtent,
  double mainAxisSpacing = 0.0,
  double crossAxisSpacing = 0.0,
  double childAspectRatio = 1.0,
})
```

maxCrossAxisExtent是子元素在横轴方向上的最大长度。之所以是最大长度而不是最终长度，这是因为子元素在横轴方向上的长度仍然是等分的。所以子元素在横轴方向上的元素个数num = 横轴长度 / maxCrossAxisExtent + 1，最终的子元素的宽度= 横轴长度 / num。其他参数和SliverGridDelegateWithFixedCrossAxisCount相同。

SliverGrid的使用方法跟GridView的使用方法保持一致。


### SliverAnimatedList
SliverAnimatedList是带有动画的SliverList，先来看构造函数：

```
SliverAnimatedList({
    Key key,
    @required this.itemBuilder,
    this.initialItemCount = 0,
  })
```

- initialItemCount：item的个数。
- itemBuilder：是一个AnimatedListItemBuilder函数，原型如下：

```
Widget Function(BuildContext context, int index, Animation<double> animation)
```

使用SliverAnimatedList在添加或删除item的时候，需要通过一下方式来操作：

- 1.定义一个GlobalKey

```
GlobalKey<SliverAnimatedListState> _listKey = GlobalKey<SliverAnimatedListState>(); 

```

- 2.将key赋值给SliverAnimatedList

```
SliverAnimatedList(
              key: _listKey,
              initialItemCount: _list.length,
              itemBuilder: _buildItem,
            )
```

- 3.通过key.currentState.insertItem或key.currentState.removeItem来进行添加或删除。

```
_listKey.currentState.insertItem(_index);
_listKey.currentState.removeItem(_index,
        (context, animation) => _buildItem(context, item, animation));
```

- 4._buildItem函数原型如下：

```
Widget _buildItem(BuildContext context, int index, Animation<double> animation) {
    return SizeTransition(
      sizeFactor: animation,
      child: Card(
        child: ListTile(
          title: Text(
            'Item $index',
          ),
        ),
      ),
    );
  }
```

如果想修改动画类型，就需要修改_buildItem中的动画方式。


```
List<int> _list = [];
var _key = GlobalKey<SliverAnimatedListState>();

@override
  Widget build(BuildContext context) {   
return CustomScrollView(
  slivers: <Widget>[
    SliverAppBar(
      actions: <Widget>[
        IconButton(
          icon: Icon(Icons.add),
          onPressed: () {
            SliverAnimatedList.of(context)
            final int _index = _list.length;
            _list.insert(_index, _index);
            _key.currentState.insertItem(_index);
          },
        ),
        IconButton(
          icon: Icon(Icons.clear),
          onPressed: () {
            final int _index = _list.length - 1;
            var item = _list[_index].toString();
            _key.currentState.removeItem(_index,
                (context, animation) => _buildItem(item, animation));
            _list.removeAt(_index);
          },
        ),
      ],
    ),
    SliverAnimatedList(
      key: _key,
      initialItemCount: _list.length,
      itemBuilder:
          (BuildContext context, int index, Animation<double> animation) {
        return _buildItem(_list[index].toString(), animation);
      },
    ),
  ],
);
  }
```

动画1：

```
Widget _buildItem(String _item, Animation _animation) {
  return SlideTransition(
    position: _animation
        .drive(CurveTween(curve: Curves.easeIn))
        .drive(Tween<Offset>(begin: Offset(1, 1), end: Offset(0, 1))),
    child: Card(
      child: ListTile(
        title: Text(
          _item,
        ),
      ),
    ),
  );
}
 
```

动画2：

```
Widget _buildItem(String _item, Animation _animation) {
  return SizeTransition(
    sizeFactor: _animation,
    child: Card(
      child: ListTile(
        title: Text(
          _item,
        ),
      ),
    ),
  );
}
```

### SliverPersistentHeader

这个组件可以实现控件吸顶的效果。先来看构造函数：

```
SliverPersistentHeader({
    Key key,
    @required this.delegate,
    this.pinned = false,
    this.floating = false,
  })
```

其中pinned的效果就是控制header是否保持吸顶效果。另一个重要的属性则是delegate，它的类型是SliverPersistentHeaderDelegate，这是一个抽象类，所以要使用的话，需要自己定义一个子类。
子类需要重写4个父类的函数：

```
@override
  Widget build(BuildContext context, double shrinkOffset, bool overlapsContent) {
    // TODO: implement build
    throw UnimplementedError();
  }
 
  @override
  // TODO: implement maxExtent
  double get maxExtent => throw UnimplementedError();
 
  @override
  // TODO: implement minExtent
  double get minExtent => throw UnimplementedError();
 
  @override
  bool shouldRebuild(SliverPersistentHeaderDelegate oldDelegate) {
    // TODO: implement shouldRebuild
    throw UnimplementedError();
  }
```

其中build返回header显示的内容，maxExtent和minExtent表示最大值和最小值，即header展开和闭合时的高度，若相同则header高度保持不变，若不同，则滚动的时候header的高度会自动在两只之间进行变化。shouldRebuild表示是否需要重新绘制，需要的话则返回true。

### SliverAppBar
构造函数：

```
SliverAppBar({
    this.flexibleSpace,
    this.forceElevated = false,
    this.expandedHeight,
    this.floating = false,
    this.pinned = false,
    this.snap = false,
    this.stretch = false,
    this.stretchTriggerOffset = 100.0,
    this.onStretchTrigger,
    ...
  })
```

其他的参数都跟AppBar是一致的，就忽略了。其中有一些重要的参数：

- expandedHeight：展开时AppBar的高度。
- flexibleSpace：空间大小可变的组件，Flutter给我们提供了一个现成的FlexibleSpaceBar组件，给我们处理好了title过渡的效果。
- floating：向上滚动时，AppBar会跟随着滑出屏幕；向下滚动时，会有限显示AppBar，只有当AppBar展开时才会滚动ListView。
- pinned：当SliverAppBar内容滑出屏幕时，将始终渲染一个固定在顶部的收起状态AppBar。
- snap：当手指离开屏幕时，AppBar会保持跟手指滑动方向相一致的状态，即手指上滑则AppBar收起，手指下滑则AppBar展开。
- stretch：是否拉伸。

**只有floating设置为true时，snap才可以设置为true。**

FlexibleSpaceBar是Flutter提供的一个现成的空间大小可变的组件，并且处理好了title和background的过渡效果。重点说一下其中的stretchModes属性，这个是用来设置AppBar拉伸效果的，有三个枚举值，可以互相搭配使用，但是前提是stretch为true。

- blurBackground：拉伸时使用模糊效果。
- fadeTitle：拉伸时标题将消失。
- zoomBackground：默认值，拉伸时widget将填充额外的空间。

示例：

```

SliverAppBar(
  floating: true,
  snap: true,
  pinned: true,
  expandedHeight: 250,
  flexibleSpace: FlexibleSpaceBar(
    title: Text(this.title),
    background: Image.network(
      'http://img1.mukewang.com/5c18cf540001ac8206000338.jpg',
      fit: BoxFit.cover,
    ),
  ),

```


