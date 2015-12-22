# SharePreference和DiskLruCache的一些对比

## 初始化

### SharePreference
  `getSharedPreferences`用特定的模式包名目录下的特定文件名的文件，在新线程下使用系统下的`XmlUtils`来解析整个XML文件，并保存所有的数据到全局的Map，在解析过过程中如果去存取数据都会`wait`，解析完成就`notifyAll`，把所有数据读取并保存到全局的`Map`

### DiskLruCache
  `open`方法可以打开特定的目录、版本的缓存，对于信息过期的（一般是版本号），就删除原来的，其journal文件记录了版本信息和所有的操作记录
  ，通过读取journal文件，把所有数据读取并保存到全局的`LinkedHashMap`

## 存
### SharePreference
  如果正在解析文件，那么`wait`；否则，通过`EditorImpl`的`Map`类型成员变量，记录将要写入的`key-value`，`commit`方法是在主线程中使用系统下的`XmlUtils`写入数据，每次都是重新写入全局`Map`的数据，`apply`则是在单一线程的线程池中写入


### DiskLruCache
  每个key对应一个（可以多个，valueCount指定，读写的时候通过约束好的index来读写）文件，通过`Editor`来编辑某个KEY，`Editor`通过
  提供`OutputStream`和`String`两种方式来写特定`index`的文件或直接存`String`类型，最后`commit`方法来提交

## 取
### SharePreference
  很简单，如果正在解析文件，那么`wait`，否则直接从全局的Map读取

### DiskLruCache
  默认情况下，读到的是该KEY的某个`index`的文件的`InputStream`，需要什么数据可以自己解析，另外提供`String`类型的直接读取字符串

## 其他
### SharePreference


### DiskLruCache
  可以指定版本，前后版本不一致，会删除之前的journal文件，也就是重新新建缓存库；可以指定缓存大小，超过了就移除持久化最久的数据；其journal文件保存了
  了对缓存库的所有的操作，这样有助于在每次初始化解析的时候保持其`LRU`特性，在日志冗余信息行过多的情况下，会重新构建journal文件，根据现在保存的值，只记录`CLEAN`和'DIRTY'行（如果正在编辑）；初始化、读、写操作都是在UI线程的，如果数据量大，最好在子线程来处理；存取数据的灵活性比较高
