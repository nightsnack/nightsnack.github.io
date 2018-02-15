---
title: 内部类用法
date: 2018-02-08 14:46:41
tags: 
    - 内部类
    - java
---

内部类又称为嵌套类，可以把内部类理解为外部类的一个普通成员。

### 1. 内部类的分类

#### 1.1 成员内部类

​	成员内部类可以无条件访问外部类的所有成员属性和成员方法（包括private成员和静态成员）。、

　　不过要注意的是，当成员内部类拥有和外部类同名的成员变量或者方法时，会发生隐藏现象，即默认情况下访问的是成员内部类的成员。如果要访问外部类的同名成员，需要以下面的形式进行访问：

`外部类.this.成员变量`

`外部类.this.成员方法`

#### 1.2 局部内部类

　　局部内部类是定义在一个方法或者一个作用域里面的类，它和成员内部类的区别在于局部内部类的访问仅限于方法内或者该作用域内。

```java
class People{
    public People() {
         
    }
}
 
class Man{
    public Man(){
         
    }
     
    public People getWoman(){
        class Woman extends People{   //局部内部类
            int age =0;
        }
        return new Woman();
    }
}
```

　　注意，局部内部类就像是方法里面的一个局部变量一样，是不能有public、protected、private以及static修饰符的。

#### 1.3 匿名内部类

　　匿名内部类应该是平时我们编写代码时用得最多的，在编写事件监听的代码时使用匿名内部类不但方便，而且使代码更加容易维护。下面这段代码是一段Android事件监听代码：

#### 1.4 静态内部类

　　静态内部类也是定义在另一个类里面的类，只不过在类的前面多了一个关键字static。静态内部类是不需要依赖于外部类的，这点和类的静态成员属性有点类似，并且它不能使用外部类的非static成员变量或者方法，这点很好理解，因为在没有外部类的对象的情况下，可以创建静态内部类的对象，如果允许访问外部类的非static成员就会产生矛盾，因为外部类的非static成员必须依附于具体的对象。



---

### 2.内部类的使用场景和好处

为什么在Java中需要内部类？总结一下主要有以下四点：

1. 每个内部类都能独立的继承一个接口的实现，所以无论外部类是否已经继承了某个(接口的)实现，对于内部类都没有影响。内部类使得多继承的解决方案变得完整，
2. 方便将存在一定逻辑关系的类组织在一起，又可以对外界隐藏。


1. 方便编写事件驱动程序


1. 方便编写线程代码

　个人觉得第一点是最重要的原因之一，内部类的存在使得Java的多继承机制变得更加完善。



---

### 3.内部类访问外部类

里面的可以自由访问外面的，规则和static一样。（访问非静态时必须先创建对象）

具体如下：

#### 3.1 非静态内部类的非静态方法

直接访问

　　不过要注意的是，当成员内部类拥有和外部类同名的成员变量或者方法时，会发生隐藏现象，即默认情况下访问的是成员内部类的成员。如果要访问外部类的同名成员，需要以下面的形式进行访问：

`外部类.this.成员变量`

`外部类.this.成员方法`

```java
public class Outter {  
    int i = 5;  
    static String string = "Hello";  
      
    class Inner1{  
        void Test1 (){  
            System.out.println(i);  
            System.out.println(string);  
        }  
    }  
}
```



#### 3.2. 静态内部类的非静态方法

因为静态方法访问非静态外部成员需先创建实例，所以访问i时必须先new外部类。

>  因为静态的内部类可以脱离外部类创建，不用先创建外部类实例，这样要访问外部类的静态数据还好，毕竟外部类的静态数据也不需要先创建外部类实例，但是访问外部类的非静态变量或者方法的时候就需要先创建外部类实例。

```java
public class Outter {  
    int i = 5;  
    static String string = "Hello";  
      
    static class Inner2{  
        void Test1 (){  
            System.out.println(new Outter().i);  
            System.out.println(string);  
        }  
    }  
} 
```



#### 3.3 静态内部类的静态方法

规则如上

```java
public class Outter {  
    int i = 5;  
    static String string = "Hello";  
      
    static class Inner2{  
        static void Test2 (){  
            System.out.println(new Outter().i);  
            System.out.println(string);  
        }  
    }  
}
```

但是static方法只能访问static变量，假如在Inner2里有一个非static属性int i = 16，直接在test2里访问会产生编译报错。

```java
public class Main {
    int i = 5;
    static String string = "Hello";

    static class Inner2{
        int i = 16;
        static void Test2 (){

            System.out.println(new Main().i);
            System.out.println(string);
            // System.out.println(i); 这里报错！！！
        }
    }
}

```







###  4.外部类访问内部类

大方向：因为将内部类理解为外部类的一个普通成员，所以外面的访问里面的需先new一个对象。

#### 4.1非静态方法访问非静态内部类的成员：

**new 内部类对象，然后直接访问**

```java
public class Outter {  
    void Test1(){  
        System.out.println(new Inner1().i);  
    }  
    class Inner1{  
        int i = 5;  
//      static String string = "Hello";  定义错误！  因为非静态的内部类成员不能创建静态变量，不然静态变量可以独立访问，而内部类又依赖外部类存在这样就矛盾了
    }  
}  
```



####  4.2 静态方法访问非静态内部类的成员

**静态方法Test2访问非静态Inner1需先new外部类；**

**继续访问非静态成员i需先new 内部类**

所以访问规则为：new Outter().new Inner1().i。

毕竟你都静态方法了，可以独立存在的，那访问内部类的时候万一外部类不存在不就掉不到了嘛，所以先创建个外部类

```java
public class Outter {  
    static void Test2(){  
        System.out.println(new Outter().new Inner1().i);  
    }  
    class Inner1{  
        int i = 5;  
//      static String string = "Hello";  定义错误！  
    }  
}
```



反正要记得，访问static修饰的内部类就像调用static修饰的方法一样，就是内部类里还有属性或者对象，访问静态就可以通过名字一路到底，非静态就需要创建相应的实例。




### 5.匿名内部类

匿名内部类访问外部成员变量时，成员变量前应加final关键字。

```java
final int k = 6;  
((Button)findViewById(R.id.button2)).setOnClickListener(new OnClickListener() {  
    @Override  
    public void onClick(View v) {  
        // TODO Auto-generated method stub  
        System.out.println(k);  
    }  
});  
```

原因是，内部类编译的时候会把外部类拷贝一份作为内部类的变量独立使用，为了保证数据一致性，防止内部类线程修改了外部变量，所以直接设置成final，一旦初始化不可修改。