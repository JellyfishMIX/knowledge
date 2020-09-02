# Java中的this



## java中什么时候使用this

##### 对this的理解

this只存在于方法内部，用来代表调用改方法的对象。可以理解为每一个方法内部都有一个局部变量叫this，每当初始化一个对象时，就把该对象的地址传递给了该对象每一个方法中的this变量，从而可以在方法内部使用这个的对象。

- 第一种情况 
  在一般方法中，在你的方法中的某个形参名与当前对象的某个成员有相同的名字，这时为了不至于混淆，你便需要明确使用this关键字来指明你要使用某个成员，使用方法是“this.成员名”，而不带this的那个便是形参。另外，还可以用“this.方法名”来引用当前对象的某个方法，但这时this就不是必须的了，你可以直接用方法名来访问那个方法，编译器会知道你要调用的是那一个。

  ```java
  public class DemoThis {
      private String name;
      private int age;
  
      DemoThis(String name, int age) {
          // 你可以加上this来调用方法，像这样：this.setName(name);但这并不是必须的
          setName(name);
          setAge(age);
          this.print();
      }
  
      public void setName(String name) {
          // 此处必须指明你要引用成员变量
          this.name = name;
      }
  
      public void setAge(int age) {
          this.age = age;
      }
  
      public void print() {
          // 在此行中并不需要用this，因为没有会导致混淆的东西
          System.out.println("Name=" + name + " Age=" + age);
      }
  
      public static void main(String[] args) {
          DemoThis dt = new DemoThis("Kevin", "22");
      }
  }
  
  
  ```

- 第二种情况 
  假设有两个类，容器类Container和内容类Component，在Container的成员方法中需要调用Component类的一个对象。Component的构造函数中需要一个调用它的Container类作为参数。

  ```java
  class Container{
      Component comp;
      public void addComponent(){
          comp=new Component(this);
      }
  }
  
  class Component{
      Container myContainer;
      public Component(Container c){
          myContainer=c;
      }
  }
  ```

- 第三种情况 
  构造方法不能想其他方法一样被调用，只能在系统初始化一个对象时被系统调用。虽然构造方法不能被其他函数调用，但是可以被该类的其他构造方法调用，这时用this即可。

  ```java
  class Person{
      int age;
      String name;
      public Person(){
  
      }
      public Person(int age){
          this.age=age;
      }
      public Person(int age,String name){
          this(age);
          this.name=name;
      }
  }
  ```



## this在Java中的必须使用和不推荐使用的情况

在面向对象中this是很常见的，通常是用来指当前对象。

### 必须使用this的情况

- 在setter方法中，参数名和成员变量名相同时，必须使用this加以区分。 
  e.g.

  ```java
  public class Person {
  	private byte age;
  	private String sex;
  	private String name;
  	
  	public Person(byte age,String sex,String name){
  		this.age=age;
  		this.sex=sex;
  		this.name=name;
  	}
  }
  ```

- 在某些方法中，由于命名原因造成两个变量名相同，但是作用范围不同，这时同样需要对成员属性添加this进行区分。

  e.g.

  ```java
  public void printName(){
      String name="local name";
      System.out.println("local variable value  is "+name+"name belongs to Object "+this.name);
  }
  ```

- 在容器类和部件类中，部件往往要接受一个容器的引用作为参数，这时我们可以简单的使用this即可。

  e.g.

  ```java
  /**
   * initialize AdView
   */
  private void initAdView(){
      //初始化广告视图
      AdView adView = new AdView(this);
      FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.FILL_PARENT, FrameLayout.LayoutParams.WRAP_CONTENT);
      //设置广告出现的位置(悬浮于屏幕右下角)		 
      params.gravity=Gravity.BOTTOM|Gravity.RIGHT; 
      //将广告视图加入Activity中
      addContentView(adView, params); 
  }
  ```

- 在构造函数中调用另一个构造函数使用this 

  e.g.

  ```java
  public class Person {
  	private byte age;
  	private String sex;
  	private String name;
  	
  	public Person(){
  		System.out.println("persion constructor");
  	}
  	public Person(byte age){
  		this();
  		this.age=age;
  	}
  	
  	public Person(byte age,String sex,String name){
  		this(age);
  		this.sex=sex;
  		this.name=name;
  	}
  }
  ```

那么以上情况下是必须使用this的场合，那么场合不适合使用this呢 

### 不适合使用this的情况

- 由于this表示的是类的对象，因此在静态成员变量或方法就无法使用"this.数据域"的形式。 

- 在不发生冲突的情况下每个成员属性或方法都使用this.，这么做不友好。因为如果我们有一天需要将某个方法或成员变量改成static，必须还要手动去掉"this."，那会有些麻烦。

  e.g.

  ```java
  public class Person {
  	private byte age;
  	private String sex;
  	private String name;
  	private static String nation = "I am from China";
  	
  	public static void getNation(){
  		// 错误使用
  		// System.out.println(this.nation);
  		System.out.print(nation);
  	}
  }
  ```



## 引用/参考

[java中什么时候使用this - 张哲and哲哥 - CSDN](https://blog.csdn.net/BuZiShuoquoto/article/details/80955695)

[this在Java中的必须使用和不推荐使用的情况 - iteye_15898 - CSDN](https://blog.csdn.net/iteye_15898/article/details/82233184?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.edu_weight&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.edu_weight)