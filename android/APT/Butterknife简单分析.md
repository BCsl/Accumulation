# 关键字

编译时注解、AbstractProcessor

## 使用的项目

ButterKnife、Dagger2等

### 优点

- 1.让模板式的代码更少，减少重复工作
- 2.让代码更加简洁

## 实现

### 1\. 注解的定义

```java
@Retention(CLASS) @Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```

### 2\. 实现AbstractProcessor

```java
//需要在配置文件中指定处理类 resources/META-INF/services/javax.annotation.processing.Processor
//或者使用auto-service库
public final class ButterKnifeProcessor extends AbstractProcessor {

  /**
   * 初始化一些辅助类
   */
  @Override
  public synchronized void init(ProcessingEnvironment env) {
      super.init(env);
      String sdk = env.getOptions().get(OPTION_SDK_INT);
      if (sdk != null) {
          try {
              this.sdk = Integer.parseInt(sdk);
          } catch (NumberFormatException e) {
              //...
          }
      }
      elementUtils = env.getElementUtils(); //跟元素相关的辅助类，帮助我们去获取一些元素相关的信息
      typeUtils = env.getTypeUtils(); //
      filer = env.getFiler(); // 跟文件相关的辅助类，生成JavaSourceCode.
      try {
          trees = Trees.instance(processingEnv);
      } catch (IllegalArgumentException ignored) {
      }
  }


  /**
   * 返回支持的注解类型
   */
  @Override
  public Set<String> getSupportedAnnotationTypes() {
      Set<String> types = new LinkedHashSet<>();
      for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
          types.add(annotation.getCanonicalName());
      }
      return types;
  }

  private Set<Class<? extends Annotation>> getSupportedAnnotations() {
      Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();
      //...
      annotations.add(BindView.class);
      return annotations;
  }

  @Override
  public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
      //....
      return true;
  }
```

其中`process`方法必须实现，**两个步骤**

- 收集相关注解信息
- 生成代理类

[Demo](https://github.com/zhuyongit/ViewInjectDemo)中写得比较清晰

#### 获取某个注解的Element

```java
  Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(BindView.class);
```

这里的Element有几个子类

- VariableElement //一般代表成员变量
- ExecutableElement //一般代表类中的方法
- TypeElement //一般代表类类型 ， VariableElement#getEnclosingElement可得
- PackageElement //一般代表Package

#### 生成代理类

通过`ProcessingEnvironment#getFiler()`得到的对象来写文件，通常来说代理类的全路径名是`PackageName + className+ 自定义的后缀`，根据全路径名，然后获取Writer对象

```java
JavaFileObject sourceFile = mFileUtils.createSourceFile(proxyInfo.getProxyClassFullName(), proxyInfo.getTypeElement());
          Writer writer = sourceFile.openWriter();
          writer.write(proxyInfo.generateJavaCode()); //生成完整的.java文件
          writer.flush();
          writer.close();
```

[简化类库](https://github.com/square/javapoet)

## 参考

[Android 如何编写基于编译时注解的项目](http://blog.csdn.net/lmj623565791/article/details/51931859)

[Butterknife深入剖析，自己实现Butterknife](http://www.jianshu.com/p/003be1b75e28)
