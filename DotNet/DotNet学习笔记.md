# DotNet学习笔记

## C# 语言教程

[C# crash course](http://rbwhitaker.wikidot.com/c-sharp-tutorials)

## WPF相关教程

[WPF教程](https://wpf-tutorial.com/Localization/LanguageStatus/zh/ )

[WPF教程 tutorialspoint](https://www.tutorialspoint.com/wpf/index.htm)


[WPF项目实战教程](https://www.bilibili.com/video/BV1nY411a7T8?p=1)

[WPF入门教程](https://www.bilibili.com/video/BV1mJ411F7zG?p=1)

[WPF的 布局简介](http://tianfeng.cc/Article/8220)

## MVVM是什么

MVVM模式的组成部分:

1. 模型(Model)

模型指的是领域模型，它代表真实状态的内容（一般用面向对象的方法来表述），或者指数据访问层，它代表数据内容（一种以数据为中心的方法来表述）。

2. 视图(View)

正如在Model-View-Controller（MVC）和Model-View-Presenter（MVP）模式中，View是用户在屏幕上看到的结构、布局和外观。它显示Model的表示，并接收用户与View的交互（鼠标点击）。与View的交互（鼠标点击、键盘输入、屏幕点击手势等），并通过定义的数据绑定（属性、事件回调等）将这些处理结果转发给View Model，以连接View和View Model。

3. 视图模型 （View model）

View Model是View的抽象，暴露了公共属性和命令。MVVM没有MVC模式中的Controller，也没有MVP模式中的Presenter，而是有一个绑定器（binder），它可以自动地在View
和View Model中的绑定属性之间的通信。View Model已经被描述为Model中数据的状态。
在MVP模式中，View Model和Presenter的主要区别是Presenter有一个对View的引用，而View Model没有。相反，View直接绑定到View Model上的属性来实现发送和接收，更新。为了有效地运作，这需要一种绑定技术或生成模板代码来做绑定。

4. 绑定器（Binder） 

声明性的数据和命令绑定在MVVM模式中是隐式的。在微软的解决方案堆栈中，Binder是一种叫做XAML的标记语言。Binder使开发者不必编写模板逻辑来同步 View Model和View。当在微软堆栈之外实现时，声明性数据绑定技术的存在使得这种模式成为可能。如果没有Binder，人们通常会使用MVP或MVC来代替，并且 必须写更多的模板（或用其他工具生成模板）。

比如前端的React，Vue，Angular也是用了MVVM的模式，单向数据流，或者双向绑定。





