---
layout: post
category: 设计模式
date: 2018-03-28
title: 单例模式
description: 设计模式
---

　　单例模式，是一种常见的软件设计模式．在应用这个模式时，单例对象的类必须保证只有一个实例存在．<br>
　　许多时候整个系统只需要拥有一个全局对象，这样有利于我们协调系统整体的行为．<br>
　　比如在某个服务器程序中，该服务器的配置信息存放在一个文件中，这些配置数据由一个单例对象统一读取，然后服务进程中的其他对象再通过这个单例对象获取这些配置信息．这种方式简化了在复杂情况下的配置管理．

　　实现单例模式的思路是: 一个类能返回对象一个引用(永远是同一个)和一个获得该实例的方法(必须是静态方法，通常使用 get_instance 这个名称)；当我们调用这个方法时，如果类持有的引用不为空就返回这个引用，如果类保持的引用为空就创建该类的实例并将该实例的引用赋予该类保持的引用．<br>
　　通常将该类的构造方法定义为私有方法，这样其他处的代码就无法通过调用该类的构造函数来实例化该类的对象，只能通过该类提供的静态方法来得到该类的唯一实例.

　　单例模式在多线程的应用场合下需要注意当单例尚未创建时，有两个线程同时调用 `get_instance` 方法，那么可能会创建两个实例，并且造成内存泄漏，解决这个问题需要在判断引用是否为空之前加锁.

## 代码实现

　　C++

　　~~C++ 在某些地方还是能够实现的比 Java 要优雅灵活方便的．~~

```C++
class CSingleton
{
private:
    CSingleton();
    
public:
    CSingleton(const CSingleton&) = delete;
    static CSingleton& get_instance();
};

/** Meyers's Singleton，在 C++ 11 后算是最好的实现*/
CSingleton &CSingleton::get_instance()
{
    /** 在 C++ 11 下，以下代码是线程安全的*/
    static CSingleton instance;
    return instance;
}
```

　　Java

```Java
public class Singleton
{
    private static volatile Singleton instance = null;
    private Singleton(){}
    
    public static Singleton getInstance()
    {
        if (instance == null)
        {
            /** 加锁是为了避免重复创建，仍需要再判断一次.
             *  如果在第一次判断加锁，则每次获取实例都要加锁，影响速度*/
            synchronized(Singleton.class)
            {
                if (instance == null)
                {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```