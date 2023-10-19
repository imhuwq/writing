---
title: Qt5 疑难杂症：QGraphicsScene 鼠标单击不触发自定义 QGraphicsItem 的 shape 方法
date: 2023-10-19 23:26:00
categories:
- 技术
tags:
  - Qt5
  - 桌面端开发
  - C++
---

`QGraphicsScene::items` and `QGraphicsScene::itemAt` does not return clicked item of customized class `class Item: public QGraphicsTextItem` even thought `Item::shape` and `Item::boundingRect` method has been overriden properly and `Item::mouseHoverEnter` works with customized `shape`.

今天又遇到一个 `Qt5.12.12` 的坑： `QGraphicsScene::itemsAt(event->scenePos(), QTransform())` 死活不能正确检测到被点击的 `class Item: public QGraphicsTextItem` 元素。

一般而言，这类问题都是因为没有正确使用 `event->scenePos` 或者没有正确 `override` 自定义 Item 的 `shape` 方法。  
之所以叫疑难杂症，就是因为以上一般方法都没用。
<!--more-->

更奇怪的是，只有 `scene::items()` 或者 `scene()::itemAt` 不起作用，其它类似 `mouseHoverIn` 等依赖 `shape` 方法的函数，都能正常工作。
和 GPT4 墨迹了半天，它始终在那里一本正经说废话。上网查，查到的也都是一般情况。最后只能自己去折腾。

进一步调查发现 `itemsAt` 默认使用的 `Qt::IntersectsItemShape` 模式，再细看，除了这个 by shape 的模式，还有 `Qt::IntersectsItemBoundingRect`。死马当活马医，试了一下，行了！

再进一步调查发现，`itemsAt` 或者 `items` 方法即使使用 `Qt::IntersectsItemShape` 模式，也根本没有触发自定义的 `shape` 方法。

到此我也懒得再去看 Qt5.12 的源码查根本了，因为最近已经被 Qt5 的各类大大小小的问题伤得不想动。但我强烈怀疑这是 Qt 自身的 Bug。
想查一下 Qt 得 Release Note 看看是否又相关得修复记录，然后我看到 Qt5.12.12 已经是 Qt5.12 的最后一个版本，停留在 2021 年 11 月 26 日......

最后，附上检测代码。
```c++
// dev.h
#include <QDebug>  
#include <QWidget>  
#include <QPainter>  
#include <QHBoxLayout>  
#include <QApplication>  
#include <QGraphicsView>  
#include <QPainterPath>  
#include <QGraphicsScene>  
#include <QGraphicsTextItem>  
#include <QGraphicsSceneMouseEvent>  
  
class Scene : public QGraphicsScene {  
Q_OBJECT  
public:  
    Scene() : QGraphicsScene() {  
    }  
  
protected:  
    void mousePressEvent(QGraphicsSceneMouseEvent *event) override {  
        auto items_by_rect = items(event->scenePos(), Qt::IntersectsItemBoundingRect);  
        auto items_by_shape = items(event->scenePos(), Qt::IntersectsItemShape);  
        qDebug() << items_by_rect.size() << " item(s) by rect";  
        qDebug() << items_by_shape.size() << " item(s) by shape";  
        qDebug() << "by far, no shape() method has been called on any item yet";  
  
        qDebug() << "calling shape() on all items manually";  
        for (auto it: items()) {  
            it->shape(); // this triggers the shape method  
        }  
    }  
  
};  
  
class Item : public QGraphicsTextItem {  
Q_OBJECT  
public:  
    Item(const QString &text) : QGraphicsTextItem(text) {  
    }  
  
    QPainterPath shape() const override {  
        QPainterPath path;  
        path.addRect(boundingRect());  
        qDebug() << "item shape called";  
        return path;  
    }  
  
    QRectF boundingRect() const override {  
        auto old = QGraphicsTextItem::boundingRect();  
        auto new_rect = QRectF(old.x(), old.y(), old.width() + 20, old.height() + 20);  
        return new_rect;  
    }  
  
    void paint(QPainter *painter, const QStyleOptionGraphicsItem *option, QWidget *widget) override {  
        QGraphicsTextItem::paint(painter, option, widget);  
        painter->drawRoundedRect(boundingRect(), 1, 1);  
        painter->setPen(Qt::red);  
        painter->drawRoundedRect(QGraphicsTextItem::boundingRect(), 1, 1);  
    }  
};


// dev.cpp
#include "dev.h"  
  
int main(int argc, char **argv) {  
    QApplication app(argc, argv);  
  
    auto w = new QWidget();  
    w->resize(1920, 1080);  
  
    auto scene = new Scene();  
    auto view = new QGraphicsView(scene);  
    view->resize(1920, 1080);  
  
    auto item = new Item("hello world");  
    scene->addItem(item);  
  
    auto lay = new QVBoxLayout();  
    lay->setMargin(0);  
    lay->setSpacing(0);  
    lay->addWidget(view);  
    w->setLayout(lay);  
  
    w->show();  
    return app.exec();  
}
```


一天里的自由时间又交代在这里了🤣。
