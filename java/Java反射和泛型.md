# JAVA 中的反射和泛型

`JAVA` 中的泛型只在编译的时候检查，防止错误输入，字节码阶段是去泛型的，所以可以用反射来绕过编译，用两个例子来进行说明

例子1

```java

public class TestReflect {

    public static void main(String[] args) {
        List<String> stringList = new ArrayList<String>();
        List<Integer> integersList = new ArrayList<Integer>();
        System.out.print(stringList.getClass() == integersList.getClass());//打印结果为 true

    }
}
```

查看其编译后的字节码

```java
.method public static main([Ljava/lang/String;)V
    //...

    .prologue
    .line 12  // List<String> stringList = new ArrayList<String>();
    new-instance v1, Ljava/util/ArrayList;  //新建 ArrayList 对象，并没泛型的影子

    invoke-direct {v1}, Ljava/util/ArrayList;-><init>()V  //执行 ArrayList 构造方法

    .line 13  //List<Integer> integersList = new ArrayList<Integer>();
    .local v1, "stringList":Ljava/util/List;, "Ljava/util/List<Ljava/lang/String;>;"
    new-instance v0, Ljava/util/ArrayList;  ////新建 ArrayList 对象，并没泛型的影子

    invoke-direct {v0}, Ljava/util/ArrayList;-><init>()V //执行 ArrayList 构造方法
    //...

    goto :goto_17
.end method
```

可见其字节码中，`List<Integer> integersList = new ArrayList<Integer>();` 这样的语句实际上还是新建一个 `ArrayList` 对象

例子2

```java

public class TestReflect {

    public static void main(String[] args) {
        List<String> stringList = new ArrayList<String>();

        Class clz = stringList.getClass();
        try {
            Method add = clz.getMethod("add", Object.class);
            add.invoke(stringList, 1);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        System.out.print("size:" + stringList.size()); //输出1，证明成功添加一个 int 类型对象到 List<String> 类型的列表

    }
}
```

## 参考

[Java 通过反射了解集合泛型的本本质](http://www.imooc.com/video/3738)
