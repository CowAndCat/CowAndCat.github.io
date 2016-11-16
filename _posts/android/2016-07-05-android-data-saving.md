---
layout: post
title: Android数据存储五种方式总结
category: android
comments: false
---

## 五大方式,分别如下: 

1. 使用SharedPreferences存储数据

2. 文件存储数据 

3. SQLite数据库存储数据

4. 使用ContentProvider存储数据

5. 网络存储数据

##  1. 使用SharedPreferences存储数据 


 适用范围：保存少量的数据，且这些数据的格式非常简单：字符串型、基本类型的值。比如应用程序的各种配置信息（如是否打开音效、是否使用震动效果、小游戏的玩家积分等），解锁口令密码等.

### 核心原理
保存基于XML文件存储的key-value键值对数据，通常用来存储一些简单的配置信息。通过DDMS的File Explorer面板，展开文件浏览树,很明显SharedPreferences数据总是存储在/data/data/<package name>/shared_prefs目录下。

SharedPreferences对象本身只能获取数据而不支持存储和修改,存储修改是通过SharedPreferences.edit()获取的内部接口Editor对象实现。 

SharedPreferences本身是一接口，程序无法直接创建SharedPreferences实例，只能通过Context提供的getSharedPreferences(String name, int mode)方法来获取SharedPreferences实例，该方法中name表示要操作的xml文件名，第二个参数具体如下：

     Context.MODE_PRIVATE: 指定该SharedPreferences数据只能被本应用程序读、写。

     Context.MODE_WORLD_READABLE:  指定该SharedPreferences数据能被其他应用程序读，但不能写。

     Context.MODE_WORLD_WRITEABLE:  指定该SharedPreferences数据能被其他应用程序读，写

Editor有如下重要方法：

     SharedPreferences.Editor clear():清空SharedPreferences里所有数据

     SharedPreferences.Editor putXxx(String key , xxx value): 向SharedPreferences存入指定key对应的数据，其中xxx 可以是boolean,float,int等各种基本类型据

     SharedPreferences.Editor remove(): 删除SharedPreferences中指定key对应的数据项

     boolean commit(): 当Editor编辑完成后，使用该方法提交修改

### 代码例子


	class ViewOcl implements View.OnClickListener{
        @Override
        public void onClick(View v) {

            switch(v.getId()){
            case R.id.btnSet:
                //步骤1：获取输入值
                String code = txtCode.getText().toString().trim();
                //步骤2-1：创建一个SharedPreferences.Editor接口对象，lock表示要写入的XML文件名，MODE_WORLD_WRITEABLE写操作
                SharedPreferences.Editor editor = getSharedPreferences("lock", MODE_WORLD_WRITEABLE).edit();
                //步骤2-2：将获取过来的值放入文件
                editor.putString("code", code);
                //步骤3：提交
                editor.commit();
                Toast.makeText(getApplicationContext(), "口令设置成功", Toast.LENGTH_LONG).show();
                break;
            case R.id.btnGet:
                //步骤1：创建一个SharedPreferences接口对象
                SharedPreferences read = getSharedPreferences("lock", MODE_WORLD_READABLE);
                //步骤2：获取文件中的值
                String value = read.getString("code", "");
                Toast.makeText(getApplicationContext(), "口令为："+value, Toast.LENGTH_LONG).show();
                
                break;
                
            }
        }
        
    }

### 步骤
读写其他应用的SharedPreferences：

    1、在创建SharedPreferences时，指定MODE_WORLD_READABLE模式，表明该SharedPreferences数据可以被其他程序读取

    2、创建其他应用程序对应的Context:

        Context pvCount = createPackageContext("com.tony.app", Context.CONTEXT_IGNORE_SECURITY);这里的com.tony.app就是其他程序的包名

    3、使用其他程序的Context获取对应的SharedPreferences

        SharedPreferences read = pvCount.getSharedPreferences("lock", Context.MODE_WORLD_READABLE);

    4、如果是写入数据，使用Editor接口即可，所有其他操作均和前面一致。

### 特点
SharedPreferences对象与SQLite数据库相比，免去了创建数据库，创建表，写SQL语句等诸多操作，相对而言更加方便，简洁。

但是SharedPreferences也有其自身缺陷:
比如其只能存储boolean，int，float，long和String五种简单的数据类型
比如其无法进行条件查询等。

所以不论SharedPreferences的数据存储操作是如何简单，它也只能是存储方式的一种补充，而无法完全替代如SQLite数据库这样的其他数据存储方式。


## 2. 文件存储数据

### 核心原理

Context提供了两个方法来打开数据文件里的文件IO流:
 ```FileInputStream openFileInput(String name); FileOutputStream(String name , int mode)```,这两个方法第一个参数 用于指定文件名，第二个参数指定打开文件的模式。

 具体有以下值可选：

	MODE_PRIVATE：为默认操作模式，代表该文件是私有数据，只能被应用本身访问，在该模式下，写入的内容会覆盖原文件的内容，如果想把新写入的内容追加到原文件中。可   以使用Context.MODE_APPEND

	MODE_APPEND：模式会检查文件是否存在，存在就往文件追加内容，否则就创建新文件。

	MODE_WORLD_READABLE：表示当前文件可以被其他应用读取；

	MODE_WORLD_WRITEABLE：表示当前文件可以被其他应用写入。

 除此之外，Context还提供了如下几个重要的方法：

	getDir(String name , int mode):在应用程序的数据文件夹下获取或者创建name对应的子目录

	File getFilesDir():获取该应用程序的数据文件夹得绝对路径

	String[] fileList():返回该应用数据文件夹的全部文件               

### 代码实例

	public String read() {
        try {
            FileInputStream inStream = this.openFileInput("message.txt");
            byte[] buffer = new byte[1024];
            int hasRead = 0;
            StringBuilder sb = new StringBuilder();
            while ((hasRead = inStream.read(buffer)) != -1) {
                sb.append(new String(buffer, 0, hasRead));
            }

            inStream.close();
            return sb.toString();
        } catch (Exception e) {
            e.printStackTrace();
        } 
        return null;
    }
    
    public void write(String msg){
        // 步骤1：获取输入值
        if(msg == null) return;
        try {
            // 步骤2:创建一个FileOutputStream对象,MODE_APPEND追加模式
            FileOutputStream fos = openFileOutput("message.txt",
                    MODE_APPEND);
            // 步骤3：将获取过来的值放入文件
            fos.write(msg.getBytes());
            // 步骤4：关闭数据流
            fos.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

openFileOutput()方法的第一参数用于指定文件名称，不能包含路径分隔符“/” ，如果文件不存在，Android 会自动创建它。创建的文件保存在/data/data/<package name>/files目录，

如： /data/data/cn.tony.app/files/message.txt，

### 读写sdcard上的文件

其中读写步骤按如下进行:

1、调用Environment的getExternalStorageState()方法判断手机上是否插了sd卡,且应用程序具有读写SD卡的权限，如下代码将返回true

```Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)```

2、调用Environment.getExternalStorageDirectory()方法来获取外部存储器，也就是SD卡的目录,或者使用"/mnt/sdcard/"目录

3、使用IO流操作SD卡上的文件 

注意：手机应该已插入SD卡，对于模拟器而言，可通过mksdcard命令来创建虚拟存储卡

必须在AndroidManifest.xml上配置读写SD卡的权限:
```
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

## 3. SQLite存储数据

SQLite是轻量级嵌入式数据库引擎，它支持 SQL 语言，并且只利用很少的内存就有很好的性能。现在的主流移动设备像Android、iPhone等都使用SQLite作为复杂数据的存储引擎，在我们为移动设备开发应用程序时，也许就要使用到SQLite来存储我们大量的数据，所以我们就需要掌握移动设备上的SQLite开发技巧.

SQLiteDatabase类为我们提供了很多种方法，代码中基本上囊括了大部分的数据库操作；对于添加、更新和删除来说，可以使用:

	db.executeSQL(String sql);  
	db.executeSQL(String sql, Object[] bindArgs);//sql语句中使用占位符，然后第二个参数是实际的参数集 

除了统一的形式之外，他们还有各自的操作方法：

	db.insert(String table, String nullColumnHack, ContentValues values);  
	db.update(String table, Contentvalues values, String whereClause, String whereArgs);  
	db.delete(String table, String whereClause, String whereArgs);

以上三个方法的第一个参数都是表示要操作的表名；

insert中的第二个参数表示如果插入的数据每一列都为空的话，需要指定此行中某一列的名称，系统将此列设置为NULL，不至于出现错误；

insert中的第三个参数是ContentValues类型的变量，是键值对组成的Map，key代表列名，value代表该列要插入的值；

update的第二个参数也很类似，只不过它是更新该字段key为最新的value值，第三个参数whereClause表示WHERE表达式，比如“age > ? and age < ?”等，最后的whereArgs参数是占位符的实际参数值；

delete方法的参数也是一样。

### show you the code

#### 1. 数据添加

**insert**:

	ContentValues cv = new ContentValues();//实例化一个ContentValues用来装载待插入的数据
	cv.put("title","you are beautiful");//添加title
	cv.put("weather","sun"); //添加weather
	cv.put("context","xxxx"); //添加context
	String publish = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
	                        .format(new Date());
	cv.put("publish ",publish); //添加publish
	db.insert("diary",null,cv);//执行插入操作

** 使用execSQL方式来实现**

	String sql = "insert into user(username,password) values ('Jack Johnson','iLovePopMuisc');//插入操作的SQL语句
	db.execSQL(sql);//执行SQL语句

#### 2. 数据的删除

同样有2种方式可以实现

	String whereClause = "username=?";//删除的条件
	String[] whereArgs = {"Jack Johnson"};//删除的条件参数
	db.delete("user",whereClause,whereArgs);//执行删除

使用execSQL方式的实现

	String sql = "delete from user where username='Jack Johnson'";//删除操作的SQL语句
	db.execSQL(sql);//执行删除操作

#### 3. 数据修改

同上，仍是2种方式

	ContentValues cv = new ContentValues();//实例化ContentValues
	cv.put("password","iHatePopMusic");//添加要更改的字段及内容
	String whereClause = "username=?";//修改条件
	String[] whereArgs = {"Jack Johnson"};//修改条件的参数
	db.update("user",cv,whereClause,whereArgs);//执行修改

使用execSQL方式的实现

	String sql = "update user set password = 'iHatePopMusic' where username='Jack Johnson'";//修改的SQL语句
	db.execSQL(sql);//执行修改

#### 4. 数据查询

下面来说说查询操作。查询操作相对于上面的几种操作要复杂些，因为我们经常要面对着各种各样的查询条件，所以系统也考虑到这种复杂性，为我们提供了较为丰富的查询形式：

	db.rawQuery(String sql, String[] selectionArgs);  
	db.query(String table, String[] columns, String selection, String[] selectionArgs, 
		String groupBy, String having, String orderBy);  
	db.query(String table, String[] columns, String selection, String[] selectionArgs, 
		String groupBy, String having, String orderBy, String limit);  
	db.query(String distinct, String table, String[] columns, String selection, 
			String[] selectionArgs, String groupBy, String having, String orderBy, String limit);

上面几种都是常用的查询方法，第一种最为简单，将所有的SQL语句都组织到一个字符串中，使用占位符代替实际参数，selectionArgs就是占位符实际参数集；

各参数说明：

- table：表名称
- colums：表示要查询的列所有名称集
- selection：表示WHERE之后的条件语句，可以使用占位符
- selectionArgs：条件语句的参数数组
- groupBy：指定分组的列名
- having：指定分组条件,配合groupBy使用
- orderBy：y指定排序的列名
- limit：指定分页参数
- distinct：指定“true”或“false”表示要不要过滤重复值
- Cursor：返回值，相当于结果集ResultSet


最后，他们同时返回一个Cursor对象，代表数据集的游标，有点类似于JavaSE中的ResultSet。下面是Cursor对象的常用方法：

- c.move(int offset); //以当前位置为参考,移动到指定行  
- c.moveToFirst();    //移动到第一行  
- c.moveToLast();     //移动到最后一行  
- c.moveToPosition(int position); //移动到指定行  
- c.moveToPrevious(); //移动到前一行  
- c.moveToNext();     //移动到下一行  
- c.isFirst();        //是否指向第一条  
- c.isLast();     //是否指向最后一条  
- c.isBeforeFirst();  //是否指向第一条之前  
- c.isAfterLast();    //是否指向最后一条之后  
- c.isNull(int columnIndex);  //指定列是否为空(列基数为0)  
- c.isClosed();       //游标是否已关闭  
- c.getCount();       //总数据项数  
- c.getPosition();    //返回当前游标所指向的行数  
- c.getColumnIndex(String columnName);//返回某列名对应的列索引值  
- c.getString(int columnIndex);   //返回当前行指定列的值

实现代码：

	String[] params =  {12345,123456};
	Cursor cursor = db.query("user",columns,"ID=?",params,null,null,null);//查询并获得游标
	if(cursor.moveToFirst()){//判断游标是否为空
	    for(int i=0;i<cursor.getCount();i++){
	        cursor.move(i);//移动到指定记录
	        String username = cursor.getString(cursor.getColumnIndex("username");
	        String password = cursor.getString(cursor.getColumnIndex("password"));
	    }
	}

通过rawQuery实现的带参数查询：
	Cursor result=db.rawQuery("SELECT ID, name, inventory FROM mytable");
	//Cursor c = db.rawQuery("s name, inventory FROM mytable where ID=?",new Stirng[]{"123456"});
	result.moveToFirst(); 
	while (!result.isAfterLast()) { 
	    int id=result.getInt(0); 
	    String name=result.getString(1); 
	    int inventory=result.getInt(2); 
	    // do something useful with these 
	    result.moveToNext(); 
	 } 
	 result.close();

最后当我们完成了对数据库的操作后，记得调用SQLiteDatabase的close()方法释放数据库连接，否则容易出现SQLiteException。

## 4. ContentProvider
### 简介
两个程序之间就没有办法对于数据进行交换？Android这么优秀的系统不会让这种情况发生的。解决这个问题主要靠ContentProvider。一个Content Provider类实现了一组标准的方法接口，从而能够让其他的应用保存或读取此Content Provider的各种数据类型。也就是说，一个程序可以通过实现一个Content Provider的抽象接口将自己的数据暴露出去。外界根本看不到，也不用看到这个应用暴露的数据在应用当中是如何存储的，或者是用数据库存储还是用文件存储，还是通过网上获得，这些一切都不重要，重要的是外界可以通过这一套标准及统一的接口和程序里的数据打交道，可以读取程序的数据，也可以删除程序的数据，当然，中间也会涉及一些权限的问题。

当应用继承ContentProvider类，并重写该类用于提供数据和存储数据的方法，就可以向其他应用共享其数据。虽然使用其他方法也可以对外共享数 据，但数据访问方式会因数据存储的方式而不同。

如：采用文件方式对外共享数据，需要进行文件操作读写数据；采用sharedpreferences共享数 据，需要使用sharedpreferences API读写数据。而使用ContentProvider共享数据的好处是统一了数据访问方式。

TBC

## 5. 网络存储数据
网络存储方式，需要与Android 网络数据包打交道
TBC

## 参考
> [Android数据存储五种方式总结](http://www.cnblogs.com/ITtangtang/p/3920916.html)
> [TBCTBC](http://www.codeceo.com/article/5-android-storage.html)