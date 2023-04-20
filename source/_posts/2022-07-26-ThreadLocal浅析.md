---
title: ThreadLocal浅析
date: 2022-07-26 21:56:47
categories:
  - java
tags:
  - Thread
---

# ThreadLocal浅析

**先看一个简单的实例**

```java
public class ThreadLocalTest {

  static ThreadLocal<Integer> local = new ThreadLocal<>();

  public static void main(String[] args) throws InterruptedException {
    local.set(1);
    System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: %s",
        Thread.currentThread().getName(), local.get()));
    fc();
    new Thread(() -> {
      fc();
    }).start();
    Thread.sleep(1000L); //保证下面fc执行一定在上面异步代码之后执行
    fc(); //继续在主线程内执行，验证上面那一步是否对主线程上下文内容造成影响
  }

  private static void fc() {
    System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: %s",
        Thread.currentThread().getName(), local.get()));
  }
  
}
// 当前线程名称: main, main方法内获取线程内数据为: 1
// 当前线程名称: main, fc方法内获取线程内数据为: 1
// 当前线程名称: Thread-0, fc方法内获取线程内数据为: null
// 当前线程名称: main, fc方法内获取线程内数据为: 1
```

### ThreadLocal中set方法

```java
public class ThreadLocal<T> {
  
  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else
      createMap(t, value);
  }

  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }

  static class ThreadLocalMap {
    private void set(ThreadLocal<?> key, Object value) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);

      for (Entry e = tab[i];
           e != null;
           // 解决hash冲突 如果当前位置已经有对象，那么看下一个位置
           e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
          e.value = value;
          return;
        }

        if (k == null) {
          replaceStaleEntry(key, value, i);
          return;
        }
      }

      tab[i] = new Entry(key, value);
      int sz = ++size;
      if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
    }
  }  
  
}  
```

**结构图示**

引用自：https://www.cnblogs.com/aobing/p/13382184.html

![image-20220726222748567](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/202207262227614.png)

### TheadLocal中的add方法

```java
public class ThreadLocal<T> {
  public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
  
    static class ThreadLocalMap {
      private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            // 非空并且ThreadLocal相等则返回
            if (e != null && e.get() == key)
                return e;
            else
              	// 如果hash位置上的key不等于当前的ThreadLocal 继续向下一个位置找
                return getEntryAfterMiss(key, i, e);
        }
    }
}
```

**图示**

引用自：https://www.cnblogs.com/aobing/p/13382184.html

![image-20220726222617061](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/202207262226238.png)

### InheritableThreadLocal

在主线程中创建的子线程会继承主线程的ThreadLocal

```java
public class ThreadLocalTest {

  static ThreadLocal<Integer> local = new InheritableThreadLocal<>();

  public static void main(String[] args) throws InterruptedException {
    local.set(1);
    System.out.println(String.format("当前线程名称: %s, main方法内获取线程内数据为: %s",
        Thread.currentThread().getName(), local.get()));
    fc();
    new Thread(() -> {
      fc();
    }).start();
    Thread.sleep(1000L); //保证下面fc执行一定在上面异步代码之后执行
    fc(); //继续在主线程内执行，验证上面那一步是否对主线程上下文内容造成影响
  }

  private static void fc() {
    System.out.println(String.format("当前线程名称: %s, fc方法内获取线程内数据为: %s",
        Thread.currentThread().getName(), local.get()));
  }

}
// 当前线程名称: main, main方法内获取线程内数据为: 1
// 当前线程名称: main, fc方法内获取线程内数据为: 1
// 当前线程名称: Thread-0, fc方法内获取线程内数据为: 1
// 当前线程名称: main, fc方法内获取线程内数据为: 1
```

**源码**

```java
public class Thread implements Runnable {
    
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
  
  private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        // ...... 省略其它代码
   			// 这里处理 InheritableThreadLocal
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
          			// 继承主线程的ThreadLocal
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
				// ...... 省略其它代码
    }  
}
```

### 其它待补充内容

强引用

虚引用

弱引用

软引用