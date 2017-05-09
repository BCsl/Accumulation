# JAVA 的 finally 的常见问题

## Java finally 语句到底是在 return 之前还是之后执行?

```java
public class TestFinal {

    public static void main(String[] args) {
        int result = test();
        System.out.println(result);
    }

    public static int test() {
        int t = 0;
        try {
            return t;
        } finally {
            ++t;
            System.out.println("final t:" + t);
        }
    }
}
```

这个问题很普遍的出现在各种笔试题，关于执行顺序上，不知道什么时候我已经记住了一个观点就是，变量 t 在 return 的时候记录的是和 t 的值一致的一个局部变量，所以最后的 finally 块执行的时候，对 t 的修改并不影响最终 return 的值

打印结果，finally 先执行再 return

```java
final t:1
0
```

### 查看源码验证

理解这个过程的最好方式就是查看源码，且看看编译器是怎样处理这个流程的

```java
.class public Lgithub/hellocsl/example/gradleplugin/TestFinal;
.super Ljava/lang/Object;
.source "TestFinal.java"

//....

.method public static test()I //静态 test 方法
    .registers 5

    .prologue
    .line 11
    const/4 v0, 0x0 // 把 0 存在 v0

    .line 15  //这里已经是 finally 模块，++t;
    .local v0, "t":I  
    add-int/lit8 v1, v0, 0x1  // v0 + 1 存在 v1，所以并没有修改结果值

    .line 16  // System.out.println("final t:" + t);
    .end local v0    # "t":I
    .local v1, "t":I
    sget-object v2, Ljava/lang/System;->out:Ljava/io/PrintStream;

    new-instance v3, Ljava/lang/StringBuilder;

    invoke-direct {v3}, Ljava/lang/StringBuilder;-><init>()V  

    const-string v4, "final t:" //生成String "final t:"

    invoke-virtual {v3, v4}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder; //sb.append("final t:");

    move-result-object v3

    invoke-virtual {v3, v1}, Ljava/lang/StringBuilder;->append(I)Ljava/lang/StringBuilder;  //sb.append("final t:").append("{v0 + 1}");

    move-result-object v3

    invoke-virtual {v3}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v3

    invoke-virtual {v2, v3}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    .line 13  // return t;
    return v0 //v0 的值在开始的时候就赋值为 0，之后就没有修改过了，所以返回的还是 0
.end method
```

可以看到 **finally 块会先于 return 执行，但是 return 的值却是不受到 finally 块的影响**
