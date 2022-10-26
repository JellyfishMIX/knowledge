# 设计模式--builder 模式



## 说明

1. 本文默认已经知道了 builder 模式的使用场景和意义，只分析 builder 模式的实现。
2. 直接通过 demo 讲更清晰。



## 两种传统的构造实例的方式

我们有一个 Computer 类，属性如下，构造实例时分必须和可选。

```java
public class Computer {
    // 必须
    private String cpu;
    // 必须
    private String ram;
    // 可选
    private int usbCount;
    // 可选
    private String keyboard;
    // 可选
    private String display;
}
```

不用 builder 设计模式，通常有两种传统的构造实例的方式：

第一种折叠构造函数模式(telescoping constructor pattern)，即编写多个构造方法，每个构造方法中必须的属性必有，可选属性依次增加：

```java
public class Computer {
     ...
    public Computer(String cpu, String ram) {
        this(cpu, ram, 0);
    }
    public Computer(String cpu, String ram, int usbCount) {
        this(cpu, ram, usbCount, "罗技键盘");
    }
    public Computer(String cpu, String ram, int usbCount, String keyboard) {
        this(cpu, ram, usbCount, keyboard, "三星显示器");
    }
    public Computer(String cpu, String ram, int usbCount, String keyboard, String display) {
        this.cpu = cpu;
        this.ram = ram;
        this.usbCount = usbCount;
        this.keyboard = keyboard;
        this.display = display;
    }
}
```

第二种 JavaBean 模式，就是一堆 getter&setter:

```java
public class Computer {
        ...
    public String getCpu() {
        return cpu;
    }
    public void setCpu(String cpu) {
        this.cpu = cpu;
    }
    public String getRam() {
        return ram;
    }
    public void setRam(String ram) {
        this.ram = ram;
    }
    public int getUsbCount() {
        return usbCount;
    }
...
}
```

### 两种传统构造实例方式的缺点

1. 可读性差，使用时代码量多。
2. JavaBean 的 getter&setter 使用时不是原子性的，属性是分步设置的，在构建过程中对象的状态可能发生变化，造成问题。



## 简化版 builder 模式

为了便于理解，先看简化版 builder 模式。还是要构建一个 Computer 实例，构造实例时属性分必须和可选。

```JAVA
public class Computer {
    // 必须
    private String cpu;
    // 必须
    private String ram;
    // 可选
    private int usbCount;
    // 可选
    private String keyboard;
    // 可选
    private String display;

    private Computer(Builder builder){
        this.cpu = builder.cpu;
        this.ram = builder.ram;
        this.usbCount = builder.usbCount;
        this.keyboard = builder.keyboard;
        this.display = builder.display;
    }
    
    public static class Builder{
        // 必须
        private String cpu;
        // 必须
        private String ram;
        // 可选
        private int usbCount;
        // 可选
        private String keyboard;
        // 可选
        private String display;

        public Builder(String cup,String ram){
            this.cpu=cup;
            this.ram=ram;
        }

        public Builder setUsbCount(int usbCount) {
            this.usbCount = usbCount;
            return this;
        }
        public Builder setKeyboard(String keyboard) {
            this.keyboard = keyboard;
            return this;
        }
        public Builder setDisplay(String display) {
            this.display = display;
            return this;
        }        
        public Computer build(){
            return new Computer(this);
        }
    }
  // 省略getter方法
}
```

1. 在 Computer 中编写一个静态内部类 Builder，然后将 Computer 中的属性都复制到 Builder 类中。
2. 在 Computer 中创建一个 private 的构造函数，入参为 Builder 类型。后面 builder 实例在设置好属性后，会把 builder 实例通过这个构造方法传入，构造出 computer 实例。
3. 在 Builder 中编写一个 public 的构造函数，入参为 Computer 中必须的那些属性，cpu 和 ram。
4. 在 Builder 中编写 setter 方法，对 Computer 中那些可选属性进行赋值，返回值为 Builder 类型的实例。
5. 在 Builder 中编写一个 build 方法，在其中构建 Computer 的实例并返回。



## 经典的 Builder 模式

![See the source image](https://image-hosting.jellyfishmix.com/20221013114147.png)

上面简化版 builder 模式便于理解，经典的 builder 模式有 4 个角色。

1. Product 类: 最终要生成的对象，例如 computer 实例。

2. Builder 抽象类（或使用接口代替）。定义了构建 Product 的构建用抽象方法，其 Builder 实现类需要实现这些构建用方法。还会拥有一个用来返回最终产品的方法 Product getProduct()。

3. ConcreteBuilder: Builder 的实现类，实现 Builder 的抽象方法，编写此种 Builder 实现的方法。

4. Director: 决定如何使用 Builder 实现类提供的构建用方法。拥有一个负责组装的方法 void construct(Builder builder)，在这个方法中通过组织并调用 builder 的方法，可以设置 builder。设置完成后，通过 builder 的构建方法 getProduct() 获得最终的产品。

### Product 类

```java
public class Computer {
    // 必须
    private String cpu;
    // 必须
    private String ram;
    // 可选
    private int usbCount;
    // 可选
    private String keyboard;
    // 可选
    private String display;

    public Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }
    public void setUsbCount(int usbCount) {
        this.usbCount = usbCount;
    }
    public void setKeyboard(String keyboard) {
        this.keyboard = keyboard;
    }
    public void setDisplay(String display) {
        this.display = display;
    }
    @Override
    public String toString() {
        return "Computer{" +
                "cpu='" + cpu + '\'' +
                ", ram='" + ram + '\'' +
                ", usbCount=" + usbCount +
                ", keyboard='" + keyboard + '\'' +
                ", display='" + display + '\'' +
                '}';
    }
}
```

### Builder 抽象类

```java
public abstract class ComputerBuilder {
    public abstract void setUsbCount();
    public abstract void setKeyboard();
    public abstract void setDisplay();
    
    public abstract Computer getComputer();
}
```

### ConcreteBuilder: Builder 的实现类--苹果电脑 Builder

```java
public class MacComputerBuilder extends ComputerBuilder {
    private Computer computer;
    public MacComputerBuilder(String cpu, String ram) {
        computer = new Computer(cpu, ram);
    }
    @Override
    public void setUsbCount() {
        computer.setUsbCount(2);
    }
    @Override
    public void setKeyboard() {
        computer.setKeyboard("苹果键盘");
    }
    @Override
    public void setDisplay() {
        computer.setDisplay("苹果显示器");
    }
    @Override
    public Computer getComputer() {
        return computer;
    }
}
```

### ConcreteBuilder: Builder 的实现类--联想电脑 Builder

```java
public class LenovoComputerBuilder extends ComputerBuilder {
    private Computer computer;
    public LenovoComputerBuilder(String cpu, String ram) {
        computer=new Computer(cpu,ram);
    }
    @Override
    public void setUsbCount() {
        computer.setUsbCount(4);
    }
    @Override
    public void setKeyboard() {
        computer.setKeyboard("联想键盘");
    }
    @Override
    public void setDisplay() {
        computer.setDisplay("联想显示器");
    }
    @Override
    public Computer getComputer() {
        return computer;
    }
}
```

### Director

```java
public class ComputerDirector {
    public void makeComputer(ComputerBuilder builder){
        builder.setUsbCount();
        builder.setDisplay();
        builder.setKeyboard();
    }
}
```

### 使用

1. 首先创建一个 director。
2. 然后创建一个 builder。
3. 接着使用 director 操作 builder，对 builder 进行组装。
4. builder 组装完毕后，使用 builder 构建方法创建产品实例。

```java
public static void main(String[] args) {
    	// 1
        ComputerDirector director=new ComputerDirector();
    	// 2
        ComputerBuilder builder=new MacComputerBuilder("I5处理器","三星125");
    	// 3
        director.makeComputer(builder);
    	// 4
        Computer macComputer=builder.getComputer();
        System.out.println("mac computer:"+macComputer.toString());

        ComputerBuilder lenovoBuilder=new LenovoComputerBuilder("I7处理器","海力士222");
        director.makeComputer(lenovoBuilder);
        Computer lenovoComputer=lenovoBuilder.getComputer();
        System.out.println("lenovo computer:"+lenovoComputer.toString());
}
```

输出:

```java
mac computer:Computer{cpu='I5处理器', ram='三星125', usbCount=2, keyboard='苹果键盘', display='苹果显示器'}
lenovo computer:Computer{cpu='I7处理器', ram='海力士222', usbCount=4, keyboard='联想键盘', display='联想显示器'}
```

