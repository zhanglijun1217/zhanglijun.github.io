---
title: 设计模式——观察者模式
copyright: true
date: 2018-12-24 23:48:39
tags:
	- 观察者模式
categories:
	- 设计模式
---

## 观察者模式

观察者模式也是我们经常会用到的设计模式之一，这里用一个气象站的一些数据变化通知气象板为例去记录一下观察者设计模式，值得一提的是java中提供了观察者模式的接口和类。

<!-- more -->

### demo

一个气象站通知气象板的小demo，气象站提供温度、气压、湿度的数据给一些气象板提供数据，当气象站发生变化了之后，要通知订阅气象站数据的气象板数据变更。

#### 一般方案

在气温变化的气象站中加入气象板对象，在数据变化时，去调用气象板对象的update方法。

气象站WeatherData类中，定义了看板类成员变量，在dataChange方法中调用了该气象板的update方法来动态的变更气象板的数据。

```java
@Getter
@Setter
public class WeatherData {

    /**
     * 温度
     */
    private float temperature;

    /**
     * 湿度
     */
    private float humidity;

    /**
     * 气压
     */
    private float pressure;

    // 定义进来模拟的当天的看板 在构造函数中进行初始化
    private CurrentConditions currentConditions;

    public WeatherData(CurrentConditions currentConditions) {
        this.currentConditions = currentConditions;
    }

    private void dataChange() {
        currentConditions.update(getTemperature(), getPressure(), getHumidity());
    }

    /**
     * 模拟数据变化过程
     */
    public void setData(float temperature, float pressure, float humidity) {
        setTemperature(temperature);
        setPressure(pressure);
        setHumidity(humidity);

        dataChange();
    }

}
```

模拟的气象站CurrentConditions类中，提供了update方法去修改了自己类中的成员变量。

```java
@ToString
public class CurrentConditions {

    private float currentTemperature;

    private float currentPressure;

    private float currentHumidity;

    /**
     * 进行更新看板中的数据 并且打印
     * @param currentTemperature
     * @param currentPressure
     * @param currentHumidity
     */
    public void update(float currentTemperature, float currentPressure, float currentHumidity) {
        this.currentTemperature = currentTemperature;
        this.currentPressure = currentPressure;
        this.currentHumidity = currentHumidity;

        display();

    }

    public void display() {
        System.out.println("*** Today ***" + toString());
    }

}
```

运行主方法：

```java
public static void main(String[] args) {

    // 新建一个 当天天气的公告板
    CurrentConditions currentConditions = new CurrentConditions();

    WeatherData weatherData = new WeatherData(currentConditions);

    // 模拟气象变化
    weatherData.setData(300f, 100f, 22f);

    // 再次变化
    weatherData.setData(200f, 111f, 222f);

    /**
     * 但是可以想到这种方式是只有一种当天天气的公告板
     * 但是如果有很多公告板 再接入的时候就要在WeatherData中定义这个公告板，在changeData中调用update
     * 扩展性很差
     *
     * 要用观察值模式去增强这个demo的观察者，把变得部分做抽象和接口设计
     * 看use_observer package
     */

}
```

可以看到运行结果：

```java
*** Today ***CurrentConditions(currentTemperature=300.0, currentPressure=100.0, currentHumidity=22.0)
*** Today ***CurrentConditions(currentTemperature=200.0, currentPressure=111.0, currentHumidity=222.0)
```

这种方案的弊端：

- 拓展性比较差，这只是一个气象看板的情况，如果是多个气象看板，那么需要耦合在WeatherData类中。
- 必须要知道每个订阅气象站的看板对象的update方法参数，这个也要去耦合多个逻辑。

#### 观察者模式改进

这里简单描述下观察者模式的定义：

当对象间存在一对多关系时，则使用观察者模式（Observer Pattern）。比如，当一个对象被修改时，则会自动通知它的依赖对象。观察者模式属于行为型模式。被依赖的对象为Subject，依赖的对象为Observer，Subject通知Observer变化 。

那么在我们的demo场景中，我们要去分析不变的部分和变的部分。

- **变的部分：**是一对多关系的维护，气象板可能增加并且减少，并且每个订阅气象站数据的气象板对象去更新的参数和方式也不一样。
- **不变的部分：**是气象站中对观察者们的注册和移除方法，并且要对每个观察者去通知更新其数据的接口；观察者们要去提供update接口。这些不变的部分是可以抽象成接口和其中的成员方法的。

##### 方案

定义主题接口和观察者接口：

订阅的主题Subject接口，这个接口中有观察者的订阅和观察者的移除方法，同时也有通知所有的观察者的方法。

```java
public interface Subject {

    /**
     * 观察者订阅接口
     * @param observer
     */
    void registerObserver(Observer observer);

    /**
     * 移除观察者方法
     * @param observer
     */
    void removerObserver(Observer observer);

    /**
     * 通知所有的观察者方法
     */
    void notifyObservers();
}
```

观察者要实现的接口Observer：

这个接口中定义了update的行为方法，这里是简单写了去主动传入关心的三个数据

```java
public interface Observer {

    void update(Float currentTemperature, Float humidity, Float pressure);
}
```

再去看相应的气象站和气象看板的实现，这里气象站应该去实现主题Subject接口，而气象看板应该实现Observer接口提供自己的更新逻辑。

气象站WeatherData类，里面使用`ArrayList<Observer>`去定义了观察者的集合，此外也有实现了循环去通知观察者和删除观察者的接口。

```java
@Data
public class WeatherData implements Subject {

    /**
     * 温度
     */
    private float temperature;

    /**
     * 湿度
     */
    private float humidity;

    /**
     * 气压
     */
    private float pressure;

    /**
     * 观察者集合
     */
    private ArrayList<Observer> observerArrayList;

    @Override
    public void registerObserver(Observer observer) {
        observerArrayList.add(observer);
    }

    @Override
    public void removerObserver(Observer observer) {
        if (observerArrayList.contains(observer)) {
            observerArrayList.remove(observer);
        }
    }

    @Override
    public void notifyObservers() {
        // 这里的通知 参数是写死的三个 拓展性较差
            // java内置的观察者中有拓展参数 动态去取的

        // 循环观察者去更新
        observerArrayList.forEach(e -> e.update(getTemperature(), getHumidity(), getPressure()));
    }

    /**
     * 模拟参数的变化
     */
    public void dataChange(float temperature, float humidity, float pressure) {
        setTemperature(temperature);
        setHumidity(humidity);
        setPressure(pressure);

        // 这里去通知所有的观察者即可
        notifyObservers();

    }
}
```

这里去定义两个气象看板实现Observer接口，一个是CurrentConditions，一个是ForcastConditions，这里简单模拟就是在update中输出的字符串不相同来模拟不同的任务。

CurrentConditions：

```java
@Data
@ToString
public class CurrentConditions implements Observer{

    private float currentTemperature;

    private float currentPressure;

    private float currentHumidity;

    @Override
    public void update(Float currentTemperature, Float humidity, Float pressure) {
        this.currentHumidity = humidity;
        this.currentPressure = pressure;
        this.currentTemperature = currentTemperature;

        display();
    }

    public void display() {
        System.out.println("*** Today ***" + toString());
    }
}
```

ForcastConditions:

```java
@Data
@ToString
public class ForcastConditions implements Observer {
    private float currentTemperature;

    private float currentPressure;

    private float currentHumidity;

    @Override
    public void update(Float temperature, Float humidity, Float pressure) {
        this.currentHumidity = humidity;
        this.currentPressure = pressure;
        this.currentTemperature = temperature;

        display();
    }

    public void display() {
        System.out.println("*** Forcast ***" + toString());
    }

}
```

在main中运行一下看一下效果：

```java
/**
 * 使用观察者模式增强扩展性的demo
 *      观察者模式：对象之间多对一依赖的一种设计方案，被依赖的对象为subject，依赖的对象为Observer，Subject通知Observer变化。拥有比较强的拓展性
 *
 *
 * @author 夸克
 * @date 2018/12/11 01:02
 */
public class Main {

    public static void main(String[] args) {

        // 两个气象看板 去 订阅天气的变化
        Observer cuerrent = new CurrentConditions();
        Observer forcast = new ForcastConditions();

        // 两个观察者订阅主题
        WeatherData weatherData = new WeatherData();

        ArrayList<Observer> list = new ArrayList<Observer>(){{
            add(cuerrent);
            add(forcast);
        }};
        weatherData.setObserverArrayList(list);

        // 模拟天气变化
        System.out.println("=============天气变化1=============");
        weatherData.dataChange(111f, 222f, 333f);
        System.out.println("=============天气变化2=============");
        weatherData.dataChange(333f, 444f, 555f);

        System.out.println("=============添加一个观察者=============");
        weatherData.registerObserver(new ForcastConditions());

        // 再次模拟天气变化
        weatherData.dataChange(555f, 666f, 777f);
    }
}
```

结果如下

```
=============天气变化1=============
*** Today ***CurrentConditions(currentTemperature=111.0, currentPressure=333.0, currentHumidity=222.0)
*** Forcast ***ForcastConditions(currentTemperature=111.0, currentPressure=333.0, currentHumidity=222.0)
=============天气变化2=============
*** Today ***CurrentConditions(currentTemperature=333.0, currentPressure=555.0, currentHumidity=444.0)
*** Forcast ***ForcastConditions(currentTemperature=333.0, currentPressure=555.0, currentHumidity=444.0)
=============添加一个观察者=============
*** Today ***CurrentConditions(currentTemperature=555.0, currentPressure=777.0, currentHumidity=666.0)
*** Forcast ***ForcastConditions(currentTemperature=555.0, currentPressure=777.0, currentHumidity=666.0)
*** Forcast ***ForcastConditions(currentTemperature=555.0, currentPressure=777.0, currentHumidity=666.0)
```

#### java内置的观察者模式

jdk中有内置的观察者模式，是java.util.Observable类，这个要注意并不是对应着上边自定义观察者模式中的Observer接口，而是对应着我们自定义的主题Subject接口，并且这里去设计成了类。而java中也是用Observer接口去对应观察者模式的Observer接口。

可以先看下java内置的Observer接口：可以看到也是去定义了一个update方法，然后传入了Observable类，也就是订阅的主题，还有就是一个Object的参数，其实就是update的更新数据，这里要求封装成一个Object。

```java
/**
 * A class can implement the <code>Observer</code> interface when it
 * wants to be informed of changes in observable objects.
 *
 * @author  Chris Warth
 * @see     java.util.Observable
 * @since   JDK1.0
 */
public interface Observer {
    /**
     * This method is called whenever the observed object is changed. An
     * application calls an <tt>Observable</tt> object's
     * <code>notifyObservers</code> method to have all the object's
     * observers notified of the change.
     *
     * @param   o     the observable object.
     * @param   arg   an argument passed to the <code>notifyObservers</code>
     *                 method.
     */
    void update(Observable o, Object arg);
}
```

再来看下Subject也就有Observable类：

定义了两个成员变量，changed是标识一个观察者们是否变化的状态，而obs指的就是观察者的集合，这里是用Vector存储的。

```java
private boolean changed = false;
private Vector<Observer> obs;
```

定义的添加和删除观察者都是同步的：（addElement和removeElement就是对Vector的操作添加元素和删除元素的操作，这里不去赘述）

```java
public synchronized void addObserver(Observer o) {
    if (o == null)
        throw new NullPointerException();
    if (!obs.contains(o)) {
        obs.addElement(o);
    }
}

public synchronized void deleteObserver(Observer o) {
        obs.removeElement(o);
    }

```

通知观察者的方法也可以看到分析了存在竞争条件的情况：

一种是新注册的观察者会错过主题进行的通知，另一种是未注册的观察者会被误通知，所以这里同步代码块中，判断了changed这个状态，只有当changed为true才将vetor中存储的观察者赋给arrLocal变量；并且在clearChanged方法中也同步清除了changed为true这个状态。这样去广播通知的时候就不用去同步代码块中执行。

```java
public void notifyObservers(Object arg) {
    /*
     * a temporary array buffer, used as a snapshot of the state of
     * current Observers.
     */
    Object[] arrLocal;

    synchronized (this) {
        /* We don't want the Observer doing callbacks into
         * arbitrary code while holding its own Monitor.
         * The code where we extract each Observable from
         * the Vector and store the state of the Observer
         * needs synchronization, but notifying observers
         * does not (should not).  The worst result of any
         * potential race-condition here is that:
         * 1) a newly-added Observer will miss a
         *   notification in progress
         * 2) a recently unregistered Observer will be
         *   wrongly notified when it doesn't care
         */
        if (!changed)
            return;
        arrLocal = obs.toArray();
        clearChanged();
    }

    for (int i = arrLocal.length-1; i>=0; i--)
        ((Observer)arrLocal[i]).update(this, arg);
}



 protected synchronized void clearChanged() {
        changed = false;
 }
```

在这个类中提供了一些同步方法，比如判断changed是否发生变化、比如返回当前观察者的数量。这里不去赘述。

##### 使用java内置观察者模式改造气象站demo

