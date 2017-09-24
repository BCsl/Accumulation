# Gradle 笔记 - Groovy

## 初识 [Groovy](http://www.groovy-lang.org/differences.html)

基于并扩展了 java ，运行在 JVM 上，说白了就是把写 Java 程序变得像写脚本一样简单，和 [java 的一些差别](http://www.groovy-lang.org/differences.html)

默认导入的包

```
java.io.* ;java.lang.* ;java.math.BigDecimal ;java.math.BigInteger ;java.net.* ;java.util.* ;groovy.lang.* ;groovy.util.*
```

### 基础

- 变量定义

和 js 一样，动态语言，使用关键字 `def` 定义（可用可不用），并且在定义变量的时候指定类型

```groovy
def var1 = "Hello World " ;  //def 可用可不用
// def String var1 = "Hello World !!!" ;  // 可以指定类型
println var1 +",type :"+ var1.getClass().getName(); //实际上是 java 的 String 类
```

编译输出，实际是 Java 中的 String 类

```
$ groovy Hello.groovy
Hello World ,type :java.lang.String
```

- 函数(方法)定义

def 同样可以省略，返回值和参数类型可以不指定类型，和 js 很像，且默认返回都是最后一个代码的执行结果

```groovy
def testResturn(arg){
  println "testResutn1 invole";
  return arg + " test resturn";
}

testResutn("Hello")
// testResutn "Hello" //还可以不带括号。。。但不推荐，容易出错
```

- 字符串

```groovy

def singleQuote='I am $ dolloar' //输出就是 I am $ dolloar
println singleQuote

def doubleQuoteWithoutDollar = "I am one dollar" //输出 I am one dollar
println doubleQuoteWithoutDollar

def x = 1
def doubleQuoteWithDollar = "I am $x dolloar" //输出 I am 1 dolloar
println doubleQuoteWithDollar

//三个引号'''xxx'''中的字符串支持随意换行
def multieLines = ''' begin
line 1
line 2
end
'''
println multieLines
```

编译输出

```
$ groovy string.groovy
I am $ dolloar
I am one dollar
I am 1 dolloar
 begin
line 1
line 2
end
```

### Groovy 的数据类型

- 基本数据类型

  int，boolean 这些 Java 中 的基本数据类型，在 Groovy 代码中其实对应的是它们的包装数据类型。比如 int 对应为 Integer，boolean 对应为 Boolean

- 容器类型

  - [List](http://www.groovy-lang.org/groovy-dev-kit.html#Collections-Lists)（对应 Java 中的List接口）

    类似 js 中的 Array 一样定义

    ```groovy
    def aList = [5,'string',true] //List 由[]定义，其元素可以是任何对象
    ```

  - [Map](http://www.groovy-lang.org/groovy-dev-kit.html#Collections-Maps)（底层对应 Java 中的 LinkedHashMap ）

    ```groovy
    def aMap = ['key1':'value1','key2':true]  //key 必须是字符串
    def aConfusedMap=[(key1):"who am i?"] //如果 key 是变量，使用小括号或者双引号加$的形式
    ```

  - [Range](http://www.groovy-lang.org/groovy-dev-kit.html#Collections-Ranges)

    ```groovy
    def aRange = 1..5
    def aRangeWithoutEnd = 1..<5
    println aRange.from //实际上等于调用 aRange.getFrom()
    println aRange.to   //实际上等于调用 aRange.getTo()
    ```

  - [闭包](http://www.groovy-lang.org/closures.html)

    和 js 中的闭包不太一样，JS 中的闭包是函数 + 函数内部能访问到的变量（也叫环境）的总和，组成闭包。这里的闭包则是一段可执行的代码，有点像函数的指针

    ```groovy
        //闭包定义
       def closure = {param -> println "say $param"};  //带参数的
       def closureNoParam = { println "Hello World"};    //这种case不需要->符号

       //闭包使用
       closure.call('Hello');
       closure("World");

       //隐含参数 it
       def greeting = { "Hello, $it!" }
       assert greeting('Patrick') == 'Hello, Patrick!'

       //最后一个参数是 closure 和可以省略 ()
       def lastClosure(first, closure){
           println 'first param'+first;
           closure(first);
       }
       lastClosure 'HeHe' ,{param -> println "invole closure with $param";}
    ```

### Groovy 高级用法

- 脚本和类

  可以像 java 一样定义类，默认

  ```groovy

  class Test{

  def name; //可以添加 @PackageScope 修饰符来使变量包级私有

  Test(name){
    this.name = name;
  }
  def sayName(){
     println "Hello my name is $name";
   }
  }

  Test tester =new Test("Tester1");
  println tester.name; tester.sayName();
  ```

  使用 `groovyc -d classes xxx.groovy` 命令可以把 groovy 文件转成 class 文件，得到两个文件 `Class.class` 和 `Test.class` 分别对应上面的脚本执行的环境和 Test 对象

  Test 对象的 java 层表现如下

  ```java
  public class Test implements GroovyObject {
  private Object name;  //默认为 private
  public Test(Object arg1)
  {
    Object name;
    CallSite[] arrayOfCallSite = $getCallSiteArray();
    MetaClass localMetaClass = $getStaticMetaClass();
    this.metaClass = localMetaClass;
    Object localObject1 = name;
    this.name = localObject1;
  }

  public Object sayName()
  {
    CallSite[] arrayOfCallSite = $getCallSiteArray();
    return arrayOfCallSite[0].callCurrent(this, new GStringImpl(new Object[] { this.name }, new String[] { "Hello my name is ", "" }));
    return null;
  }
  //默认为变量生成 setter 和 getter 方法
  public Object getName()
  {
    return this.name;
  }

  public void setName(Object paramObject)
  {
    this.name = paramObject;
  }
  }
  ```

  脚本执行的类，转化成 java 后，需要把 main 方法为入口

  ```java
  public class class extends Script {
  public class() {}

  public class(Binding context)
  {
    super(context);
  }

  public static void main(String... args)
  {
    CallSite[] arrayOfCallSite = $getCallSiteArray();
    arrayOfCallSite[0].call(InvokerHelper.class, class.class, args);
  }

  public Object run()
  {
    CallSite[] arrayOfCallSite = $getCallSiteArray();
    Test tester = (Test)ScriptBytecodeAdapter.castToType(arrayOfCallSite[1].callConstructor(Test.class, "Tester1"), Test.class);
    arrayOfCallSite[2].callCurrent(this, arrayOfCallSite[3].callGroovyObjectGetProperty(tester));
    return arrayOfCallSite[4].call(tester);return null;
  }
  }
  ```

- IO 操作

  虽然你可以在使用 Groovy 的时候使用 JAVA 的标准的 API ,但[GDK](http://www.groovy-lang.org/groovy-dev-kit.html) 提供了大量的工具类来进行 IO 操作

  ```groovy

  def file =new File("./Hello.groovy");
  //逐行读取文件
  file.eachLine 'utf-8',{line,lineNo -> println "$lineNo : $line"}
  ```

- XML 操作

  XmlSlurper

  ```groovy
  import groovy.util.slurpersupport.GPathResult;

  def manitest = new XmlSlurper();
  //提供多个 parse 的重载方法
  GPathResult result = manitest.parse("./AndroidManifest.xml"); //返回 GPathResult对象

  println result.name();  //当前的根节点的名字
  println result.size();  //节点数量
  println result.application.activity.size(); //activity 节点数量
  println result.application.activity[0]['@android:name'];  //获取 application 节点下的第一个 activity 的名字
  ```
