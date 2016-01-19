# XUitls源码解析之Db模块
## DbUtils对象
每个DbUtils实例主要功能是对某个表进行CRUD操作，各个表都有对应的DbUtils实例，存储在DbUtils的类变量
```java
    private static HashMap<String, DbUtils> daoMap = new HashMap<String, DbUtils>();
```
### 实例化
```java
    private DbUtils(DaoConfig config) {
        if (config == null) {
            throw new IllegalArgumentException("daoConfig may not be null");
        }
        this.database = createDatabase(config);//构造DbUtils时候根据DbConfig构造SQLiteDatabase对象
        this.daoConfig = config;
    }
    private synchronized static DbUtils getInstance(DaoConfig daoConfig) {
        DbUtils dao = daoMap.get(daoConfig.getDbName());
        if (dao == null) {
            dao = new DbUtils(daoConfig);
            daoMap.put(daoConfig.getDbName(), dao);
        } else {
            dao.daoConfig = daoConfig;
        }
        // 根据版本好跟新数据库
        SQLiteDatabase database = dao.database;
        int oldVersion = database.getVersion();
        int newVersion = daoConfig.getDbVersion();
        if (oldVersion != newVersion) {
            if (oldVersion != 0) {
                DbUpgradeListener upgradeListener = daoConfig.getDbUpgradeListener();
                if (upgradeListener != null) {
                    upgradeListener.onUpgrade(dao, oldVersion, newVersion);
                } else {
                    try {
                        dao.dropDb();//没有升级回调的话会删除之前的表
                    } catch (DbException e) {
                        LogUtils.e(e.getMessage(), e);
                    }
                }
            }
            database.setVersion(newVersion);
        }
        return dao;
    }
```

构造的时候调用的新建或打开SQLiteDatabase，可以指定路径，在DaoConfig中配置
```java
    private SQLiteDatabase createDatabase(DaoConfig config) {
        SQLiteDatabase result = null;
        String dbDir = config.getDbDir();
        if (!TextUtils.isEmpty(dbDir)) {
            File dir = new File(dbDir);
            if (dir.exists() || dir.mkdirs()) {
                File dbFile = new File(dbDir, config.getDbName());
                result = SQLiteDatabase.openOrCreateDatabase(dbFile, null);
            }
        } else {
            result = config.getContext().openOrCreateDatabase(config.getDbName(), 0, null);
        }
        return result;
    }
```
#### 事务相关 使用ReentrantLock
```java
private Lock writeLock = new ReentrantLock();
 private volatile boolean writeLocked = false;

 private void beginTransaction() {
     if (allowTransaction) {
         database.beginTransaction();
     } else {
         writeLock.lock();
         writeLocked = true;
     }
 }

 private void setTransactionSuccessful() {
     if (allowTransaction) {
         database.setTransactionSuccessful();
     }
 }

 private void endTransaction() {
     if (allowTransaction) {
         database.endTransaction();
     }
     if (writeLocked) {
         writeLock.unlock();
         writeLocked = false;
     }
 }
```
#### 建表

```java
public void createTableIfNotExist(Class<?> entityType) throws DbException {
    if (!tableIsExist(entityType)) {
        SqlInfo sqlInfo = SqlInfoBuilder.buildCreateTableSqlInfo(this, entityType);//SqlInfoBuilder对象用来建立具体的sql语句
        execNonQuery(sqlInfo);//SqlInfo对象则是用来存储sql语句和参数
        String execAfterTableCreated = TableUtils.getExecAfterTableCreated(entityType);
        if (!TextUtils.isEmpty(execAfterTableCreated)) {
            execNonQuery(execAfterTableCreated);
        }
    }
}
```
```java
public boolean tableIsExist(Class<?> entityType) throws DbException {
    Table table = Table.get(this, entityType);
    if (table.isCheckedDatabase()) {
        return true;
    }
    Cursor cursor = execQuery("SELECT COUNT(*) AS c FROM sqlite_master WHERE type='table' AND name='" + table.tableName + "'");
    if (cursor != null) {
        try {
            if (cursor.moveToNext()) {
                int count = cursor.getInt(0);
                if (count > 0) {
                    table.setCheckedDatabase(true);
                    return true;
                }
            }
        } catch (Throwable e) {
            throw new DbException(e);
        } finally {
            IOUtils.closeQuietly(cursor);
        }
    }

    return false;
}
```

#### 数据保存
支持单个实体，List对象，支持存在的时候使用更新替代插入，支持事务

#### 保存或更新一个对象

```java
    public void saveOrUpdate(Object entity) throws DbException {
        try {
            beginTransaction();
            createTableIfNotExist(entity.getClass());//建表
            saveOrUpdateWithoutTransaction(entity);
            setTransactionSuccessful();
        } finally {
            endTransaction();
        }
    }
```
用SqlInfoBuilder来构建插入语句，先把对象转换成KeyValue的List对象，构造如 insert into TableName(key1,key2,...)values(?,?....)的sql语句，对应的值顺序存在SqlInfo的bindArgs，sql存在SqlInfo的Sql
```java
private void saveOrUpdateWithoutTransaction(Object entity) throws DbException {
     Table table = Table.get(this, entity.getClass());
     Id id = table.id;
     if (id.isAutoIncrement()) {
         if (id.getColumnValue(entity) != null) {
             execNonQuery(SqlInfoBuilder.buildUpdateSqlInfo(this, entity));
         } else {
             saveBindingIdWithoutTransaction(entity);
         }
     } else {
         execNonQuery(SqlInfoBuilder.buildReplaceSqlInfo(this, entity));
     }
 }
```
执行具体的sql语句，由SqlInfo对象保存sql语句和对象的参数
```java
    public void execNonQuery(SqlInfo sqlInfo) throws DbException {
        debugSql(sqlInfo.getSql());
        try {
            if (sqlInfo.getBindArgs() != null) {
                database.execSQL(sqlInfo.getSql(), sqlInfo.getBindArgsAsArray());
            } else {
                database.execSQL(sqlInfo.getSql());
            }
        } catch (Throwable e) {
            throw new DbException(e);
        }
    }
```
##  Column

列注解包含列名和默认值
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String column() default "";
    String defaultValue() default "";
}
```
### ColumnUtils

构造Column的辅助类，__记录能识别的字段类型，所以如果需要存储自定义的对象，这里就需要作出修改__，
获取字段的setter/getter方法(boolean值则以is为前缀)

使用HashSet记录能识别的列类型
```java
//在静态构造块存储好支持的列类
static {
    DB_PRIMITIVE_TYPES.add(int.class.getName());
    DB_PRIMITIVE_TYPES.add(long.class.getName());
    DB_PRIMITIVE_TYPES.add(short.class.getName());
    DB_PRIMITIVE_TYPES.add(byte.class.getName());
    DB_PRIMITIVE_TYPES.add(float.class.getName());
    DB_PRIMITIVE_TYPES.add(double.class.getName());
    //.....省略.....
}
//判断是否是支持类型
public static boolean isDbPrimitiveType(Class<?> fieldType) {
      return DB_PRIMITIVE_TYPES.contains(fieldType.getName());
  }
```

#### Setter方法的获取

主要构造某个字段的Setter/Getter Method对象，__目的用来在从数据库读取/存储的时候为对象赋值/获取对象的值__，Getter类似,(boolean/Boolean 则以is为前缀)，
如果setter/getter不存在，会遍历所有的父类，如果还是找不到，就为NULL,__所以为了效率，最好还是加上setter/getter方法__

```java
public static Method getColumnSetMethod(Class<?> entityType, Field field) {
      String fieldName = field.getName();
      Method setMethod = null;
      if (field.getType() == boolean.class) {
          setMethod = getBooleanColumnSetMethod(entityType, field);//
      }
      if (setMethod == null) {
          String methodName = "set" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);
          try {
              setMethod = entityType.getDeclaredMethod(methodName, field.getType());
          } catch (NoSuchMethodException e) {
              LogUtils.d(methodName + " not exist");
          }
      }
      if (setMethod == null && !Object.class.equals(entityType.getSuperclass())) {
          return getColumnSetMethod(entityType.getSuperclass(), field);
      }
      return setMethod;
  }
```

### Column对象

Column记录了所属的表对象，类中的字段field，__类型转换器（下面会讲）__，列名，默认值和索引，当然还有对应的setter/getter方法，
另外还可以 __操作字段获取该字段的值和存储值__（经过转换器转换后的值）

@Id,@Foreign,@Finder都是Column的子类

#### 构造
```java
    /* package */ Column(Class<?> entityType, Field field) {
        this.columnField = field;
        this.columnConverter = ColumnConverterFactory.getColumnConverter(field.getType());//获取类型转换器
        this.columnName = ColumnUtils.getColumnNameByField(field);//获取列名
        //默认值设置，
        if (this.columnConverter != null) {
            this.defaultValue = this.columnConverter.getFieldValue(ColumnUtils.getColumnDefaultValue(field));
        } else {
            this.defaultValue = null;
        }
        this.getMethod = ColumnUtils.getColumnGetMethod(entityType, field);
        this.setMethod = ColumnUtils.getColumnSetMethod(entityType, field);
    }
```

#### 取值（存储对象某个字段值）
```java
@SuppressWarnings("unchecked")
public Object getColumnValue(Object entity) {
    Object fieldValue = getFieldValue(entity);
    return columnConverter.fieldValue2ColumnValue(fieldValue);
}
public Object getFieldValue(Object entity) {
    Object fieldValue = null;
    if (entity != null) {
        if (getMethod != null) {
            try {
                fieldValue = getMethod.invoke(entity);
            } catch (Throwable e) {
                LogUtils.e(e.getMessage(), e);
            }
        } else {
            try {
                this.columnField.setAccessible(true);
                fieldValue = this.columnField.get(entity);
            } catch (Throwable e) {
                LogUtils.e(e.getMessage(), e);
            }
        }
    }
    return fieldValue;
}
```

#### 改值,默认使用setter方法，如果没有就直接操作该字段
```java
@SuppressWarnings("unchecked")
public void setValue2Entity(Object entity, Cursor cursor, int index) {
    this.index = index;
    Object value = columnConverter.getFieldValue(cursor, index);
    if (value == null && defaultValue == null) return;
    if (setMethod != null) {
        try {
            setMethod.invoke(entity, value == null ? defaultValue : value);
        } catch (Throwable e) {
            LogUtils.e(e.getMessage(), e);
        }
    } else {
        try {
            this.columnField.setAccessible(true);
            this.columnField.set(entity, value == null ? defaultValue : value);
        } catch (Throwable e) {
            LogUtils.e(e.getMessage(), e);
        }
    }
}
```

## Table
表注解,包括表名和一个String,该String会在表创建后执行,

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Table {
    String name() default "";
    String execAfterTableCreated() default "";
}



```
### TableUtils
构建表对象（Table）的辅助类，通过注解找出对象的表名，对象的所有的列

#### 获取@Table标识的内容
```java
public static String getTableName(Class<?> entityType) {
    Table table = entityType.getAnnotation(Table.class);
    if (table == null || TextUtils.isEmpty(table.name())) {
        return entityType.getName().replace('.', '_');//如果没指定表名，使用全限定符的模式
    }
    return table.name();
}
public static String getExecAfterTableCreated(Class<?> entityType) {
    Table table = entityType.getAnnotation(Table.class);
    if (table != null) {
        return table.execAfterTableCreated();
    }
    return null;
}
```
#### 获取@Column标识的内容

#### 传递一个记录表结构的类，获取所有的列，以KEY为字段名VALUE为Column的HashMap的形式（除了字段名为id或者_id或者使用@Id注解的字段）存在KEY为表名的HashMap

```java
    /**
     * key: entityType.name <类名,<列名，Column>>
     */
    private static ConcurrentHashMap<String, HashMap<String, Column>> entityColumnsMap = new ConcurrentHashMap<String, HashMap<String, Column>>();

    /* package */
    static synchronized HashMap<String, Column> getColumnMap(Class<?> entityType) {
        if (entityColumnsMap.containsKey(entityType.getName())) {
            return entityColumnsMap.get(entityType.getName());
        }
        HashMap<String, Column> columnMap = new HashMap<String, Column>();
        String primaryKeyFieldName = getPrimaryKeyFieldName(entityType);//获取主键名id或者_id或者@Id注解时候指定的值
        addColumns2Map(entityType, primaryKeyFieldName, columnMap);
        entityColumnsMap.put(entityType.getName(), columnMap);
        return columnMap;
    }
    /**
    *找出所有可以操作（@Columu,@Foregin,@Finder）的字段并存储，前提是没@Transient和非静态字段
    */
    private static void addColumns2Map(Class<?> entityType, String primaryKeyFieldName, HashMap<String, Column> columnMap) {
        if (Object.class.equals(entityType)) return;
        try {
            Field[] fields = entityType.getDeclaredFields();
            for (Field field : fields) {
                //忽略带@Transient或者静态字段
                if (ColumnUtils.isTransient(field) || Modifier.isStatic(field.getModifiers())) {
                    continue;
                }
                //识别的优先级 @Id>@Column>@Foreign>@Finder 当前类>父类
                if (ColumnConverterFactory.isSupportColumnConverter(field.getType())) {
                    if (!field.getName().equals(primaryKeyFieldName)) {
                        Column column = new Column(entityType, field);
                        if (!columnMap.containsKey(column.getColumnName())) {
                            columnMap.put(column.getColumnName(), column);//非主键字段，存在columnMap中
                        }
                    }
                } else if (ColumnUtils.isForeign(field)) {//如果是外键关联
                    Foreign column = new Foreign(entityType, field);
                    if (!columnMap.containsKey(column.getColumnName())) {
                        columnMap.put(column.getColumnName(), column);//外键键字段，也存在columnMap中
                    }
                } else if (ColumnUtils.isFinder(field)) {
                    Finder column = new Finder(entityType, field);
                    if (!columnMap.containsKey(column.getColumnName())) {
                        columnMap.put(column.getColumnName(), column);//Finder字段，也存在columnMap中
                    }
                }
            }//end for
            //会把父类中的列也找出并存储
            if (!Object.class.equals(entityType.getSuperclass())) {
                addColumns2Map(entityType.getSuperclass(), primaryKeyFieldName, columnMap);
            }
        } catch (Throwable e) {
            LogUtils.e(e.getMessage(), e);
        }
    }
```

### 处理外键
可以看到，外键的类型

### 1对多处理 Filder


#### 传递一个记录表结构的类，获取ID对象

```java
/**
     * key: entityType.name
     */
    private static ConcurrentHashMap<String, com.lidroid.xutils.db.table.Id> entityIdMap = new ConcurrentHashMap<String, com.lidroid.xutils.db.table.Id>();
    //获取主键
    /* package */
    static synchronized com.lidroid.xutils.db.table.Id getId(Class<?> entityType) {
        if (Object.class.equals(entityType)) {
            throw new RuntimeException("field 'id' not found");
        }
        if (entityIdMap.containsKey(entityType.getName())) {
            return entityIdMap.get(entityType.getName());
        }
        Field primaryKeyField = null;
        Field[] fields = entityType.getDeclaredFields();
        if (fields != null) {
            //通过两种策略来查找主键，Id注解和field名（id或_id）,field名优先

            for (Field field : fields) {
                if (field.getAnnotation(Id.class) != null) {
                    primaryKeyField = field;
                    break;
                }
            }

            if (primaryKeyField == null) {
                for (Field field : fields) {
                    if ("id".equals(field.getName()) || "_id".equals(field.getName())) {
                        primaryKeyField = field;
                        break;
                    }
                }
            }
        }
        //如果没找到，还会从父类中查找
        if (primaryKeyField == null) {
            return getId(entityType.getSuperclass());
        }

        com.lidroid.xutils.db.table.Id id = new com.lidroid.xutils.db.table.Id(entityType, primaryKeyField);
        entityIdMap.put(entityType.getName(), id);//缓存
        return id;
    }

```

### Table对象
主要记录了表的名称，表的主键（Id对象），表的各个字段（Column对象）

####  私有的构造器
```java
    private Table(DbUtils db, Class<?> entityType) {
        this.db = db;
        this.tableName = TableUtils.getTableName(entityType);//获取@Table注释的表名
        this.id = TableUtils.getId(entityType);//取主键
        this.columnMap = TableUtils.getColumnMap(entityType);//获取表的各个字段

        finderMap = new HashMap<String, Finder>();//转存@Finder修饰的字段
        for (Column column : columnMap.values()) {
            column.setTable(this);
            if (column instanceof Finder) {
                finderMap.put(column.getColumnName(), (Finder) column);
            }
        }
    }
```

####  获取和移除表对象

```java
private static final HashMap<String, Table> tableMap = new HashMap<String, Table>();//记录
//获取表对象，缓存在tableMap中
public static synchronized Table get(DbUtils db, Class<?> entityType) {
        String tableKey = db.getDaoConfig().getDbName() + "#" + entityType.getName();
        Table table = tableMap.get(tableKey);
        if (table == null) {
            table = new Table(db, entityType);//构造表
            tableMap.put(tableKey, table);
        }

        return table;
    }
    //移除表对象
    public static synchronized void remove(DbUtils db, String tableName) {
    if (tableMap.size() > 0) {
        String key = null;
        for (Map.Entry<String, Table> entry : tableMap.entrySet()) {
            Table table = entry.getValue();
            if (table != null && table.tableName.equals(tableName)) {
                key = entry.getKey();
                if (key.startsWith(db.getDaoConfig().getDbName() + "#")) {
                    break;
                }
            }
        }
        if (TextUtils.isEmpty(key)) {
            tableMap.remove(key);
        }
    }
}
```

### 类型值转换
__类字段实际值和存储值的转换__，例如：如果是boolean，在存储的时候使用1/0来存，Data存储的是Integer
这里为我们所需对象的时候提供了扩展

#### 接口定义
```java
//数据库字段类型
public enum ColumnDbType {
    INTEGER("INTEGER"), REAL("REAL"), TEXT("TEXT"), BLOB("BLOB");
    private String value;
    ColumnDbType(String value) {
        this.value = value;
    }
    @Override
    public String toString() {
        return value;
    }
}

public interface ColumnConverter<T> {

    T getFieldValue(final Cursor cursor, int index);

    T getFieldValue(String fieldStringValue);//存储值转类字段值，获取的时候用到

    Object fieldValue2ColumnValue(T fieldValue);//类字段的值转成存储值，存储的时候用到

    ColumnDbType getColumnDbType();
}
// 例如
public class SqlDateColumnConverter implements ColumnConverter<java.sql.Date> {
    @Override
    public java.sql.Date getFieldValue(final Cursor cursor, int index) {
        return cursor.isNull(index) ? null : new java.sql.Date(cursor.getLong(index));
    }
    @Override
    public java.sql.Date getFieldValue(String fieldStringValue) {
        if (TextUtils.isEmpty(fieldStringValue)) return null;
        return new java.sql.Date(Long.valueOf(fieldStringValue));
    }
    @Override
    public Object fieldValue2ColumnValue(java.sql.Date fieldValue) {
        if (fieldValue == null) return null;
        return fieldValue.getTime();
    }
    @Override
    public ColumnDbType getColumnDbType() {
        return ColumnDbType.INTEGER;
    }
}

```
#### ColumnConverterFactory来获取ColumnConverter

使用工厂方法来获取转换器，在静态初始化的时候先初始化好Map对象，所以有自己的转换器，记得在这里加上

```java

    private static final ConcurrentHashMap<String, ColumnConverter> columnType_columnConverter_map;

    static {
        columnType_columnConverter_map = new ConcurrentHashMap<String, ColumnConverter>();

        BooleanColumnConverter booleanColumnConverter = new BooleanColumnConverter();
        columnType_columnConverter_map.put(boolean.class.getName(), booleanColumnConverter);
        columnType_columnConverter_map.put(Boolean.class.getName(), booleanColumnConverter);

        ByteArrayColumnConverter byteArrayColumnConverter = new ByteArrayColumnConverter();
        columnType_columnConverter_map.put(byte[].class.getName(), byteArrayColumnConverter);

        ByteColumnConverter byteColumnConverter = new ByteColumnConverter();
        columnType_columnConverter_map.put(byte.class.getName(), byteColumnConverter);
        columnType_columnConverter_map.put(Byte.class.getName(), byteColumnConverter);
        //````这里省略好多
    }
```
可以注册自己的转换器
```java
//增加自己的转换器，例如某个类转存成json，获取的时候json转对象
  public static void registerColumnConverter(Class columnType, ColumnConverter columnConverter) {
      columnType_columnConverter_map.put(columnType.getName(), columnConverter);
  }
```
## SqlInfoBuilder构建sql语句
用来构建真实的sql语句，可以搭配WhereBuilder和Selector使用

entity2KeyValueList方法，把表的列名-值转成KeyValue类型的列表
```java
public static List<KeyValue> entity2KeyValueList(DbUtils db, Object entity) {

     List<KeyValue> keyValueList = new ArrayList<KeyValue>();
     Class<?> entityType = entity.getClass();
     Table table = Table.get(db, entityType);//建立Table对象，Table对象初始化的过程已经初始化好了Id和columnMap
     Id id = table.id;

     if (!id.isAutoIncrement()) {
         Object idValue = id.getColumnValue(entity);
         KeyValue kv = new KeyValue(id.getColumnName(), idValue);
         keyValueList.add(kv);
     }
     Collection<Column> columns = table.columnMap.values();
     for (Column column : columns) {
         if (column instanceof Finder) {//忽略Finder列
             continue;
         }
         KeyValue kv = column2KeyValue(entity, column);
         if (kv != null) {
             keyValueList.add(kv);
         }
     }
     return keyValueList;
 }

 private static KeyValue column2KeyValue(Object entity, Column column) {
    KeyValue kv = null;
    String key = column.getColumnName();
    if (key != null) {
        Object value = column.getColumnValue(entity);
        value = value == null ? column.getDefaultValue() : value;
        kv = new KeyValue(key, value);
    }
    return kv;
}
```

### 建表

```java
public static SqlInfo buildCreateTableSqlInfo(DbUtils db, Class<?> entityType) throws DbException {
    Table table = Table.get(db, entityType);
    Id id = table.id;
    StringBuffer sqlBuffer = new StringBuffer();
    sqlBuffer.append("CREATE TABLE IF NOT EXISTS ");
    sqlBuffer.append(table.tableName);
    sqlBuffer.append(" ( ");

    if (id.isAutoIncrement()) {
        sqlBuffer.append("\"").append(id.getColumnName()).append("\"  ").append("INTEGER PRIMARY KEY AUTOINCREMENT,");
    } else {
        sqlBuffer.append("\"").append(id.getColumnName()).append("\"  ").append(id.getColumnDbType()).append(" PRIMARY KEY,");
    }
    Collection<Column> columns = table.columnMap.values();
    for (Column column : columns) {
        if (column instanceof Finder) {
            continue;
        }
        sqlBuffer.append("\"").append(column.getColumnName()).append("\"  ");
        sqlBuffer.append(column.getColumnDbType());
        if (ColumnUtils.isUnique(column.getColumnField())) {
            sqlBuffer.append(" UNIQUE");//Unique约束
        }
        if (ColumnUtils.isNotNull(column.getColumnField())) {//Not null约束
            sqlBuffer.append(" NOT NULL");
        }
        String check = ColumnUtils.getCheck(column.getColumnField());
        if (check != null) {
            sqlBuffer.append(" CHECK(").append(check).append(")");//自定义的check约束
        }
        sqlBuffer.append(",");
    }

    sqlBuffer.deleteCharAt(sqlBuffer.length() - 1);
    sqlBuffer.append(" )");
    return new SqlInfo(sqlBuffer.toString());
}
```
### 插入和更新

更新和插入类似
```java
public static SqlInfo buildInsertSqlInfo(DbUtils db, Object entity) throws DbException {

    List<KeyValue> keyValueList = entity2KeyValueList(db, entity);
    if (keyValueList.size() == 0) return null;

    SqlInfo result = new SqlInfo();
    StringBuffer sqlBuffer = new StringBuffer();

    sqlBuffer.append("INSERT INTO ");
    sqlBuffer.append(TableUtils.getTableName(entity.getClass()));
    sqlBuffer.append(" (");
    for (KeyValue kv : keyValueList) {
        sqlBuffer.append(kv.key).append(",");
        result.addBindArgWithoutConverter(kv.value);
    }
    sqlBuffer.deleteCharAt(sqlBuffer.length() - 1);
    sqlBuffer.append(") VALUES (");
    int length = keyValueList.size();
    for (int i = 0; i < length; i++) {
        sqlBuffer.append("?,");
    }
    sqlBuffer.deleteCharAt(sqlBuffer.length() - 1);
    sqlBuffer.append(")");

    result.setSql(sqlBuffer.toString());

    return result;
}
```
### 更新

可以搭配whereBuilder使用
```java
public static SqlInfo buildUpdateSqlInfo(DbUtils db, Object entity, WhereBuilder whereBuilder, String... updateColumnNames) throws DbException {

       List<KeyValue> keyValueList = entity2KeyValueList(db, entity);
       if (keyValueList.size() == 0) return null;

       HashSet<String> updateColumnNameSet = null;
       if (updateColumnNames != null && updateColumnNames.length > 0) {
           updateColumnNameSet = new HashSet<String>(updateColumnNames.length);
           Collections.addAll(updateColumnNameSet, updateColumnNames);
       }

       Class<?> entityType = entity.getClass();
       String tableName = TableUtils.getTableName(entityType);

       SqlInfo result = new SqlInfo();
       StringBuffer sqlBuffer = new StringBuffer("UPDATE ");
       sqlBuffer.append(tableName);
       sqlBuffer.append(" SET ");
       for (KeyValue kv : keyValueList) {
           if (updateColumnNameSet == null || updateColumnNameSet.contains(kv.key)) {
               sqlBuffer.append(kv.key).append("=?,");
               result.addBindArgWithoutConverter(kv.value);
           }
       }
       sqlBuffer.deleteCharAt(sqlBuffer.length() - 1);
       if (whereBuilder != null && whereBuilder.getWhereItemSize() > 0) {
           sqlBuffer.append(" WHERE ").append(whereBuilder.toString());
       }

       result.setSql(sqlBuffer.toString());
       return result;
   }
```
