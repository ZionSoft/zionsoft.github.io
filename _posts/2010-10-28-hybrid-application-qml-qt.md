---
layout: post
title: Hybrid Application Using QML and Qt C++
---

Though QML provides a nice way to design user interfaces, and JavaScript is employed there to implement the application logic and works pretty nice in many cases, we might still need Qt C++ in some situations (well, at least JavaScript has limited access outside its sandbox).

## integrate QML into Qt C++

Suppose we have a QML file, named *myqml.qml*, like this:

{% highlight qml %}
// this line should be "import QtQuick 1.0" since Qt 4.7.1
import Qt 4.7
 
Rectangle {
  id: myRectangle
  width: 800
  height: 480
  color: "lightgray"
 
  Text {
    id: myText
    text: "I love hybrid application!"
    anchors.centerIn: parent
    font.pointSize: 28
    font.bold: true
  }
}
{% endhighlight %}

One easy way to integrate is to use the QDeclarativeView class, which provides a widget to display QML files. You just need the following three lines:

{% highlight cpp %}
QDeclarativeView view;
view.setSource(QUrl::fromLocalFile("myqml.qml"));
view.show();
{% endhighlight %}

However, *QDeclarativeView* consumes more resources than normal widgets. Fortunately, we can **integrate QML into a graphics scene**. The following lines shows the basic usage:

{% highlight cpp %}
// provides an environment for instantiating QML components
QDeclarativeEngine engine;
 
// encapsulates a QML component definition
QDeclarativeComponent component(&engine, QUrl::fromLocalFile("myqml.qml"));
 
// creates the graphics item for QML at the engine's root context
QDeclarativeItem *item = qobject_cast<QDeclarativeItem *>(component.create());
 
scene.addItem(item);
{% endhighlight %}

Then with the help of the QDeclarativeItem class, you can easily **access the properties of the QML element**, e.g.:

{% highlight cpp %}
qDebug() << item->property("color").typeName();
item->setProperty("color", QColor(255, 255, 255));
{% endhighlight %}

## exposing Qt C++ objects to QML

You can also expose **native Qt C++ objects to QML** through *QDeclarativeContext*:

{% highlight cpp %}
QDeclarativeContext *context = engine->rootContext();
context->setContextProperty("textFromQt", QString("I love hybrid application!"));
{% endhighlight %}

Then in QML, you can have e.g. the following line to access them:

{% highlight qml %}
text: textFromQt
{% endhighlight %}

You can also use *QDeclarativePropertyMap* to manage the exposed properties:

{% highlight cpp %}
QDeclarativePropertyMap map;
map.insert("key1", "value1");
map.insert("key2", "value2");
context->setContextProperty("map", &map);
{% endhighlight %}

In a QML engine, there could be a couple of contexts, forming a tree structure. The child contexts inherit properties in the parent context. By default, there is only one root context, but you can always add more to give finer control of the exposed data, i.e. different QDeclarativeComponent inside the same context have the same exposed data set.

To **expose a self-defined object**, we can use the following code:

{% highlight cpp %}
// define a class with properties
class MyObject: public QObject
{
  Q_OBJECT
 
  // the NOTIFY signal is needed to inform about changes
  // all properties will be exposed to QML
  Q_PROPERTY(QString text READ text WRITE setText NOTIFY textChanged)
public:
  MyObject(QObject *parent = 0) : QObject(parent), m_text("I love hybrid application!") {}
 
  QString text() const { return m_text; }
  void setText(QString &text)
  {
    m_text = text;
    emit textChanged();
  }
 
signals:
  void textChanged();
 
private:
  QString m_text;
};
 
// then just expose it so QML can access it through name "myObject"
engine->rootContext()->setContextProperty("myObject", new MyObject());
{% endhighlight %}

Moreover, we can **create new QML types**:

{% highlight cpp %}
// define the new type
class MyType : public QDeclarativeItem
{
  Q_OBJECT
 
public:
  MyType(QDeclarativeItem *parent = 0) : QDeclarativeItem(parent)
  {
    setFlag(QGraphicsItem::ItemHasNoContents, false);
  }
 
  void paint(QPainter *painter, const QStyleOptionGraphicsItem *option, QWidget *widget = 0)
  {
    QPen pen(QColor(100, 100, 100), 2);
    painter->setPen(pen);
    painter->drawLine(0, 100, 100, 100);
  }
};
 
// then register to expose it
qmlRegisterType<mychart>("net.zionsoft.mytype", 1, 0, "MyType");
{% endhighlight %}

In QML, you can use it like this:

{% highlight qml %}
import net.zionsoft.mytype 1.0
 
MyChart {
 id: myChart
 width: 100
 height: 200
}
{% endhighlight %}

Now let’s jump to **invoke a Qt C++ function from QML**. Basically, QML can invoke slots and functions declared with *Q_INVOKABLE*. Suppose we have the following function in MyObject:

{% highlight cpp %}
Q_INVOKABLE void showMessage()
{
  QMessageBox::information(NULL, "My Test", "Invoking a native function ;)");
}
{% endhighlight %}

Then you can invoke it in QML:

{% highlight qml %}
myObject.showMessage();
{% endhighlight %}

## write plugins as QML extension

The benefits for using plugins as QML extensions are similar to using shared libraries, and it can be easily achieved with the help of *QDeclarativeExtensionPlugin*. Let’s reuse the MyType class defined in the previous section.

First, we need to **create a plugin**:

{% highlight cpp %}
class MyPlugin : public QDeclarativeExtensionPlugin
{
  Q_OBJECT
public:
  void registerTypes(const char *uri)
  {
    qmlRegisterType<MyType>(uri, 1, 0, "MyType");
  }
};
 
Q_EXPORT_PLUGIN2(myPlugin, MyPlugin);
{% endhighlight %}

Then create a file named *qmldir* to define which plugin to load from where (suppose the plugin is called *myplugin*):

{% highlight text %}
plugin myplugin /path/to/plugin
{% endhighlight %}

Now we can use qmlviewer to launch the QML file:

{% highlight qml %}
// no need to import now
MyChart {
 id: myChart
 width: 100
 height: 200
}
{% endhighlight %}

## summary

* Use *QDeclarativeView* or *QDeclarativeComponent* to integrate a QML file into native Qt C++.
* Qt C++ can access the properties of QML elements through *QDeclarativeItem*.
* Expose native objects to QML through *QDeclarativeContext*.
* New QML types can be exported through *qmlRegisterType*.
* The properties of native objects are exported as properties, and the slots or functions declared with *Q_INVOKABLE* can be invoked in QML.
* Create plugins for extension using *QDeclarativeExtensionPlugin*.
