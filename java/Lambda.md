# Lambda教程

## OverView

* [创建方法用来匹配指定特点的成员](#创建方法用来匹配指定特点的成员)
* [创建更加通用的方法](#创建更加通用的方法)
* [在本地类中指定搜索条件](#在本地类中指定搜索条件)
* [使用匿名类指定搜索条件](#使用匿名类指定搜索条件)
* [使用lambda表达式搜索条件](#使用lambda表达式搜索条件)
* [使用lambda表达式的标准的函数](#使用lambda表达式的标准的函数)


想象一下，有这样的一个需求，需要从`List<Person>`队列中查找出特定的`Person`成员
```java

public class Person {

    public enum Sex {
        MALE, FEMALE
    }

    String name;
    LocalDate birthday;
    Sex gender;
    String emailAddress;

    public int getAge() {
        // ...
    }

    public void printPerson() {
        // ...
    }
}

```


## 创建方法用来匹配指定特点的成员

```java
public static void printPersonsOlderThan(List<Person> roster, int age) {
    for (Person p : roster) {
        if (p.getAge() >= age) {
            p.printPerson();
        }
    }
}
```
## 创建更加通用的方法

```java
public static void printPersonsWithinAgeRange(
    List<Person> roster, int low, int high) {
    for (Person p : roster) {
        if (low <= p.getAge() && p.getAge() < high) {
            p.printPerson();
        }
    }
}
```

## 在类中指定搜索条件

```java
public static void printPersons(
    List<Person> roster, CheckPerson tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
}
```

`CheckPerson`接口定义

```java
interface CheckPerson {
    boolean test(Person p);
}
```

`CheckPerson`实现

```java
class CheckPersonEligibleForSelectiveService implements CheckPerson {
    public boolean test(Person p) {
        return p.gender == Person.Sex.MALE &&
            p.getAge() >= 18 &&
            p.getAge() <= 25;
    }
}
```

使用

```java
public static void printPersons(
    roster, new CheckPersonEligibleForSelectiveService());
```

## 使用匿名内部类指定搜索条件

```java
public static void printPersons(
    roster,
    new CheckPerson() {
        public boolean test(Person p) {
            return p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25;
        }
    }
);
```

使用匿名内部类现对上面来说减少了不少代码，因为不用每次查找的时候都要新建一个类，然而考虑这个接口只有一个方法，这样看起来有点臃肿，这种情况下就可以考虑使用Lambda表达式来替代匿名内部类

## 使用lambda表达式搜索条件

`CheckPerson`只是一个__功能性接口__，功能性接口通常只有一个抽象方法（功能性接口可以包含多个默认方法或者静态方法）。因为这样，所以可以在实现的时候省略方法名。
如下使用lambda表达式来代替匿名内部类：

```java
public static void printPersons(
    roster,
    (Person p) -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25
);
```

## 使用lambda表达式实现标准的功能性接口

重新考虑下接口`CheckPerson`

```java
interface CheckPerson {
    boolean test(Person p);
}
```
这个功能性接口只有一个方法，并且参数也只有一个和返回一个Boolean的结果。这样的接口好简单，JDK默认提供了几个功能性接口，可以在`java.util.function`下找到。
例如，可以使用`Predicate<T>`接口替代`CheckPerson`：

```java
interface Predicate<T> {
    boolean test(T t);
}
```
方法定义

```java
public static void printPersonsWithPredicate(
    List<Person> roster, Predicate<Person> tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
}
```

使用Lambda表达式

```java
printPersonsWithPredicate(
    roster,
    p -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25
);
```


## 在你的应用中使用Lambda

重新考虑如下方法，想想可以在哪些地方使用Lambda表达式

```java
public static void printPersonsWithPredicate(
    List<Person> roster, Predicate<Person> tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
}
```

记住要使用Lambda表达式， __一定__ 要实现带一个抽象方法的__功能性接口__。
下面的方法，使用`Consumer<Person>`接口的实例的`accept`方法替代`printPerson`方法来打印`Person`:

```java
public static void processPersons(
    List<Person> roster,
    Predicate<Person> tester,
    Consumer<Person> block) {
        for (Person p : roster) {
            if (tester.test(p)) {
                block.accept(p);//简单地代替打印方法
            }
        }
}
```

以下方法使用Lambda表达式打印

```java
processPersons(
     roster,
     p -> p.getGender() == Person.Sex.MALE
         && p.getAge() >= 18
         && p.getAge() <= 25,
     p -> p.printPerson()
);
```

假设你想验证`Person`的有效性或者获取它的联系方式？你可以需要一个这样功能性接口`Function<T,R> `并有一个方法`R apply(T t)`

```java
public static void processPersonsWithFunction(
    List<Person> roster,
    Predicate<Person> tester,
    Function<Person, String> mapper,
    Consumer<String> block) {
    for (Person p : roster) {
        if (tester.test(p)) {
            String data = mapper.apply(p);//获取某种信息
            block.accept(data); //打印信息
        }
    }
}

```
如下则是使用Lambda表达式的效果

```java
processPersonsWithFunction(
    roster,
    p -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25,
    p -> p.getEmailAddress(),
    email -> System.out.println(email)
);
```

## 广泛地使用泛型
`processPersonsWithFunction`的泛型化版本

```java
public static <X, Y> void processElements(
    Iterable<X> source,
    Predicate<X> tester,
    Function <X, Y> mapper,
    Consumer<Y> block) {
    for (X p : source) {
        if (tester.test(p)) {
            Y data = mapper.apply(p);
            block.accept(data);
        }
    }
}
```

使用Lambda表达式的方式和上一节一样

## 使用接受Lambda表达式的聚合的方法

下面的例子使用聚合操作打印名单中特定角色的电子邮件地址

```java
roster
    .stream()
    .filter(
        p -> p.getGender() == Person.Sex.MALE
            && p.getAge() >= 18
            && p.getAge() <= 25)
    .map(p -> p.getEmailAddress())
    .forEach(email -> System.out.println(email));
```

## Lambda语法

**1.一个在括号内已逗号分隔的形式参数**

例如`CheckPerson`接口的`test`方法包含一个参数,p代表Person类的一个实例。
___注意：在Lambda表达式中可以省略数据类型，另外如果只有一个参数,可以省略括号___
    例如:

```java
p -> p.getGender() == Person.Sex.MALE
    && p.getAge() >= 18
    && p.getAge() <= 25
```

**2.->标志**

**3.Body,只有一句表达式或语句块**

    例如：

```java
p.getGender() == Person.Sex.MALE
    && p.getAge() >= 18
    && p.getAge() <= 25
```
如果你使用**一个表达式**，JVM会计算表达式的值并返回。当然，你还是可以使用return，
return声明并**不是一个表达式**，在Lambda中，你必须使用{}来围绕声明。

```java
p -> {
    return p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25;//可以去掉return，{}也可以去掉
}
```
如果返回类型是Void，可以忽略{}，如：

```java
email -> System.out.println(email)
```

完整实例

```java
public class Calculator {

    interface IntegerMath {
        int operation(int a, int b);   
    }

    public int operateBinary(int a, int b, IntegerMath op) {
        return op.operation(a, b);
    }

    public static void main(String... args) {

        Calculator myApp = new Calculator();
        IntegerMath addition = (a, b) -> a + b; //一个在括号内已逗号分隔的形式参数和一个表达式
        IntegerMath subtraction = (a, b) -> a - b;
        System.out.println("40 + 2 = " +
            myApp.operateBinary(40, 2, addition));
        System.out.println("20 - 10 = " +
            myApp.operateBinary(20, 10, subtraction));    
    }
}
```
## 访问局部变量

和本地方法和匿名内部方法一样，可以访问外部变量，它们有相同的访问局部变量的能力。但是Lambda表达式
