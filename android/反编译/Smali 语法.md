# Smali 语法

推荐使用 `Idea` + [java2smali](https://github.com/ollide/intellij-java2smali) 插件来编译和查看，很方便

## 条件和循环

原 java 文件

```java
public class TestSmali {

    private int accumulate(int a) {
        if (a <= 0) {
            return 0;
        }
        int sum = 0;
        for (int i = 0; i <= a; i++) {
            sum += a;
        }
        return sum;
    }
}
```

生成的 smali 文件

```smali

.class public Lcom/downloader/video/tumblr/TestSmali;
.super Ljava/lang/Object;
.source "TestSmali.java"

# 默认的无参构造方法
# direct methods
.method public constructor <init>()V
    .registers 1

    .prologue #方法开始
    .line 6
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

    return-void
.end method

.method private accumulate(I)I
    .registers 4
    .param p1, "a"    # I   #输入参数 a，类型为 int，并复制到 p1

    .prologue
    .line 9             # if (a <= 0)
    if-gtz p1, :cond_4  #条件判断，p1 是否大于 0，则跳到 :cond_4 的位置，即 .line 12

    .line 10            # return 0;
    const/4 v1, 0x0     # 0 赋值到 v1，32 位

    .line 16            # return sum;
    :cond_3
    return v1           # 返回 v1，即返回 0

    .line 12            # int sum = 0;
    :cond_4
    const/4 v1, 0x0     # 0 赋值到 v1，32 位

    .line 13            # for (int i = 0; i <= a; i++)
    .local v1, "sum":I  # 定义一个 int 类型变量 sum，用 v1 来初始化
    const/4 v0, 0x0     # 0 赋值到 v0，32 位

    .local v0, "i":I    # 定义一个 int 类型变量 i，用 v1 来初始化
    :goto_6
    if-gt v0, p1, :cond_3 # 如果 v0 的值大于 p1 的值，跳到 cond_3，即返回 v1 的值，结束循环

    .line 14            # sum += a;
    add-int/2addr v1, p1   # 计算 v1 = v1 + p1

    .line 13            # for (int i = 0; i <= a; i++)
    add-int/lit8 v0, v0, 0x1  # 计算 v0 = v0 + 1，即 i++

    goto :goto_6        # 跳到 :goto_6 的位置，继续循环
.end method
```
