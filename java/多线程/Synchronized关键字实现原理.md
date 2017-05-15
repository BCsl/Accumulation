# Synchronized 实现原理

同步的意义：一个是竞态条件，另一个是内存可见性

## Synchronized 用法

- 类非静态方法

  修饰类非静态方法，是对**当前实例**对象上锁，保护的是同一个对象的方法调用，怎么说？多个线程是可以同时执行同一个 `synchronized` 实例方法，只要它们（线程）访问的实例对象是不同的

- 类静态方法

  保护的是所修饰的类 XXX.class，即保护同一个类的类方法的调用

- 代码块

  在于修饰的是对象还是 xxx.class

## 实现

如下代码为例

```java
public class TestSynchronized {

    public static void main(String[] args) {
        TestSynchronized testSynchronized = new TestSynchronized();
        testSynchronized.method();
    }

    public void method() {
        synchronized (this) {
            System.out.println("Method 1 start");
        }
    }
}
```

反编译

```java
//...

# virtual methods
.method public method()V
    .registers 3

    .prologue
    .line 14      //同步块开始
    monitor-enter p0

    .line 15
    :try_start_1
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;

    const-string v1, "Method 1 start"

    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V  //System.out.println("Method 1 start");

    .line 16        //同步块结束
    monitor-exit p0

    .line 17      //方法结束
    return-void

    .line 16      //这里又有个 同步块结束，NullPointerException - the object reference on the stack is null.
    :catchall_a
    move-exception v0

    monitor-exit p0
    :try_end_c
    .catchall {:try_start_1 .. :try_end_c} :catchall_a

    throw v0
.end method
```

可以看到同步块开始和结束的地方使用了 `monitor-enter` 和 `monitor-exit` 两个语法，[详情](https://cs.au.dk/~mis/dOvs/jvmspec/ref--44.html)，简单来说，每个对象有一个监视器锁（monitor）。当 `monitor` 被占用时就会处于锁定状态，线程执行 `monitor-enter` 指令时尝试获取 `monitor` 的所有权，过程如下：

- 1.如果 `monitor` 的进入数为 0，则该线程进入 `monitor`，然后将进入数设置为 1，该线程即为 `monitor` 的所有者
- 2.如果线程已经占有该 `monitor`，只是重新进入，则进入 `monitor` 的进入数加1（可重入性，并不是所有锁都是可重入的）
- 3.如果其他线程已经占用了 `monitor`，则该线程进入阻塞状态，直到 `monitor` 的进入数为 0，再重新尝试获取 `monitor` 的所有权

可以看出 `Synchronized` 的实现原理，`Synchronized` 的语义底层是通过一个 `monitor` 的对象来完成
