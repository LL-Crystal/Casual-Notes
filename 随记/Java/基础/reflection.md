# Java反射机制

Java是一门强类型语言，一般情况下初始化一个对象必须先知道对象的类型：类的定义，然后根据new初始化对象。
而反射一开始并不知道我要初始化的类对象是什么，自然也无法使用 new 关键字来创建对象

>反射就是在运行时才知道要操作的类是什么，并且可以在运行时获取类的完整构造，
并调用其对应的方法
---

正射
```
Apple apple = new Apple();
apple.setPrice(4);
```

反射
```
Class clz = Class.forName("com.chenshuyi.reflect.Apple");
Method method = clz.getMethod("setPrice", int.class);
Constructor constructor = clz.getConstructor();
Object object = constructor.newInstance();
method.invoke(object, 4);
```

### 反射的使用

1、使用

实例获取步骤：

- 获取类的 Class 对象实例：
```
Class clz = Class.forName("com.zhenai.api.Apple");
```

- 根据 Class 对象实例获取 Constructor 对象：
```
Constructor appleConstructor = clz.getConstructor();
```

- 使用 Constructor 对象的 newInstance 方法获取反射类对象：
```
Object appleObj = appleConstructor.newInstance();
```

方法调用步骤：

- 获取方法的 Method 对象：
Method setPriceMethod = clz.getMethod("setPrice", int.class);
- 利用 invoke 方法调用方法:
setPriceMethod.invoke(appleObj, 14);

2、api

1、获取反射中的Class对象

- 第一种，使用 Class.forName 静态方法。当你知道该类的全路径名时，你可以使用该方法获取 Class 类对象。
```
Class clz = Class.forName("java.lang.String");
```

- 第二种，使用 .class 方法，这种方法只适合在编译前就知道操作的 Class。
```
Class clz = String.class;
```

- 第三种，使用类对象的 getClass() 方法。
```
String str = new String("Hello");
Class clz = str.getClass();
```

2、通过反射创建类对象

通过反射创建类对象主要有两种方式：通过 Class 对象的 newInstance() 方法、通过 Constructor 对象的 newInstance() 方法。

- 第一种：通过 Class 对象的 newInstance() 方法。
```
Class clz = Apple.class;
Apple apple = (Apple)clz.newInstance();
```

- 第二种：通过 Constructor 对象的 newInstance() 方法
```
Class clz = Apple.class;
Constructor constructor = clz.getConstructor();
Apple apple = (Apple)constructor.newInstance();
```

通过 Constructor 对象创建类对象可以选择特定构造方法，而通过 Class 对象则只能使用默认的无参数构造方法。下面的代码就调用了一个有参数的构造方法进行了类对象的初始化。
```
Class clz = Apple.class;
Constructor constructor = clz.getConstructor(String.class, int.class);
Apple apple = (Apple)constructor.newInstance("红富士", 15);
```

3、通过反射获取类属性、方法、构造器

- 通过 Class 对象的 getFields() 方法可以获取 Class 类的属性，但无法获取私有属性
```
Class clz = Apple.class;
Field[] fields = clz.getFields();
for (Field field : fields) {
    System.out.println(field.getName());
}
```

- 通过Class 对象的 getDeclaredFields() 方法则可以获取包括私有属性在内的所有属性
```
Class clz = Apple.class;
Field[] fields = clz.getDeclaredFields();
for (Field field : fields) {
    System.out.println(field.getName());
}
```

### 反射的实现

反射的实现包括java应用层层面和jvm层面

- 应用层面：任何一个写好的编译成功的Class文件，在被类加载加载完成后，都会对应这一个java.lang.Class<T>的实例，包含了该类的所有属性结构信息，从而通过该实例可以获取该类的任何信息。
从Class类结构上来看java的访问控制符都停留在编译层，也就是它不会在.class文件中留下任何的痕迹，因此通过反射可以拿到类的任何属性。

- jvm层面：底层实现

### 源码分析


[https://zhuanlan.zhihu.com/p/34168509](https://zhuanlan.zhihu.com/p/34168509)