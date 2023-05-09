---
layout:     post
title:      flutter 常用组件：列表ListView
subtitle:   flutter 常用组件：列表ListView
date:       2023-05-06
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - flutter
---

# 1、通过构造方法直接构建
ListView 提供了一个默认构造函数 ListView，我们可以通过设置它的 children 参数，很方便地将所有的子 Widget 包含到 ListView 中。

不过，这种创建方式要求提前将所有子 Widget 一次性创建好，而不是等到它们真正在屏幕上需要显示时才创建，所以有一个很明显的缺点，就是性能不好。因此，这种方式仅适用于列表中含有少量元素的场景。

```
  Widget _listView(){
    return ListView(
        scrollDirection: Axis.vertical,//排列方向
        itemExtent: 140, //item延展尺寸(宽度)
        children: [
          Container(color: Colors.black),
          Container(color: Colors.red),
          Container(color: Colors.blue),
          Container(color: Colors.green),
          Container(color: Colors.yellow),
          Container(color: Colors.orange),
        ]);
  }
```
**效果展示：**
![](https://images.xiaozhuanlan.com/photo/2022/9efd9aac3a7810d7590d16025bdcf7db.png)

根据代码可以看出，children中添加了多个不同颜色的Container，每个Container实际就是列表页的一行cell，真正开发场景需要我们自定义这个cell，当然flutter也为我们提供了一个默认样式模板ListTile，代码如下：

```
  Widget _listView3(){
    return ListView(
        children: [
          ListTile(leading: Icon(Icons.map),  title: Text('Map')),
          ListTile(leading: Icon(Icons.mail), title: Text('Mail')),
          ListTile(leading: Icon(Icons.message), title: Text('Message')),
        ]);
  }
```
**效果展示：**
![](https://images.xiaozhuanlan.com/photo/2022/1111573391b065097ca7995b0dbb8ae8.png)

ListTile 是 Flutter 提供的用于快速构建列表项元素的一个小组件单元，用于 1~3 行（leading、title、subtitle）展示文本、图标等视图元素的场景，通常与 ListView 配合使用。


# 2、通过ListView.builder构建

- ListView 的另一个构造函数 ListView.builder，则适用于子 Widget 比较多的场景。这个构造函数有两个关键参数：

- itemBuilder，是列表项的创建方法。当列表滚动到相应位置时，ListView 会调用该方法创建对应的子 Widget。itemCount，表示列表项的数量，如果为空，则表示 ListView 为无限列表。

**代码如下：**
```
 Widget _listView1(){
    return Container(
      child: ListView.builder(
          itemCount: 100,//行数
          itemExtent: 50.0,//每行高度
          itemBuilder: (BuildContext context, int index) => ListTile(title: Text("title $index"), subtitle: Text("subtitle $index"))
        ),
      );
  }
```
![](https://images.xiaozhuanlan.com/photo/2022/cb2e496e517381ae7c0933d5a047cdaa.png)


|属性名|描述|
|--|--|
|itemCount| 行数|
|itemExtent|每行高度|
|itemBuilder|生成每行的视图|

# 3、通过ListView.separated构建分割线列表

```
 Widget _listView2(){
    return  Container(
        child:  ListView.separated(
          itemCount: 100, //元素个数
          separatorBuilder: (BuildContext context, int index) => index %2 ==0? Divider(color: Colors.green, height: 10,) : Divider(color: Colors.red, height: 10,),//index为偶数，创建绿色分割线；index为奇数，则创建红色分割线
          itemBuilder: (BuildContext context, int index)  {
            return Container(color: Colors.red,height: 40,);
          },
        )
    );
  }
```
**效果展示**
![](https://images.xiaozhuanlan.com/photo/2022/1da7c2d2a72a8420a78e95c8c5f1dbf9.png)

**这里有一点需要注意：**就是分割线Divider自己会默认带高度的，如果想让分割线的高度为0需要手动设置  height:0，如果未设置则分割线默认是有高度的

```
Divider(color: Colors.red, height: 0,)
```

# 总结
|构造函数名称|特点|适用场景|使用频次|
|--|--|--|--|
|ListView|一次性创造好全部子Widget|适用于展示少量连续子Widget的场景|中|
|ListView.builder|提供了子Widget创造方法，仅在需要展示时才创建|适用于子Widget较多，且视觉效果呈现某种规律性的场景|高|
|ListView.separated|与ListView.builder类似，并提供了自定义分割线的功能|与ListView.builder场景类似|中|


