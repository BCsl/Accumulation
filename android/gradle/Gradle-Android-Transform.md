# Transform API 的使用

根据 [gradle transform](http://tools.android.com/tech-docs/new-build-system/transform-api) 中提到，Android Gradle plugin 在 1.5 开始允许第三方 Gradle 插件可以在 .class 文件打包成 dex 文件前操控 .class 文件

目的是简化自定义 class 注入，而不再需要操作 Task，而且过程也变得更灵活了

怎么使用？实现 Transform 类，并注册 `project.android.registerTransform(yourTransform)`

## Transform

实现 Transform 类

```java
public class PitchingTransform extends Transform {

    Project mProject;

    PitchingTransform(Project project) {
        mProject = project;
    }

    /**
     * {@link TransformManager#getTaskNamePrefix}
     * @return
     */
    @Override
    String getName() {
        return "Pitching"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS //处理的输入数据类型为 classes
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT; //作用域为整个项目
    }

    @Override
    boolean isIncremental() {
        return false    //是否支持增量编译
    }

    public void transform(@NonNull TransformInvocation transformInvocation)
            throws TransformException, InterruptedException, IOException {
              //只是简单的把输入文件拷贝到输出文件夹，以保证下一个 transform 可以正确运行
              transformInvocation.getInputs().forEach(new Consumer<TransformInput>() {
                  @Override
                  void accept(TransformInput transformInput) {
                      transformInput.getJarInputs().forEach(new Consumer<JarInput>() {
                          @Override
                          void accept(JarInput jarInput) {
                              mProject.logger.error("jarInput:" + jarInput.getFile().getAbsolutePath());
                              File destDir = transformInvocation.outputProvider.getContentLocation(jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR);
                              mProject.logger.error("jarInput->outputPath:" + destDir.getAbsolutePath().toString());
                              FileUtils.copyFile(jarInput.file, destDir)
                          }
                      });

                      transformInput.getDirectoryInputs().forEach(new Consumer<DirectoryInput>() {
                          @Override
                          void accept(DirectoryInput directoryInput) {
                              mProject.logger.error("DirectoryInput:" + directoryInput.getFile().getAbsolutePath());
                              File destDir = transformInvocation.outputProvider.getContentLocation(directoryInput.name, directoryInput.contentTypes, directoryInput.scopes, Format.DIRECTORY);
                              mProject.logger.error("DirectoryInput->outputPath:" + destDir.getAbsolutePath().toString());
                              FileUtils.copyFile(directoryInput.file, destDir)
                          }
                      });
                  }
              })
    }
}
```

**处理类型和作用域**

其中 ContentType 决定处理的数据类型，可以是 `CLASSES` 和 `RESOURCES` 这两种类型的组合

ScopeType 确定 transform 的作用域，6 种作用域可以选，`PROJECT`、`SUB_PROJECTS`、`PROJECT_LOCAL_DEPS`、`EXTERNAL_LIBRARIES`、`PROVIDED_ONLY`、`TESTED_CODE`

> getInputTypes 和 getScopes 都返回集合，TransformManager 中封装了默认的几种集中类型

![Transform 的工作流程](./img/Transform.png)

可以看到，Transform 需要处理 Input 并导出 Output 作为下一个 Transform 的输入。而最佳实践则是有多少的输入就应该有多少输出，其中输出路径通过 `TransformOutputProvider.getContentLocation` 方法来获取，方法的具体实现如下：

```groovy
public static File getContentLocation(
        @NonNull File rootLocation,
        @NonNull String name,
        @NonNull Set<ContentType> types,
        @NonNull Set<? super Scope> scopes,
        @NonNull Format format) {
    // runtime check these since it's (indirectly) called by 3rd party transforms.
    // ...null checker...
    switch (format) {
        case DIRECTORY: {
            File location = FileUtils.join(rootLocation,
                    FOLDERS,
                    typesToString(types),
                    scopesToString(scopes),
                    name);
            return location;
        }
        case JAR: {
            File location = FileUtils.join(rootLocation,
                    JARS,
                    typesToString(types),
                    scopesToString(scopes),
                    name + DOT_JAR);
            return location;
        }
        default:
            throw new RuntimeException("Unexpected Format: " + format);
    }
}
```

输入输出的格式如下：

```java
//JAR
jarInput:/Users/liusubing/.android/build-cache/8ca2423aab75400e7892acfc55662c1d7b0dbbc0/output/jars/classes.jar
jarInput->outputPath:/Users/liusubing/Project/github/tinkerTest/HotFixDemo/app/build/intermediates/transforms/Pitching/dev/debug/jars/1/10/6fa056d336664c43d91a6be7d9a790c08e39bf3c.jar
//DIRECTORY
DirectoryInput:/Users/liusubing/Project/github/tinkerTest/HotFixDemo/app/build/intermediates/classes/dev/debug
DirectoryInput->outputPath:/Users/liusubing/Project/github/tinkerTest/HotFixDemo/app/build/intermediates/transforms/Pitching/dev/debug/folders/1/1/5bbe0afa1e4e4b1edbf487eaf43304e330168757
```

# 参考

- [如何理解 Transform API](http://www.jianshu.com/p/37df81365edf)
