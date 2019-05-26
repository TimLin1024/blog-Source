---
title: 拆轮子系列——LitePal 原理解析
date: 2017-12-03 10:36:11
tags: 
- 源码分析
- 框架原理
categories: 
- 源码分析
- 框架原理
---



## LitePal 解决了什么问题？

>   LitePal 采取的是对象关系映射(ORM)的模式，那么什么是对象关系映射呢？简单点说，我们使用的编程语言是面向对象语言，而我们使用的数据库则是关系型数据库，那么将面向对象的语言和面向关系的数据库之间建立一种映射关系，这就是对象关系映射了。

>   但是我们为什么要使用对象关系映射模式呢？这主要是因为大多数的程序员都很擅长面向对象编程，但其中只有少部分的人才比较精通关系型数据库。而且数据库的 SQL 语言晦涩难懂，就算你很精通它，恐怕也不喜欢经常在代码中去写它吧？而对象关系映射模式则很好地解决了这个问题，它允许我们使用面向对象的方式来操作数据库，从而可以从晦涩难懂的 SQL 语言中解脱出来。

<!--more-->

Android 已经为我们提供了 SQLiteOpenHelper 以帮助我们完成数据库的创建、表的创建、以及数据的 CRUD 操作。但是使用起来还是不够方便。建立数据库需要继承 SQLiteOpenHelper 重写相应的方法，同时为了方便后续的操作，我们每创建一张表，都要一个相应的 DAO 类，完成相应表的 CRUD 操作。加在一起工作量真的不小。

如果使用 LitePal，只需要建立一个 xml 配置文件，在其中指定数据库名称、版本号，以及需要进行映射的类名，然后让 App 的 Application 类继承 LitePalApplication 进行初始化，就可以完成数据库、表的创建工作。同时，数据的 CRUD 操作只需要通过 DataSupport 的相应方法就能完成。这样工作量是不是少了很多。



## 初始化

文档中建议通过将在 Manifest 文件中指定 application 为 LitePalApplication  或者通过继承 LitePalApplication 来实现初始化。初始化的目的为了方便获取 Context 对象

```java
public class LitePalApplication extends Application {
  
   static Context sContext;
  
   public LitePalApplication() {//默认的构造方法，初始化 Context 静态变量
      sContext = this;
   }
}
```



## 数据库的创建

查看文档说明，我们发现 table 的创建和修改都是自动实现的。对外公开的主要是对数据的操作，所以我们就顺着对数据的操作来分析。而进行操作的前提是表中有数据，我们就从插入操作（对应 save 方法）看起。

`DataSupport#save()`

```java
public synchronized boolean save() {
  	  //代码省略
      saveThrows();    
  	  //代码省略
}
```

save 会调用 saveThrows 方法，

`DataSupport#saveThrows`

```java
public synchronized void saveThrows() {
   SQLiteDatabase db = Connector.getDatabase();//获取数据库
//代码省略
}
```

saveThrows 方法中首先调用了 Connector.getDatabase() 获取数据库。

### Connector#getDatabase()

`Connector#getDatabase`主要调用流程如下

```java
Connector#getDatabase	
	 --> getWritableDatabase
	 	--> buildConnection //获取 LitePalOpenHelper （实际上是一个 SQLiteOpenHelper）
	 		 --> LitePalAttr.getInstance();//获取单例对象
	 		 	 --> LitePalAttr = new LitePalAttr();//创建 LitePalAttr
                  --> loadLitePalXMLConfiguration();//加载 liepal.xml 配置文件中的数据
                  	 --> LitePalParser.parseLitePalConfiguration() //首先进行解析
                  	 --> //将解析到的数据（ 数据库名、数据库版本号、需要映射为表的类名、是否大小写敏感、存储路径 ）赋值给 LitePalAttr 中的字段
			 --> LitePalAttr.checkSelfValid();//参数合法性检查
			 --> //构造存储路径	
			 --> //创建 LitePalOpenHelper
	 	--> SQLiteOpenHelper#getWritableDatabase
```



虽然 LitePal 号称零配置，但是基础的准备工作还是需要做的。按照官方教程，在使用 LitePal 之前，需要在 res 目录下面创建一个 xml 目录，然后在该目录中创建一个名为 LitePal 的 xml 文件。举个栗子：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LitePal>
    <!--数据库名-->
    <dbname value="library"/>
    <!--版本号-->
    <version value="1"/>
    <!--映射表-->
    <list>
        <mapping class="com.android.rdc.librarysystem.bean.Book"/>
        <mapping class="com.android.rdc.librarysystem.bean.BookType"/>
        <mapping class="com.android.rdc.librarysystem.bean.Borrow"/>
    </list>
</LitePal>
```

其中的 `dbname ` 标签就是用于指定数据库名的。

`Connector#buildConnection`

```java
private static LitePalOpenHelper buildConnection() {
   LitePalAttr LitePalAttr = LitePalAttr.getInstance(); //获取单例对象——LitePalAttr 
   LitePalAttr.checkSelfValid();//参数合法性检查
   if (mLitePalHelper == null) {
      String dbName = LitePalAttr.getDbName();//
    		   //……省略路径处理代码
               dbName = dbPath + "/" + dbName;
           
      mLitePalHelper = new LitePalOpenHelper(dbName, LitePalAttr.getVersion());//
   }
   return mLitePalHelper;
}
```

首次连接数据库时，LitePal 会通过 `LitePalParser#parseLitePalConfiguration();`方法对 xml 配置文件进行解析。大体流程就是使用 `XmlPullParser` 对 xml 文件的解析，解析完成之后，就将结果保存在 `LitePalConfig `类中。 LitePalConfig 类字段如下所示：

```java
public class LitePalConfig {
    private int version;//数据库版本号
    private String dbName;//数据库名
    private String cases;//数据库大小写敏感性
    private String storage;//数据库文件的存储路径，可选择内部存储或者外部存储
    private List<String> classNames;//需要进行映射的模型类，每一个类名都需要给全名（包括包名）
 	//代码省略
}
```

#### 注：大小写敏感

SQLite 是大小写敏感的，具体：

1.  创建表、列的时候，SQL 语句里是什么大小写，表名、列名就是什么大小写；
2.  SQL 语句执行的时候：==表名、列名大小写不敏感，都能识别==；
3.  SQL 语句里面，“=”还是 LIKE 都是大小写敏感；

### buildConnection

`buildConnection` 方法首先会去获取 LitePalAttr 单例对象（首次调用会触发解析 LitePal.xml 文件），然后检查参数合法性，并根据解析得到的 LitePalAttr  中的字段创建一个 LitePalOpenHelper。

到这里，数据库已经创建完成了。那么数据库中的**表是什么时候创建的**呢？回到前面的 saveThrows 方法，该方法在获取数据库之后就开始做保存工作了（表创建工作不应该放在 保存工作过程中，因为这样不合理）。因此，表很可能就是在数据库创建过程中顺便创建的。实际上也确实如此。

### `LitePalOpenHelper#onCreate` 调用链

主要流程如下：

```java
LitePalOpenHelper#onCreate//数据库 onCreate 回调方法
   --> Generator#create()
  	 	--> create(db, true);
			 --> new Creator();
			 --> creator.createOrUpgradeTable(db, force);		
				 --> 迭代调用 createOrUpgradeTable(tableModel, db, force);
						--> getCreateTableSQLs(tableModel, db, force) //拼接建表 sql 语句，存储在 ArrayList 中
						--> execute();	
							 --> 迭代调用 db.execSQL() 执行 SQL 语句
						--> AssociationCreator#giveTableSchemaACopy
	 	--> addAssociation(db, true);	
```

上述过程中，我们看下 getCreateTableSQLs 方法，该方法会先生成删除表的语句 sql，然后再生成创建表的语句，并将它们按序存在 ArrayList 中。也就是说，每次建表之前先尝试删除同名的表（如果有的话）。

```java
protected List<String> getCreateTableSQLs(TableModel tableModel, SQLiteDatabase db, boolean force) {
       List<String> sqls = new ArrayList<String>();
   if (force) {
           sqls.add(generateDropTableSQL(tableModel));
           sqls.add(generateCreateTableSQL(tableModel));
   } else {
      if (DBUtility.isTableExists(tableModel.getTableName(), db)) {
         return null;
      } else {
               sqls.add(generateCreateTableSQL(tableModel));
      }
   }
       return sqls;
}
```



至此数据库和表创建完毕。



## dataSupport#save() 的背后

先看下整体的调用流程

```java
datasupport#save
 --> saveThrows
 	 --> 创建 SaveHanlder 
	 --> 调用 SaveHanlder#onSave
	 	 --> getSupportedFields//获取支持域
		 --> getSupportedGenericFields//获取支持的泛型域
		 --> getAssociationInfo//获取关联关系
	 	 --> if(baseObjId > 0)  doUpdateAction//执行更新操作
	 	 --> else doSaveAction
	 	 	 --> values.clear();//清空 ContentValue
	 	 	 --> beforeSave();//存放值（对象本身的值与外键）
	 	 	 --> long id = saving();
				 --> SQLiteDatabase#insert()// 将记录插入数据库
                   	 --> SQLiteDatabase#insertWithOnConflict()//拼接 sql 语句 
			 --> afterSave() 	 	 
 	 --> clearAssociatedData();//清除所有的关联数据
	 --> SQLiteDatabase#setTransactionSuccessful();//设置事务成功
```



```java
public synchronized boolean save() {
   try {
      saveThrows();
      return true;
   } catch (Exception e) {
      e.printStackTrace();
      return false;
   }
}
```

LitePal 文档中对 save 方法的说明如下:

>   如果是一条新记录，则插入一个新的行。如果保存过程中出错，整个操作取消，相应的修改会回滚。如果数据类中含有名为 id 或者 _id 并且类型为 int / long 的字段，那么在该对象被保存之后它们会被赋予相应的  id 值。（与 DataSupport 类中的 baseObjId 相同）

从 synchronized 可以看出 save 是一个同步方法，保证线程安全。它在内部调用了 saveThrows 方法。如果捕获到异常就 return false；否则 return true 表示保存成功。

### saveThrows

`DataSupport#saveThrows`

```java
public synchronized void saveThrows() {
   SQLiteDatabase db = Connector.getDatabase();//获取数据库
   db.beginTransaction();//开始事务
   try {
      SaveHandler saveHandler = new SaveHandler(db);//创建 SaveHandler，构造方法会创建一个 ContentValues 
      saveHandler.onSave(this);//保存
      clearAssociatedData();//清除关联数据
      db.setTransactionSuccessful();//事务成功
   } catch (Exception e) {
      throw new DataSupportException(e.getMessage(), e);
   } finally {
      db.endTransaction();//结束事务
   }
}
```

saveThrows 方法中使用了事务保证了数据的原子性。不过它没有做具体的保存工作，而是创建一个 SaveHandler 然后将自身的引用传递给 onSave 方法。

### SaveHandler

继承结构如下：

![image](https://user-images.githubusercontent.com/16668676/33593102-d1bd9cb6-d9c8-11e7-94fe-38117ed0ee50.png)

**DataHandler** 是 CRUD 组件的基类。里面定义了一些 CRUD 操作所需要通用方法。DataHandler 中又继承自**LitePalBase**。LitePalBase 是所有 的 LitePal 组件的基类。主要是给有如下需求的组件提供解决方案：①需要与其他组件进行交互的组件②有通用的逻辑代码的组件。

**SaveHandler** 继承自 DataHandler。从继承关系图中可以看到所有的 CRUD 操作都有相应的 Handler 。而且他们都继承自 DataHandler。是 DataSupport 下面的一个组件。SaveHandler 处理保存的工作。所有的实现基于所有的实现都基于 java 反射 API 和 Android SQLiteDatabase API。它如果关联模型已经存储了，它会自动在当前模型与关联模型之间建立关系。

`SaveHandler#onSave`

```java
void onSave(DataSupport baseObj) throws SecurityException, IllegalArgumentException,
      NoSuchMethodException, IllegalAccessException, InvocationTargetException {
   String className = baseObj.getClassName();//获取要保存对象的类名
   List<Field> supportedFields = getSupportedFields(className);//获取支持的字段
       List<Field> supportedGenericFields = getSupportedGenericFields(className);//获取支持的泛型信息
   Collection<AssociationsInfo> associationInfos = getAssociationInfo(className);//获取关联信息
   if (!baseObj.isSaved()) {
           if (!ignoreAssociations) {
               analyzeAssociatedModels(baseObj, associationInfos);
           }
      doSaveAction(baseObj, supportedFields, supportedGenericFields);//对 id 值<=0 对象执行保存操作
           if (!ignoreAssociations) {
               analyzeAssociatedModels(baseObj, associationInfos);
           }
   } else {
           if (!ignoreAssociations) {
               analyzeAssociatedModels(baseObj, associationInfos);
           }
      doUpdateAction(baseObj, supportedFields, supportedGenericFields);//对象本身存在数据库中，执行更新操作
   }
}
```

 onSave 方法会先去收集 LitePal 支持的类型和基础类型的 Filed 并将它们分别存在两个 List 中，同时也会获取关联信息。然后通过判断对象的 baseObjid > 0 是否成立，不成立，说明对象是一个未保存的对象，执行 doSaveAction 方法；否则调用 doUpdateAction 方法。

`LitePalBase#getSupportedFields`

```java
protected List<Field> getSupportedFields(String className) {
       List<Field> fieldList = classFieldsMap.get(className);//尝试从缓存中获取
       if (fieldList == null) {//
           List<Field> supportedFields = new ArrayList<Field>();
           Class<?> clazz;
           try {
               clazz = Class.forName(className);//获取类名
           } catch (ClassNotFoundException e) {
               throw new DatabaseGenerateException(DatabaseGenerateException.CLASS_NOT_FOUND + className);
           }
           recursiveSupportedFields(clazz, supportedFields);//递归类型
           classFieldsMap.put(className, supportedFields);//缓存到 hashMap 中
           return supportedFields;//返回
       }
       return fieldList;//缓存命中
}
```



```java
private void recursiveSupportedFields(Class<?> clazz, List<Field> supportedFields) {
    if (clazz == DataSupport.class || clazz == Object.class) {
        return;
    }
    Field[] fields = clazz.getDeclaredFields();
    if (fields != null && fields.length > 0) {
        for (Field field : fields) {
            Column annotation = field.getAnnotation(Column.class);
            if (annotation != null && annotation.ignore()) {//有注解，但是注解设置为忽略
                continue;
            }
            int modifiers = field.getModifiers();//获取修饰符
            if (!Modifier.isStatic(modifiers)) {//不是静态的 
                Class<?> fieldTypeClass = field.getType();//获取字段域类型
                String fieldType = fieldTypeClass.getName();//字段类型的名称
                if (BaseUtility.isFieldTypeSupported(fieldType)) {
                    supportedFields.add(field);//非静态、并且受支持的字段添加到 List<Filed> 中
                }
            }
        }
    }
    recursiveSupportedFields(clazz.getSuperclass(), supportedFields);
}

```

递归调用，直到是 DataSupport 或者 Object 为止，通过反射取得所有的 fields，当不需要忽略时，不是静态，一些基本支持的数据类型时会直接加入到这个 fieldList 里边。

getSupportedGenericFields() 方法与上面的类似，只不过处理的是泛型。

### LitePalBase#getAssociationInfo

```java
private Collection<AssociationsInfo> mAssociationInfos;

protected Collection<AssociationsInfo> getAssociationInfo(String className) {
   if (mAssociationInfos == null) {
      mAssociationInfos = new HashSet<AssociationsInfo>();
   }
   mAssociationInfos.clear();//清理关系集
   analyzeClassFields(className, GET_ASSOCIATION_INFO_ACTION);//分析类的字段域
   return mAssociationInfos;
}
```

 

分析类字段以找出关联关系

```java
private void analyzeClassFields(String className, int action) {
   try {
      Class<?> dynamicClass = Class.forName(className);//加载类
      Field[] fields = dynamicClass.getDeclaredFields();//
      for (Field field : fields) {
         if (isNonPrimitive(field)) {//不是原生类型
            Column annotation = field.getAnnotation(Column.class);//获取注解
            if (annotation != null && annotation.ignore()) {
            	continue;
			}
            oneToAnyConditions(className, field, action);//一对？关系
            manyToAnyConditions(className, field, action);//多对？关系
         }
      }
   } catch (ClassNotFoundException ex) {
		//代码省略
   }
}
```



#### LitePalBase#oneToAnyConditions

首先我们要明确，oneToAnyConditions  是在遍历处理类的所有字段的过程中调用的。并且只有当字段类型是非原生类型时才会进入该方法。

设有 A 和 B 两个类。现在对 A 中存在类型为 B 的字段。首先判断 B 是否在配置文件里边配置的。如果是，就找到这个类的所有字段。找到一个和 A 相同的字段，这代表 A 里边有一个 B，B 里边有一个 A，是一对一关系；如果 B 找到的和 A 相同的字段是一个集合，代表 A 里边有一个 B，B 里边有多个 A，是多对一关系。

最后统一把关系加入到一个集合里边来管理。

```java
private void oneToAnyConditions(String className, Field field, int action) throws ClassNotFoundException {
   Class<?> fieldTypeClass = field.getType();//获取字段的类型

  //字段的类型包含在 LitePal 配置文件映射的列表中
   if (LitePalAttr.getInstance().getClassNames().contains(fieldTypeClass.getName())) {
      Class<?> reverseDynamicClass = Class.forName(fieldTypeClass.getName());//通过类名加载字段的类（简称为 B）
      Field[] reverseFields = reverseDynamicClass.getDeclaredFields();
      boolean reverseAssociations = false;
      //  开始检查类 B 中的属性
      for (int i = 0; i < reverseFields.length; i++) {
         Field reverseField = reverseFields[i];//
         if (!Modifier.isStatic(reverseField.getModifiers())) {
            Class<?> reverseFieldTypeClass = reverseField.getType();
            // B 中也有 A 的引用。因此是一对一的双向关系
            if (className.equals(reverseFieldTypeClass.getName())) {
               if (action == GET_ASSOCIATIONS_ACTION) {
                  addIntoAssociationModelCollection(className, fieldTypeClass.getName(),
                        fieldTypeClass.getName(), Const.Model.ONE_TO_ONE);
               } else if (action == GET_ASSOCIATION_INFO_ACTION) {
                  addIntoAssociationInfoCollection(className, fieldTypeClass.getName(),
                        fieldTypeClass.getName(), field, reverseField, Const.Model.ONE_TO_ONE);
               }
               reverseAssociations = true;
            }
            // B 类中有字段： List<A> 说明是多对一关系
            else if (isCollection(reverseFieldTypeClass)) {
               String genericTypeName = getGenericTypeName(reverseField);
               if (className.equals(genericTypeName)) {
                  if (action == GET_ASSOCIATIONS_ACTION) {
                     addIntoAssociationModelCollection(className, fieldTypeClass.getName(),
                           className, Const.Model.MANY_TO_ONE);
                  } else if (action == GET_ASSOCIATION_INFO_ACTION) {
                     addIntoAssociationInfoCollection(className, fieldTypeClass.getName(),
                           className, field, reverseField, Const.Model.MANY_TO_ONE);
                  }
                  reverseAssociations = true;
               }
            }
         }
      }
     	  // B 中没有 A 的引用，AB 是单向的一对一关系
           if (!reverseAssociations) {
               if (action == GET_ASSOCIATIONS_ACTION) {
                   addIntoAssociationModelCollection(className, fieldTypeClass.getName(),
                           fieldTypeClass.getName(), Const.Model.ONE_TO_ONE);
               } else if (action == GET_ASSOCIATION_INFO_ACTION) {
                   addIntoAssociationInfoCollection(className, fieldTypeClass.getName(),
                           fieldTypeClass.getName(), field, null, Const.Model.ONE_TO_ONE);
               }
           }
   }
}
```

manyToAnyConditions 也是通过类似的逻辑进行分析处理。

回到 saveThrows 方法，可以看到后续会调用 doSaveAction。

### doSaveAction

```java
private void doSaveAction(DataSupport baseObj, List<Field> supportedFields, List<Field> supportedGenericFields)
      throws SecurityException, IllegalArgumentException, NoSuchMethodException,
      IllegalAccessException, InvocationTargetException {
   values.clear();//先清除 contentValue 中的值
   beforeSave(baseObj, supportedFields, values);
   long id = saving(baseObj, values);//保存
   afterSave(baseObj, supportedFields, supportedGenericFields, id);
}
```

doSaveAction 方法中依次调用了 beforeSave  saving afterSave 三个方法。

-   beforeSave 方法会将相应的字段列与值以键值对的形式存放到 values 中， values 将作为后面 saving 方法的参数。
-   saving 方法内部会调用 调用 `SQLiteDatabase#insert` 方法对数据进行保存。
-   afterSave 所做的工作是获取刚刚保存的那条记录的 id ，然后以反射的形式将它赋给对象的 baseObjId 字段，同时还会更新关联表的数据

```java
private long saving(DataSupport baseObj, ContentValues values) {
       if (values.size() == 0) {//如果 contentValue 中的大小为 0，那就存储一个仅有 id 的空行
           values.putNull("id");
       }
   return mDatabase.insert(baseObj.getTableName(), null, values);//终于看到了 SqlLite 的保存操作
}
```



### SQLiteDatabase#insert

insert 方法内部会调用 SQLiteDatabase#insertWithOnConflict 方法。

可以看到该方法的实现是将相应的数据结构拼接为一个 SQL 插入语句

```java
private static final String[] CONFLICT_VALUES = new String[]
        {"", " OR ROLLBACK ", " OR ABORT ", " OR FAIL ", " OR IGNORE ", " OR REPLACE "};

public long insertWithOnConflict(String table, String nullColumnHack,
        ContentValues initialValues, int conflictAlgorithm) {
    acquireReference();
    try {
        StringBuilder sql = new StringBuilder();
        sql.append("INSERT");
        sql.append(CONFLICT_VALUES[conflictAlgorithm]);
        sql.append(" INTO ");
        sql.append(table);//表名
        sql.append('(');

        Object[] bindArgs = null;
        int size = (initialValues != null && !initialValues.isEmpty())
                ? initialValues.size() : 0;
        if (size > 0) {
            bindArgs = new Object[size];
            int i = 0;
            for (String colName : initialValues.keySet()) {
                sql.append((i > 0) ? "," : "");
                sql.append(colName);//添加名称
                bindArgs[i++] = initialValues.get(colName);//列名对应的值，将其存在数组中
            }
            sql.append(')');
            sql.append(" VALUES (");
            for (i = 0; i < size; i++) {
                sql.append((i > 0) ? ",?" : "?");
            }
        } else {
            sql.append(nullColumnHack + ") VALUES (NULL");//空行
        }
        sql.append(')');

        SQLiteStatement statement = new SQLiteStatement(this, sql.toString(), bindArgs);// 利用表名，以及需要更新的列名，还有相应的列参数「bindArgs」 创建一个 SQLiteStatement
        try {
            return statement.executeInsert();//执行插入操作
        } finally {
            statement.close();
        }
    } finally {
        releaseReference();//释放引用
    }
}
```

至此，save 过程分析完成。

## find 过程浅析

DataSupport#find 大体流程：

```java
find()
  --> find(modelClass, id, false);
       --> 创建 QueryHandler 
       --> queryHandler#onFind
           --> query  //获取符合条件的 list
             --> DataHandler#query
                     --> getSupportedFields //获取支持的字段
                     --> //调用相关方法。做一些铺垫工作
                    --> SQLiteDatabase#query  //调用 SQLiteDataBase#query （该方法最终会调用 buildQueryString 拼接出字符串）。然后通过 cursor 逐行取出数据
                         --> //省略一些转发方法
                           --> SQLiteDatabase#queryWithFactory
                           		--> SQLiteQueryBuilder#buildQueryString//拼接 sql 语句
```



### SQLiteQueryBuilder#buildQueryString

```java
public static String buildQueryString(
        boolean distinct, String tables, String[] columns, String where,
        String groupBy, String having, String orderBy, String limit) {
    if (TextUtils.isEmpty(groupBy) && !TextUtils.isEmpty(having)) {
        throw new IllegalArgumentException(
                "HAVING clauses are only permitted when using a groupBy clause");
    }
    if (!TextUtils.isEmpty(limit) && !sLimitPattern.matcher(limit).matches()) {
        throw new IllegalArgumentException("invalid LIMIT clauses:" + limit);
    }

    StringBuilder query = new StringBuilder(120);

    query.append("SELECT ");
    if (distinct) {
        query.append("DISTINCT ");
    }
    if (columns != null && columns.length != 0) {
        appendColumns(query, columns);
    } else {
        query.append("* ");
    }
    query.append("FROM ");
    query.append(tables);
    appendClause(query, " WHERE ", where);
    appendClause(query, " GROUP BY ", groupBy);
    appendClause(query, " HAVING ", having);
    appendClause(query, " ORDER BY ", orderBy);
    appendClause(query, " LIMIT ", limit);

    return query.toString();
}
```

小结：find 就是通过相应的查询条件拼接为 SQL 查询语句，然后将查询结果返回。



## 问题：

### 同步/异步

对于每一个 CRUD 操作，LitePal 都有同步以及相对应的异步方法。

#### 以异步 save 为例

异步更新只是新开一条线程来执行 save 方法，并在主线程进行结果回调

```java
public SaveExecutor saveAsync() {
    final SaveExecutor executor = new SaveExecutor();//参加一个 SaveExecutor
    Runnable runnable = new Runnable() {//创建一个 Runnable 匿名内部子类，
        @Override
        public void run() {
            synchronized (DataSupport.class) {
                final boolean success = save();//调用保存方法
                if (executor.getListener() != null) {
                    LitePal.getHandler().post(new Runnable() {//切换到主线程中回调
                        @Override
                        public void run() {
                            executor.getListener().onFinish(success);//回调
                        }
                    });
                }
            }
        }
    };
    executor.submit(runnable);
    return executor;
}
```

```java
public class SaveExecutor extends AsyncExecutor {

    private SaveCallback cb;//回调接口

    /**
     * 注册回调接口之后，马上执行异步任务
     */
    public void listen(SaveCallback callback) {//注册回调
        cb = callback;
        execute();//执行异步任务
    }

    public SaveCallback getListener() {
        return  cb;
    }

}
```

SaveExecutor 继承自 AsyncExecutor。特别之处在于它是通过 listen 方法来触发异步任务执行的。



```java
public abstract class AsyncExecutor {

    private Runnable pendingTask;//延时任务

    /**
     * 提交任务
     */
    public void submit(Runnable task) {
        pendingTask = task;
    }

    /**
     * 后台执行延时任务
     */
    void execute() {
        if (pendingTask != null) {
            new Thread(pendingTask).start();//新建线程执行任务
        }
    }

}
```

AsyncExecutor 一个简单的异步执行器，在后台线程中运行任务。它是所有的异步操作执行器的基类。

![image](https://user-images.githubusercontent.com/16668676/33612586-0077ccc2-da0d-11e7-80a5-5e093b7e79ba.png)

## 总结

LitePal 需要在 Application 中进行初始化。首次获取数据库的时会触发数据库以及表的创建。后续的 CRUD 操作都是通过**反射机制和 Android SQLite API** 来实现的。它会自动分析表与表之间的关联关系，并帮助创建表、自动管理表。

### 优点：

**配置简单，操作方便，业务对象清晰，LitePal 很「轻」，jar 包只有 100k 不到**。

### 缺点

1.  反射影响性能。
2.  无法直接进行关联查询。

### 与同类框架（GreenDAO）对比：

GreetDao 一开始就人工生成业务需要的 Model 和 DAO 文件，业务中可以直接调用相应的 DAO 文件进行数据库的增删改查操作，从而避免了因反射带来的性能损耗和效率低。因此大批量的插入、更新、查询等操作，greenDAO 用时短，执行快更适合。但是 greenDAO 使用不方便，学习成本较高。

如果应用对性能要求不是特别苛刻并且数据量也不大的情况下，轻量级的 LitePal，基本能满足我们的需求了。



## 参考资料与学习资源推荐

-   [数据库 ORM 之 LitePal](http://www.jianshu.com/p/35c704701638)

-   [Android 数据库高手秘籍](http://blog.csdn.net/column/details/android-database-pro.html)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源，谢谢！



