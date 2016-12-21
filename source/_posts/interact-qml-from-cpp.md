---
title: interact qml from cpp
date: 2016-12-19 14:54:07
tags: ["C++", "Qt", "Qml"]
---

本文翻译一下Qt官网WiKi的一篇文章["Interacting with QML Objects from C++"](http://doc.qt.io/qt-5/qtqml-cppintegration-interactqmlfromcpp.html)

所有QML对象类型都是基于**QObject-derived**的类型，无论这些QML对象类型是内部实现于**引擎**或实现于第三方。这意味着QML引擎可以使用"Qt Meta Object System"来动态实现任何QML对象类型并进行对象自省。

对于基于C++代码生成QML对象，这非常有用，无论是需要通过一个QML object显示呈现出来亦或者实现一个无显示的QML对象数据与C++应用进行数据通信都是如此。一旦生成一个QML对象，它能与C++进行自省，以便进行对属性的读/写、方法调用以及接受信号槽信号。

# Loading QML Objects from C++

一个QML文档可以被**QQmlComponent**或者**QQuickView**加载，**QQmlComponent**加载QML文档作为一个C++对象并能被C++源码修改。**QQuickView**可以做同样的事情，另外**QQuickView**作为一个QWindow-derived类，被加载的文档会作为可视化界面被呈现出来；**QQuickView**通常被用来整合一个可视化QML文档来实现应用的用户界面。

举个例子，假设下面有个**MyItem.qml**文件：
```Javascript
import QtQuick 2.0

Item {
    width: 100; height: 100
}
```
**MyItem.qml**会被**QQmlComponent**或者**QQuickView**通过以下C++代码加载。使用**QQmlComponent**需要调用**QQmlComponent::create()**来生成新的组件实例，当一个**QQuickView**自动生成一个组件实例后，它可以通过**QQuickView::rootObject()**进行访问。
```Javascript
// Using QQmlComponent
QQmlEngine engine;
QQmlComponent component(&engine,
        QUrl::fromLocalFile("MyItem.qml"));
QObject *object = component.create();
...
delete object;

// Using QQuickView
QQuickView view;
view.setSource(QUrl::fromLocalFile("MyItem.qml"));
view.show();
QObject *object = view.rootObject();
```
这里的**object**是一个**MyItem.qml**组件生成的实例。你可以通过**QObject::setProperty()**或者**QQmlProperty**修改*Item*的属性。
```Javascript
object->setProperty("width", 500);
QQmlProperty(object, "width").write(500);
```
类似的，你可以转换到具体的类型然后编译器安全的调用。这种情况下，**MyItem.qml**作为一个被**QQuickItem**类定义的**Item**。
```Javascript
QQuickItem *item = qobject_cast<QQuickItem*>(object);
item->setWidth(500);
```