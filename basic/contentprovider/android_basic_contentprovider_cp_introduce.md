# ContentProvider介绍

###一、使用ContentProvider（内容提供者）共享数据

ContentProvider在android中的作用是对外共享数据，也就是说你可以通过ContentProvider把应用中的数据共享给其他应用访问，其他应用可以通过ContentProvider对你应用中的数据进行添删改查。

使用ContentProvider对外共享数据的好处是统一了数据的访问方式。

当应用需要通过ContentProvider对外共享数据时，第一步需要继承ContentProvider并重写下面方法：

```java
public class DemoContentP extends ContentProvider {
    @Override
    public boolean onCreate() {
        return false;
    }
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        return 0;
    }
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        return 0;
    }
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        return null;
    }
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        return null;
    }
    @Override
    public String getType(Uri uri) {
        return null;
    }
}
```
第二步需要在`AndroidManifest.xml`使用`<provider>`对该ContentProvider进行配置，为了能让其他应用找到该ContentProvider，ContentProvider采用了authorities（主机名/域名）对它进行唯一标识，你可以把ContentProvider看作是一个网站（想想，网站也是提供数据者），authorities 就是他的域名：
```xml
<manifest.... >
   <application android:icon="@drawable/icon" android:label="@string/app_name">
      <provider android:name=".PersonContentProvider" 
           android:authorities="com.ljq.providers.personprovider"/>
   </application>
</manifest>
```

###二、Uri介绍

Uri代表了要操作的数据'链接'，Uri主要包含了两部分信息：

- 1.需要操作的ContentProvider
- 2.对ContentProvider中的什么数据进行操作

一个Uri由以下几部分组成：          
![](imgs/a_b_cp_01.png)

**scheme** ContentProvider的scheme由Android规定为：`content://`

**主机名**（或叫Authority）用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。

**路径**（path）可以用来表示我们要操作的数据，路径的构建应根据业务而定，如下:
```
要操作person表中id为10的记录，可以构建这样的路径:/person/10
要操作person表中id为10的记录的name字段， person/10/name
要操作person表中的所有记录，可以构建这样的路径:/person
要操作xxx表中的记录，可以构建这样的路径:/xxx
```
当然要操作的数据不一定来自数据库，也可以是文件、xml或网络等其他存储方式，如下:
```
要操作xml文件中person节点下的name节点，可以构建这样的路径：/person/name
```
如果要把一个字符串转换成Uri，可以使用Uri类中的parse()方法，如下：
```java
Uri uri = Uri.parse("content://com.ljq.provider.personprovider/person")
```
###三、UriMatcher类使用介绍

因为Uri代表了要操作的数据，所以我们经常需要解析Uri，并从Uri中获取数据。Android系统提供了两个用于操作Uri的工具类，分别为UriMatcher和ContentUris 。掌握它们的使用，会便于我们的开发工作。

UriMatcher类用于匹配Uri，它的用法如下：

```java
//常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码
UriMatcher  sMatcher = new UriMatcher(UriMatcher.NO_MATCH);
//如果match()方法匹配content://com.ljq.provider.personprovider/person路径，返回匹配码为1
sMatcher.addURI("com.ljq.provider.personprovider", "person", 1);//添加需要匹配uri，如果匹配就会返回匹配码
//如果match()方法匹配content://com.ljq.provider.personprovider/person/230路径，返回匹配码为2
sMatcher.addURI("com.ljq.provider.personprovider", "person/#", 2);//#号为通配符
switch (sMatcher.match(Uri.parse("content://com.ljq.provider.personprovider/person/10"))) { 
   case 1
     break;
   case 2
     break;
   default://不匹配
     break;
}
```
注册完需要匹配的Uri后，就可以使用`sMatcher.match(uri)`方法对输入的Uri进行匹配，如果匹配就返回匹配码，匹配码是调用`addURI()`方法传入的第三个参数，假设匹配`content://com.ljq.provider.personprovider/person`路径，返回的匹配码为1。

###四、ContentUris类使用介绍

ContentUris类用于操作Uri路径后面的ID部分，它有两个比较实用的方法：

`withAppendedId(uri, id)`用于为路径加上ID部分：
```java
Uri uri = Uri.parse("content://com.ljq.provider.personprovider/person")
Uri resultUri = ContentUris.withAppendedId(uri, 10); 
//生成后的Uri为：content://com.ljq.provider.personprovider/person/10
```
`parseId(uri)`方法用于从路径中获取ID部分：
```java
Uri uri = Uri.parse("content://com.ljq.provider.personprovider/person/10")
long personid = ContentUris.parseId(uri);//获取的结果为:10
```

###五、使用ContentProvider共享数据

ContentProvider类主要方法：

```java
public boolean onCreate();//该方法在ContentProvider创建后就会被调用，Android开机后，ContentProvider在其它应用第一次访问它时才会被创建。
public Uri insert(Uri uri, ContentValues values);//该方法用于供外部应用往ContentProvider添加数据。
public int delete(Uri uri, String selection, String[] selectionArgs);//该方法用于供外部应用从ContentProvider删除数据。
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs);//该方法用于供外部应用更新ContentProvider中的数据。
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder);//该方法用于供外部应用从ContentProvider中获取数据。
public String getType(Uri uri);//该方法用于返回当前Url所代表数据的MIME类型。
···
```

- 如果操作的数据属于集合类型，那么MIME类型字符串应该以`vnd.android.cursor.dir/`开头。

例如：要得到所有person记录的Uri为`content://com.ljq.provider.personprovider/person`，那么返回的MIME类型字符串应该为：`"vnd.android.cursor.dir/person"`。

- 如果要操作的数据属于非集合类型数据，那么MIME类型字符串应该以`vnd.android.cursor.item/`开头。

例如：得到id为10的person记录，Uri为`content://com.ljq.provider.personprovider/person/10`，那么返回的MIME类型字符串为：`"vnd.android.cursor.item/person"`。

              

                

###六、使用ContentResolver操作ContentProvider中的数据

当外部应用需要对ContentProvider中的数据进行添加、删除、修改和查询操作时，可以使用ContentResolver类来完成，要获取ContentResolver对象，可以使用Activity提供的`getContentResolver()`方法。ContentResolver类提供了与ContentProvider类相同签名的四个方法：
```java
public Uri insert(Uri uri, ContentValues values);//该方法用于往ContentProvider添加数据。
public int delete(Uri uri, String selection, String[] selectionArgs);//该方法用于从ContentProvider删除数据。
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs);//该方法用于更新ContentProvider中的数据。
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder);//该方法用于从ContentProvider中获取数据。
```
这些方法的第一个参数为Uri，代表要操作的ContentProvider和对其中的什么数据进行操作。

假设给定的是：`Uri.parse("content://com.ljq.providers.personprovider/person/10")`，那么将会对主机名为`com.ljq.providers.personprovider`的ContentProvider进行操作，操作的数据为person表中id为10的记录。

使用ContentResolver对ContentProvider中的数据进行添加、删除、修改和查询操作：

```java
ContentResolver resolver =  getContentResolver();
Uri uri = Uri.parse("content://com.ljq.provider.personprovider/person");
//添加一条记录
ContentValues values = new ContentValues();
values.put("name", "linjiqin");
values.put("age", 25);
resolver.insert(uri, values);  
//获取person表中所有记录
Cursor cursor = resolver.query(uri, null, null, null, "personid desc");
while(cursor.moveToNext()){
   Log.i("ContentTest", "personid="+ cursor.getInt(0)+ ",name="+ cursor.getString(1));
}
//把id为1的记录的name字段值更改新为zhangsan
ContentValues updateValues = new ContentValues();
updateValues.put("name", "zhangsan");
Uri updateIdUri = ContentUris.withAppendedId(uri, 2);
resolver.update(updateIdUri, updateValues, null, null);
//删除id为2的记录
Uri deleteIdUri = ContentUris.withAppendedId(uri, 2);
resolver.delete(deleteIdUri, null, null);
```

###七、监听ContentProvider中数据的变化

如果想让ContentProvider的访问者需要知道ContentProvider中的数据发生变化，可以在ContentProvider发生数据变化时调用`getContentResolver().notifyChange(uri, null)`来通知注册在此URI上的访问者，例子如下：
```java
public class PersonContentProvider extends ContentProvider {
   public Uri insert(Uri uri, ContentValues values) {
      db.insert("person", "personid", values);
      getContext().getContentResolver().notifyChange(uri, null);
   }
}
```
如果ContentProvider的访问者需要得到数据变化通知，必须使用ContentObserver对数据（数据采用uri描述）进行监听，当监听到数据变化通知时，系统就会调用`ContentObserver的onChange()`方法：

```java
getContentResolver().registerContentObserver(Uri.parse("content://com.ljq.providers.personprovider/person"),
       true, new PersonObserver(new Handler()));
public class PersonObserver extends ContentObserver{
   public PersonObserver(Handler handler) {
      super(handler);
   }
   public void onChange(boolean selfChange) {
      //此处可以进行相应的业务处理
   }
}
```