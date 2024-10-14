# 架构设计基本原则

## 开闭原则（OCP）

一个软件实体如类、模块和函数应该对修改封闭，对扩展开放。

## 单一职责原则

一个类只做一件事，一个类应该只有一个引起它修改的原因。

## 接口隔离原则

客户端不应依赖它不需要的接口。如果一个接口在实现时，部分方法由于冗余被客户端空实现，则应该将接口拆分，让实现类只需依赖自己需要的接口方法。

## 里氏替换原则

子类应该可以完全替换父类。也就是说在使用继承时，只扩展新功能，而不要破坏父类原有的功能。

## 依赖倒置原则

细节应该依赖于抽象，抽象不应依赖于细节。把抽象层放在程序设计的高层，并保持稳定，程序的细节变化由低层的实现层来完成。

## 迪米特法则

又名「最少知道原则」，一个类不应知道自己操作的类的细节，换言之，只和朋友谈话，不和朋友的朋友谈话。

## 合成复用原则

尽量使用对象组合,而不是继承来达到复用的目的。

# 构建型模式

- 工厂方法模式
- 抽象工厂模式
- 单例模式
- 建造型模式
- 原型模式

## 工厂模式

### 简单工厂模式

```java
public class FruitFactory {
    public Fruit create(String type){
        switch (type){
            case "苹果": return new Apple();
            case "梨子": return new Pear();
            default: throw new IllegalArgumentException("暂时没有这种水果");
        }
    }
}

public class User {
    private void eat(){
        FruitFactory fruitFactory = new FruitFactory();
        Fruit apple = fruitFactory.create("苹果");
        Fruit pear = fruitFactory.create("梨子");
        apple.eat();
        pear.eat();
    }
}

```

事实上，将构建过程封装的好处不仅可以降低耦合，如果某个产品构造方法相当复杂，使用工厂模式可以大大减少代码重复。比如，如果生产一个苹果需要苹果种子、阳光、水分，将工厂修改如下：

```java
public class FruitFactory {
    public Fruit create(String type) {
        switch (type) {
            case "苹果":
                AppleSeed appleSeed = new AppleSeed();
                Sunlight sunlight = new Sunlight();
                Water water = new Water();
                return new Apple(appleSeed, sunlight, water);
            case "梨子":
                return new Pear();
            default:
                throw new IllegalArgumentException("暂时没有这种水果");
        }
    }
}
```

总而言之，简单工厂模式就是让一个工厂类承担构建所有对象的职责。调用者需要什么产品，让工厂生产出来即可。它的弊端也显而易见：

- 如果需要生产的产品过多，此模式会导致工厂类过于庞大，承担过多的职责，变成超级类。当苹果生产过程需要修改时，要来修改此工厂。梨子生产过程需要修改时，也要来修改此工厂。也就是说这个类不止一个引起修改的原因。违背了单一职责原则。
- 当要生产新的产品时，必须在工厂类中添加新的分支。而开闭原则告诉我们：类应该对修改封闭。我们希望在添加新功能时，只需增加新的类，而不是修改既有的类，所以这就违背了开闭原则。

### 工厂方法模式

为了解决简单工厂模式的这两个弊端，工厂方法模式应运而生，它规定每个产品都有一个专属工厂。比如苹果有专属的苹果工厂，梨子有专属的梨子工厂，Java 代码如下：

```java
public class AppleFactory {
    public Fruit create(){
        return new Apple();
    }
}

public class PearFactory {
    public Fruit create(){
        return new Pear();
    }
}

public class User {
    private void eat(){
        AppleFactory appleFactory = new AppleFactory();
        Fruit apple = appleFactory.create();
        PearFactory pearFactory = new PearFactory();
        Fruit pear = pearFactory.create();
        apple.eat();
        pear.eat();
    }
}

```

调用者无需知道苹果的生产细节，当生产过程需要修改时也无需更改调用端。同时，工厂方法模式解决了简单工厂模式的两个弊端。

- 当生产的产品种类越来越多时，工厂类不会变成超级类。工厂类会越来越多，保持灵活。不会越来越大、变得臃肿。如果苹果的生产过程需要修改时，只需修改苹果工厂。梨子的生产过程需要修改时，只需修改梨子工厂。符合单一职责原则。
- 当需要生产新的产品时，无需更改既有的工厂，只需要添加新的工厂即可。保持了面向对象的可扩展性，符合开闭原则。

但是相比起简单工厂模式，你需要关心的类又变多了，简单工厂中你只需要和一个工厂类打交道，现在你需要和多个工厂类打交道了。

### 抽象工厂模式

工厂方法模式可以进一步优化，提取出工厂接口：

```java
public interface IFactory {
    Fruit create();
}

public class AppleFactory implements IFactory {
    @Override
    public Fruit create(){
        return new Apple();
    }
}

public class PearFactory implements IFactory {
    @Override
    public Fruit create(){
        return new Pear();
    }
}

public class User {
    private void eat(){
        IFactory appleFactory = new AppleFactory();
        Fruit apple = appleFactory.create();
        IFactory pearFactory = new PearFactory();
        Fruit pear = pearFactory.create();
        apple.eat();
        pear.eat();
    }
}

```

可以看到，我们在创建时指定了具体的工厂类后，在使用时就无需再关心是哪个工厂类，只需要将此工厂当作抽象的 IFactory 接口使用即可。这种经过抽象的工厂方法模式被称作抽象工厂模式。由于客户端只和 IFactory 打交道了，调用的是接口中的方法，使用时根本不需要知道是在哪个具体工厂中实现的这些方法，这就使得替换工厂变得非常容易。

IFactory 中只有一个抽象方法时，或许还看不出抽象工厂模式的威力。实际上抽象工厂模式主要用于替换一系列方法。例如将程序中的 SQL Server 数据库整个替换为 Access 数据库，使用抽象方法模式的话，只需在 IFactory 接口中定义好增删改查四个方法，让 SQLFactory 和 AccessFactory 实现此接口，调用时直接使用 IFactory 中的抽象方法即可，调用者无需知道使用的什么数据库，我们就可以非常方便的整个替换程序的数据库，并且让客户端毫不知情。

抽象工厂模式很好的发挥了开闭原则、依赖倒置原则，但缺点是抽象工厂模式太重了，如果 IFactory 接口需要新增功能，则会影响到所有的具体工厂类。使用抽象工厂模式，替换具体工厂时只需更改一行代码，但要新增抽象方法则需要修改所有的具体工厂类。所以抽象工厂模式适用于增加同类工厂这样的横向扩展需求，不适合新增功能这样的纵向扩展。

## 单例模式

优点：

- 它能够避免对象重复创建，节约空间并提升效率
- 避免由于操作不同实例导致的逻辑错误

### 饿汉式

```java
public class Singleton {
  
    private static Singleton instance = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return instance;
    }
}

```

### 懒汉式

#### **DCL**

```java
public class Singleton {
    
    private static volatile Singleton instance = null;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

```

#### 静态内部类

除了双检锁方式外，还有一种比较常见的静态内部类方式保证懒汉式单例的线程安全：

```java
public class Singleton {
    private static class SingletonHolder {
        public static Singleton instance = new Singleton();
    }
  
    private Singleton() {
    }
  
    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```

- 静态内部类方式是怎么实现懒加载的
- 静态内部类方式是怎么保证线程安全的

Java 类的加载过程包括：加载、验证、准备、解析、初始化。初始化阶段即执行类的 clinit 方法（clinit = class + initialize），包括为类的静态变量赋初始值和执行静态代码块中的内容。但**不会立即加载内部类**，内部类会在使用时才加载。所以当此 Singleton 类加载时，SingletonHolder 并不会被立即加载，所以不会像饿汉式那样占用内存。

另外，Java 虚拟机规定，**当访问一个类的静态字段时，如果该类尚未初始化，则立即初始化此类**。当调用Singleton 的 getInstance 方法时，由于其使用了 SingletonHolder 的静态变量 instance，所以这时才会去初始化 SingletonHolder，在 SingletonHolder 中 new 出 Singleton 对象。这就实现了懒加载。

第二个问题的答案是 Java 虚拟机的设计是非常稳定的，早已经考虑到了多线程并发执行的情况。**虚拟机在加载类的 clinit 方法时，会保证 clinit 在多线程中被正确的加锁、同步。即使有多个线程同时去初始化一个类，一次也只有一个线程可以执行 clinit 方法**，其他线程都需要阻塞等待，从而保证了线程安全。

#### 枚举

```java
public enum  ElvisEnum {
    INSTANCE;
    
    public void get() {}
}

```

懒加载方式在平时非常常见，比如打开我们常用的美团、饿了么、支付宝 app，应用首页会立刻刷新出来，但其他标签页在我们点击到时才会刷新。这样就减少了流量消耗，并缩短了程序启动时间。再比如游戏中的某些模块，当我们点击到时才会去下载资源，而不是事先将所有资源都先下载下来，这也属于懒加载方式，避免了内存浪费。

但懒汉式的缺点就是将程序加载时间从启动时延后到了运行时，虽然启动时间缩短了，但我们浏览页面时就会看到数据的 loading 过程。如果用饿汉式将页面提前加载好，我们浏览时就会特别的顺畅，也不失为一个好的用户体验。比如我们常用的 QQ、微信 app，作为即时通讯的工具软件，它们会在启动时立即刷新所有的数据，保证用户看到最新最全的内容。著名的软件大师 Martin 在《代码整洁之道》一书中也说到：不提倡使用懒加载方式，因为程序应该将构建与使用分离，达到解耦。饿汉式在声明时直接初始化变量的方式也更直观易懂。所以在使用饿汉式还是懒汉式时，需要权衡利弊。

一般的建议是：对于构建不复杂，加载完成后会立即使用的单例对象，推荐使用饿汉式。对于构建过程耗时较长，并不是所有使用此类都会用到的单例对象，推荐使用懒汉式。

## 建造型模式

用于创建构造过程稳定的对象，不同的 Builder 可以定义不同的配置。

## 原型模式

**用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。**

# 结构性模式

- 适配器模式
- 桥接模式
- 组合模式
- 装饰模式
- 外观模式
- 享元模式
- 代理模式

## 适配器模式

适配器模式适用于 **有相关性但不兼容的结构**，源接口通过一个中间件转换后才可以适用于目标接口，这个转换过程就是适配，这个中间件就称之为适配器。

家用电源和 USB 数据线有相关性：家用电源输出电压，USB 数据线输入电压。但两个接口无法兼容，因为一个输出 220V，一个输入 5V，通过适配器将输出 220V 转换成输出 5V 之后才可以一起工作。

让我们用程序来模拟一下这个过程。

```java
class HomeBattery {
    int supply() {
        // 家用电源提供一个 220V 的输出电压
        return 220;
    }
}

class USBLine {
    void charge(int volt) {
        // 如果电压不是 5V，抛出异常
        if (volt != 5) throw new IllegalArgumentException("只能接收 5V 电压");
        // 如果电压是 5V，正常充电
        System.out.println("正常充电");
    }
}

class Adapter {
    int convert(int homeVolt) {
        // 适配过程：使用电阻、电容等器件将其降低为输出 5V
        int chargeVolt = homeVolt - 215;
        return chargeVolt;
    }
}

public class User {
    @Test
    public void chargeForPhone() {
        HomeBattery homeBattery = new HomeBattery();
        int homeVolt = homeBattery.supply();
        System.out.println("家庭电源提供的电压是 " + homeVolt + "V");

        Adapter adapter = new Adapter();
        int chargeVolt = adapter.convert(homeVolt);
        System.out.println("使用适配器将家庭电压转换成了 " + chargeVolt + "V");

        USBLine usbLine = new USBLine();
        usbLine.charge(chargeVolt);
    }
}


```

这就是适配器模式。在我们日常的开发中经常会使用到各种各样的 Adapter，都属于适配器模式的应用。

但适配器模式并不推荐多用。因为未雨绸缪好过亡羊补牢，如果事先能预防接口不同的问题，不匹配问题就不会发生，只有遇到源接口无法改变时，才应该考虑使用适配器。比如现代的电源插口中很多已经增加了专门的充电接口，让我们不需要再使用适配器转换接口，这又是社会的一个进步。

## 桥接模式

## 组合模式

## 装饰模式

## 外观模式

## 享元模式

## 代理模式
### 应用
- 动态代理相关  
    - JDK  
        - Proxy  
    - cglib  
    - 相关应用  
        性能检测、 日志、事务
        - Spring源码  
            - AOP  
        - Mybatis源码

# 行为型模式

## 模板方法模式

## 策略模式

## 命令模式

## 职责链模式

## 状态模式

## 观察者模式
```java
public class Subject {
    private List<Observer> observers = new ArrayList<>();
    
    private int state;
    
    public int getState() {
        return this.state;
    }
    
    public void setState(int state) {
        if(state == this.state) {
            return;
        }
        this.state = state;
        notifyAllObserver();
    }
    
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    private void notifyAllObserver() {
        observers.stream().forEach(Observer::update);
    }
}
```

```java
public abstract class Observer {
    protected Subject subject;
    
    public Observer(Subject subject) {
        this.subject = subject;
        this.subject.attach(this);
    }
    
    public abstract void update();
}
```
## 中介者模式

## 迭代器模式

## 访问者模式

## 备忘录模式

## 解释器模式


> 编者注：需要掌握常用的设计模式有哪些，以及运用场景。
> * 单例模式：资源管理，线程池等
> * 工厂模式 + 策略模式：支付方式
> * 模板模式：父类制定流程，子类实现不同流程
> * 状态模式：状态流转
> * 适配器模式
> * 代理模式