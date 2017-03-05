# 官方MultiDex源码分析

目的是为了解决65535问题，支持的SDK是4以上，低了会抛异常，Android5.0以上的虚拟机本来就可以支持Dex分包加载

主要原理，

## 流程分析

### 基本流程

1、校验（Vm是否已经支持分包如21+，最低SDK版本是4，是否已经分包过了）

2、清理旧的的dex分包的目录下文件，`data/data/packageName/file/secondary-dexes`

3、进行dex分包，存放目录`data/data/packageName/code_cache/secondary-dexes`

- 3.1 主要是读取apk压缩包下的的classes2.dex,..classesN.dex依次写入`/data/data/pkgName/code_cache/secondary-dexes/base.apk.classesN.zip`下

4、校验分包的dex压缩包是否有效，无效再进行一次分包

5、Dex压缩包文件安装加载，通过`DexPathList#makeDexElements`的方法进行dex的加载，用返回的`Element`数组扩充原来`ClassLoader`下的`Elements`实现加载

```java
public static void install(Context context) {
    if (IS_VM_MULTIDEX_CAPABLE) {
        Log.i(TAG, "VM has multidex support, MultiDex support library is disabled.");
        return;
    }
    if (Build.VERSION.SDK_INT < MIN_SDK_VERSION) {
        throw new RuntimeException("Multi dex installation failed. SDK " + Build.VERSION.SDK_INT + " is unsupported. Min SDK version is " + MIN_SDK_VERSION + ".");
    }
    try {
        ApplicationInfo applicationInfo = getApplicationInfo(context);
        if (applicationInfo == null) {
            // Looks like running on a test Context, so just return without patching.
            return;
        }
        synchronized (installedApk) {
            String apkPath = applicationInfo.sourceDir;
            if (installedApk.contains(apkPath)) {  //是否已经安装了
                return;
            }
            installedApk.add(apkPath);
            if (Build.VERSION.SDK_INT > MAX_SUPPORTED_SDK_VERSION) {
                //警告:高于20的可以使用内建的dex分包能力
            }
            /*
             */
            ClassLoader loader;
            try {
                loader = context.getClassLoader();
            } catch (RuntimeException e) {
                // 测试MockContext
                return;
            }
            if (loader == null) {
                // Robolectric tests
                return;
            }
            try {
              clearOldDexDir(context);  //清理应用内部文件存储目录（一般data/data/pkg-name/）下的secondary-dexes目录
            } catch (Throwable t) {
            }
            // data/data/packageName/code_cache/secondary-dexes
            File dexDir = new File(applicationInfo.dataDir, SECONDARY_FOLDER_NAME);
            List<File> files = MultiDexExtractor.load(context, applicationInfo, dexDir, false); //返回分包后的zip文件列表
            if (checkValidZipFiles(files)) {  //检验zip文件是否有效
                installSecondaryDexes(loader, dexDir, files);
            } else {
                //如果第一失败了，再进行一次相同的加载操作
            }
        }

    } catch (Exception e) {
    }
}
```

### 如何安装

`DexClassLoader`在构造的时候就会读取指定目录下的zip、dex、jar等文件，加载成`DexFile`，并构造成`Element`数组，记录在成员`pathList`下，以后类的加载都会尝试在这些`DexFile`中寻找，而在dex分包后，就需要自己把"新的dex的文件路径" 告诉`DexClassLoader`，这里以SDK19+为例子来说(对14,15,16,17and18来说区别在于`DexPathList#makeDexElements`方法签名的改变，4到13的改变稍微有点大，但现在也不会开发14以下的了就不细看了)

```java
private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files) {
    if (!files.isEmpty()) {
        if (Build.VERSION.SDK_INT >= 19) {
            V19.install(loader, files, dexDir);
        } else if (Build.VERSION.SDK_INT >= 14) {
            V14.install(loader, files, dexDir);
        } else {
            V4.install(loader, files);
        }
    }
}
```

主要用`DexPathList#makeDexElements`的方法进行dex的加载，用返回的`Element`数组扩充原来`ClassLoader`下的`Elements`

```java
private static final class V19 {

    private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory) {

        Field pathListField = findField(loader, "pathList");  //loader#pathList字段，DexPathList类型
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList<File>(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
        if (suppressedExceptions.size() > 0) {
            for (IOException e : suppressedExceptions) {
                Log.w(TAG, "Exception in makeDexElement", e);
            }
            //.....
        }
    }

    /**
     * {@code private static final dalvik.system.DexPathList#makeDexElements}.
     * 这个方法用来执行DexPathList#makeDexElements的方法输入需要加载的dex目录，返回`Element`数组
     */
    private static Object[] makeDexElements(
            Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) {
        Method makeDexElements = findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class,  ArrayList.class);
        return (Object[]) makeDexElements.invoke(dexPathList, files, optimizedDirectory,suppressedExceptions);
    }
}
```

### Dex读取

Dex的拆分在`MultiDexExtractor#load`方法进行

```java
MultiDexExtractor.java

static List<File> load(Context context, ApplicationInfo applicationInfo, File dexDir, boolean forceReload) throws IOException {
    final File sourceApk = new File(applicationInfo.sourceDir); //data/app/packageName/base.apk

    long currentCrc = getZipCrc(sourceApk); //返回一个crc32值，类似MD5?反正应该是获取一个文件的标志

    List<File> files;
    //检验安装文件是否发生了改变，如果是重新加载
    if (!forceReload && !isModified(context, sourceApk, currentCrc)) {
            //...
            files = loadExistingExtractions(context, sourceApk, dexDir);
            //...
    } else {
        files = performExtractions(sourceApk, dexDir);
        putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1); //dex分包情况记录在sp,以便下次可以根据别配加载
    }
    return files;
}

private static boolean isModified(Context context, File archive, long currentCrc) {
    SharedPreferences prefs = getMultiDexPreferences(context);//multidex.version
    return (prefs.getLong(KEY_TIME_STAMP, NO_VALUE) != getTimeStamp(archive)) || (prefs.getLong(KEY_CRC, NO_VALUE) != currentCrc);
}
```

主要来看怎么拆分dex，获取apk文件的名字`classesNdex`的`ZipEntry`，写入到文件`data/data/packageName/code_cache/secondary-dexes/base.apk.classesN.zip`，N为dex的数量，2开始。Android系统在启动app时只加载了第一个`Classes.dex`，其他的DEX需要我们人工进行安装

```java
private static List<File> performExtractions(File sourceApk, File dexDir) throws IOException {

    final String extractedFilePrefix = sourceApk.getName() + "classes"; //base.apk.classes

    // Ensure that whatever deletions happen in prepareDexDir only happen if the zip that
    // contains a secondary dex file in there is not consistent with the latest apk.  Otherwise,
    // multi-process race conditions can cause a crash loop where one process deletes the zip
    // while another had created it.
    prepareDexDir(dexDir, extractedFilePrefix); //删除非base.apk.classes为前缀的文件

    List<File> files = new ArrayList<File>();

    final ZipFile apk = new ZipFile(sourceApk); //data/app/packageName/base.apk
    try {
        int secondaryNumber = 2;

        ZipEntry dexFile = apk.getEntry("classes" + secondaryNumber + "dex"); //获取ZipEntry
        while (dexFile != null) {
            String fileName = extractedFilePrefix + secondaryNumber + "zip"; //base.classes2.zip，往后便是base.classes3.dex、base.classes4.dex、base.classesN.dex
            File extractedFile = new File(dexDir, fileName);   //data/data/packageName/code_cache/secondary-dexes/base.classes2.zip
            files.add(extractedFile);

            int numAttempts = 0;
            boolean isExtractionSuccessful = false;
            while (numAttempts < 3 && !isExtractionSuccessful) { //最多3次尝试
                numAttempts++;
                // Create a zip file (extractedFile) containing only the secondary dex file  (dexFile) from the apk.
                extract(apk, dexFile, extractedFile, extractedFilePrefix);  //ZipEntry写入到指定文件

                isExtractionSuccessful = verifyZipFile(extractedFile);  //是否是有效的zip文件

                // Log the sha1 of the extracted zip file
                if (!isExtractionSuccessful) {
                    // Delete the extracted file
                    extractedFile.delete();
                    //...
                }
            }
            if (!isExtractionSuccessful) {
                //...
            }
            secondaryNumber++;
            dexFile = apk.getEntry(DEX_PREFIX + secondaryNumber + DEX_SUFFIX);
        } //end while
    } finally {
        //..
    }

    return files;
}
```

删除`data/data/packageName/code_cache/secondary-dexes/`目录下所有非`base.apk.classes`开头的文件

```java
/**
 * This removes any files that do not have the correct prefix.
 */
private static void prepareDexDir(File dexDir, final String extractedFilePrefix) throws IOException {
    /* mkdirs() has some bugs, especially before jb-mr1 and we have only a maximum of one parent
     * to create, lets stick to mkdir().
     */
    File cache = dexDir.getParentFile();
    mkdirChecked(cache);  //`data/data/packageName/code_cache/`
    mkdirChecked(dexDir); //`data/data/packageName/code_cache/secondary-dexes/`

    // Clean possible old files
    FileFilter filter = new FileFilter() {

        @Override
        public boolean accept(File pathname) {
            return !pathname.getName().startsWith(extractedFilePrefix); //过滤base.apk.classes前缀的文件
        }
    };
    File[] files = dexDir.listFiles(filter);
    if (files == null) {
        return;
    }
    for (File oldFile : files) {
        if (!oldFile.delete()) {
            Log.w(TAG, "Failed to delete old file " + oldFile.getPath());
        } else {
            Log.i(TAG, "Deleted old file " + oldFile.getPath());
        }
    }
}
```

把`ZipEntry`写入文件，具体文件`data/data/packageName/code_cache/secondary-dexes/base.apk.classesN.zip`

```java
/**
* apk ： apk的压缩包文件
* dexFile : Apk文件zip解压后得到的从dex文件，classes2.dex…classesN.dex
* extractTo : data/data/packageName/code_cache/secondary-dexes/base.apk.classesN.zip
* extractedFilePrefix : base.apk.classes
*/
private static void extract(ZipFile apk, ZipEntry dexFile, File extractTo, String extractedFilePrefix) {

    InputStream in = apk.getInputStream(dexFile);
    ZipOutputStream out = null;
    File tmp = File.createTempFile(extractedFilePrefix, "zip", extractTo.getParentFile());
    try {
        out = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(tmp)));
        try {
            ZipEntry classesDex = new ZipEntry("classes.dex");
            // keep zip entry time since it is the criteria used by Dalvik
            classesDex.setTime(dexFile.getTime());
            out.putNextEntry(classesDex);

            byte[] buffer = new byte[BUFFER_SIZE];
            int length = in.read(buffer);
            while (length != -1) {
                out.write(buffer, 0, length);
                length = in.read(buffer);
            }
            out.closeEntry();
        } finally {
            out.close();
        }
        if (!tmp.renameTo(extractTo)) {
            //...
        }
    } finally {
        closeQuietly(in);
        tmp.delete(); // return status ignored
    }
}
```

## 参考

- [MultiDex安装过程源码分析](http://mouxuejie.com/blog/2016-06-11/multidex-source-analysis/)
- [Android编译及Dex过程源码分析](http://mouxuejie.com/blog/2016-06-21/multidex-compile-and-dex-source-analysis/)
- [Android分包MultiDex原理详解](http://blog.csdn.net/yzzst/article/details/48290701)
- [美团Android DEX自动拆包及动态加载简介](http://tech.meituan.com/mt-android-auto-split-dex.html)
