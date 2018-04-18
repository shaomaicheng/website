---
title: 单例模式学习总结 
date: 2016-09-07 16:23:16
tags:
---
今天温习了Singleton模式，仔细把2本书上的相关内容研究了一遍，现在做一下总结
### 1. 懒汉单例模式
懒汉单例模式是声明一个static变量，在用户第一次调用getInstance时进行初始化。示例代码如下
```
public class Singleton {

    private static Singleton mInstance;

    private Singleton() {

    }

    public synchronized static Singleton getInstance() {
        if (mInstance == null) {
            mInstance = new Singleton();
        }
        return mInstance;
    }
}
```
***懒汉单例模式总结***
**优点：** 只有在使用的时候才进行实例化，在一定程度上节约资源。
**缺点：** 即使mInstance不为null都需要每次进行线程同步，造成不必要的同步开销。

### 2. DCL单例模式（Double CheckLock）
DCL单例模式是进行2次非空检查，第一次检查，如果mInstance非空，则不用进行同步，避免了不必要的同步开销。第二次检查，则是类似懒汉模式的在非空状态下才进行实例化。
DCL模式示例代码如下
```
public class Singleton {
	private static Singleton mInsatnce = null;

    private Singleton() {

    }

    public static Singleton getInsatnce() {
        if (mInsatnce == null) {
            synchronized (Singleton.class) {
                if (mInsatnce == null) {
                    mInsatnce = new Singleton();
                }
            }
        }
        return mInsatnce;
    }
}
```
***DCL单例模式总结***
**优点：** 第一次执行getInstance的时候单例对象才会被实例化，并且避免了不必要的同步开销，资源利用率很高。
**缺点：** 第一加载的时候比较慢，可能会失败。失败的原因源自java内存模型，假设线程A执行到mInsatnce = new Singleton()的时候，jvm会做3件事
1. 给对象实例分配内存
2. 调用对象构造函数，初始化成员字段
3. 将mInstance指向分配的内存空间，此时mInstance已经不是null
但是在jdk1.5之前，java编译器允许乱序执行。所以最终的步骤既可能是1->2->3，也可能是1->3->2。如果是后者，那么A执行完3执行2之前，切换到B线程上，第3步骤已经完成，线程直接取出mInstance对象，但是因为第2步骤没有完成，B线程使用的时候会直接出错，这个就是DCL失效。

### 3. 最常用的单例模式
示例代码如下：
```
public class Singleton {
	private Singleton() {

    }

    public static Singleton getInstance() {
        return SingletonViewHolder.mInsatnce;
    }

    private static class SingletonHolder {
        private static final Singleton mInsatnce = new Singleton();
    }
}
```
***常用单例模式总结***
此种写法在第一次加载Singleton类的时候不会初始化mInsatnce，只有在第一次调用getInsatnce方法的时候，JVM才会去加载SingletonHolder类，此种方法可以保证线程安全性和单例对象的唯一性。所以是最推荐使用的单例模式写法。


### 4. 枚举单例
在java中，也可以利用枚举实现单例模式
示例代码如下：
```
public enum SingletonEnum {
	INSTANCE;
	public void doSth() {
		//do something in this method
	}
}
```
***枚举单例总结***
**优点：** 线程安全，代码简单，任何情况下都能保证对象的单例

***说明***
上述单例实现方法（枚举除外），都有对象反序列化之后会重新创建对象实例的问题，若需要解决此问题，需要添加一个方法
```
private Object readResolve() throws ObjectStreamException {
	return mInstance;
}
```
在此方法中，将反序列化时默认返回新建一个对象强制改为返回原有的mInstance对象。

### 5. 使用容器实现单例模式
代码示例：
```
public class SingletonManager {
    private static HashMap<String, Object> map = new HashMap<>();

    public static void registerService(String key, Object instance) {
        if (!map.containsKey(key)) {
            map.put(key, instance);
        }
    }

    public static Object getService(String key) {
        return map.get(key);
    }
}
```
***单例容器总结***
使用容器对单例对象进行统一管理，对用户隐藏了具体实现，降低了耦合度。在Android中，Context.getSystemService()也就是一种单例容器的使用案例。


#*单例模式总结*
**优点**
1. 只存在一个实例对象，减少了内存开支
2. 避免对文件等资源的多重占用
3. 可以利用单例模式设置全局变量，用来进行共享资源的访问

**缺点**
1. 一般没有接口，扩展困难
2. 在Android中，如果单例模式持有Context的引用，很容易引发内存泄漏。所以在单例中推荐使用ApplicationContext。

综上述
个人认为在简单需求下，问题最少，代码最简单，用起来最为简便的仍然是枚举单例，但枚举单例。但是在复杂的需求的时候，enum仍然很难满足需求。

**参考书籍**
1. 《Effective Java》
2. 《Android源码设计模式解析与实战》
