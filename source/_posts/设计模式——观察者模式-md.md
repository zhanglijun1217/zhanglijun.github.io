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