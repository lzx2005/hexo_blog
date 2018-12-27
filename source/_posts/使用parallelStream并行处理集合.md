---
title: 使用parallelStream并行处理集合
date: 2018-12-27 13:39:22
tags:
    - Java
    - Stream
categories:
    - Java
---

最近在公司的风控系统搬砖，写代码时，其中的某一个步骤是将一个大的Map（内有上万条Key，Value值）遍历一遍，逐一分析每一个Key，Value值，并进行因子解析，以便建立用户画像。其中因子解析的步骤，可能会很长，涉及到数值计算，数据库查询或者接口调用，处理速度从1ms-100ms不等，当总数据量超过一万条时，对整个系统的性能损耗非常大。

## 使用parallelStream

我接手到的代码是这样的：

```java
map.forEach((key, value) -> {
    // 处理数据，耗时1ms-100ms或更高
    System.out.println(key +":"+ value);
});
```

既然用到了Stream来处理，于是我便使用了parallelStream来实现集合的`并行处理`，只需要对Stream调用链加上`parallelStream()`方法即可打开：

```java
map.entrySet().parallelStream().forEach(entry -> {
    // do something
    System.out.println(entry.getKey() +":"+ entry.getValue());
});
```

该方法即可打开Java并行处理集合的功能，让我们来写方法验证该方法是否可以真的提高处理速度。
首先我们构建一个测试数据，一个只有大小为10的HashMap：

```java
private static final HashMap<String,Integer> map = new HashMap<String,Integer>();

static {
    for(int i=0;i<10;i++){
        map.put("k"+i, i);
    }
}
```

编写两个方法（不用并行与使用并行），都输出值，并计算耗时：

```java
long start = System.currentTimeMillis();

map.forEach((key, value) -> {
    sleep(100); // 模拟处理时间
    System.out.print(value + ",");
});

long mid = System.currentTimeMillis();
System.out.println(mid - start + "ms");

map.entrySet().parallelStream().forEach(entry -> {
    sleep(100); // 模拟处理时间
    System.out.print(entry.getValue() + ",");
});

long end = System.currentTimeMillis();
System.out.println(end - mid + "ms");
```

执行后，得到的结果为：

```java
0,1,2,3,4,5,6,7,8,9,1129ms
9,0,5,1,7,3,6,2,4,8,321ms
```

从以上的输出可以得出的基本结论有：

1. 使用parallelStream的代码确实是并行运行了，因为输出不是正序的
2. 使用parallelStream确实可以在某种程度提高集合处理速度


## 线程安全

显而易见，parallelStream是非线程安全的，举个简单的例子：

```Java
private static List<Integer> list1 = new ArrayList<>();
private static List<Integer> list2 = new ArrayList<>();
private static List<Integer> list3 = new ArrayList<>();
private static Lock lock = new ReentrantLock();

public static void main(String[] args) {
    IntStream.range(0, 10000).forEach(list1::add);

    IntStream.range(0, 10000).parallel().forEach(list2::add);

    IntStream.range(0, 10000).forEach(i -> {
        lock.lock();
        try {
            list3.add(i);
        }finally {
            lock.unlock();
        }
    });

    System.out.println("串行执行的大小：" + list1.size());
    System.out.println("并行执行的大小：" + list2.size());
    System.out.println("加锁并行执行的大小：" + list3.size());
}

串行执行的大小：10000
并行执行的大小：9595
加锁并行执行的大小：10000
```

所以，我们在使用parallelStream时，需要注意线程安全的问题，该加锁的就加锁，外部调用的ArrayList，HashMap等也必须使用和其对等的线程安全类，例如：ConcurrentHashMap等。
