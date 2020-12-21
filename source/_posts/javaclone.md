---
title: Java深拷贝和浅拷贝，克隆，对象引用
date: 2018-02-06 11:16:49
tags:
    - java
---
### Java深拷贝和浅拷贝，克隆，对象引用


<!-- more -->

#### 1.克隆和对象引用

```java
Person p = new Person(23, "zhang");  
Person p1 = p;  
  
System.out.println(p);  
System.out.println(p1);  
```

执行结果：

> com.pansoft.zhangjg.testclone.Person@2f9ee1ac
> com.pansoft.zhangjg.testclone.Person@2f9ee1ac

**赋值语句“=”是复制了对象的引用**

```java
Person p = new Person(23, "zhang");  
Person p1 = (Person) p.clone();  
  
System.out.println(p);  
System.out.println(p1);  
```

> com.pansoft.zhangjg.testclone.Person@2f9ee1ac
> com.pansoft.zhangjg.testclone.Person@67f1fba0

.clone()方法是克隆一个对象到新地址，可以克隆的对象继承了cloneable

### 2.浅拷贝与深拷贝

浅拷贝：

如果一个对象里有多个成员变量

```java
public class Person implements Cloneable{  
      
    private int age ;  
    private String name;  
      
    public Person(int age, String name) {  
        this.age = age;  
        this.name = name;  
    }  
      
    public Person() {}  
  
    public int getAge() {  
        return age;  
    }  
  
    public String getName() {  
        return name;  
    }  
      
    @Override  
    protected Object clone() throws CloneNotSupportedException {  
        return (Person)super.clone();  
    }  
}  
```



![](http://www.2cto.com/uploadfile/Collfiles/20140120/20140120090020287.jpg) 

要实现深拷贝：

覆盖.clone()方法，在内部实现对所有成员变量的复制，而且每个成员变量都必须实现cloneable接口：

```java
static class Body implements Cloneable{  
    public Head head;  
    public Body() {}  
    public Body(Head head) {this.head = head;}  
  
    @Override  
    protected Object clone() throws CloneNotSupportedException {  
        Body newBody =  (Body) super.clone();  
        newBody.head = (Head) head.clone();  
        return newBody;  
    }  
      
}  
static class Head implements Cloneable{  
    public  Face face;  
      
    public Head() {}  
    public Head(Face face){this.face = face;}  
    @Override  
    protected Object clone() throws CloneNotSupportedException {  
        return super.clone();  
    }  
}   
```



但是如果多次继承，那每一个父类都需要实现cloneable接口，并复写.clone()，否则就会出现拷贝不彻底的情况。

![](http://www.2cto.com/uploadfile/Collfiles/20140120/20140120090021288.jpg)

浅拷贝也可以用 `org.apache.commons.BeanUtils.copyProperties`实现

### 彻底的深拷贝

1. 手动

2. 利用序列化和反序列化的方式：

   ```java
   public Object deepClone() throws Exception
       {
           // 序列化
           ByteArrayOutputStream bos = new ByteArrayOutputStream();
           ObjectOutputStream oos = new ObjectOutputStream(bos);

           oos.writeObject(this);

           // 反序列化
           ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
           ObjectInputStream ois = new ObjectInputStream(bis);

           return ois.readObject();
       }
   ```

3. apache common类里的`SerializationUtils*.clone(T)`

   ​

   ​
