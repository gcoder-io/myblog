>数据库作为App缓存设计的首选，存在一些开发的陷阱，同时需要考虑性能、开发效率和可维护性，笔者建议自行搭建数据库管理类，同时配合成熟的开源ORM框架快速搭建数据库模块。
本方案采用ORM框架greenDAO，greenDAO是成熟的ORM框架，在性能和内存占用上很出色
https://github.com/greenrobot/greenDAO

##Android数据库存在的问题：
1. SQLite是**数据库级别**锁，同一个数据库连接可以多线程操作，同时读、写、读写操作，SQLite底层做了同步处理；
2. 多个数据库连接，只可以多线程读操作，不可写或读写操作，否则会抛出锁表异常
`android.database.sqlite.SQLiteDatabaseLockedException: database is locked`
总结：多个数据库连接不能同时操作涉及到写的操作，多线程写操作必须保证一个数据库连接。

##解决方案：
###1. 同步锁方案：
考虑性能，建议使用**ReadWriteLock**，ReadWriteLock可以并发读，可以大幅减少读操作造成的锁开销；
* 在Application onCreate方法中创建全局唯一的ReadWriteLock；
```java
public static ReadWriteLock lock = new ReentrantReadWriteLock(false);
```
* 在DAO中调用Application的ReadWriteLock，
* 读操作前调用`lock.readLock().lock();`加锁，执行完毕调后调用
`lock.readLock.unlock();`解锁；
* 写操作调用`lock.writeLock().lock();` 加锁，执行完毕后调用
`lock.writeLock().unlock();` 解锁；
* 在每次读写操作前需要获取数据库连接，用完后关闭释放资源。

```java
public class UserInfoDAO {
  private Context context;
  private ReadWriteLock lock;
  public TestDAO(Context context, ReadWriteLock lock){
    this.context = context;
    this.lock = lock;
  }

  public SQLiteDatabase getConnection() {
    SQLiteDatabase sqliteDatabase = null;
    try {
      sqliteDatabase = new DatabaseHelper(context).getWritableDatabase();
    } catch (Exception e) {
    }
    return sqliteDatabase;
  }

  public void insert(UserInfo userInfo){
    lock.writeLock().lock(); 
    SQLiteDatabase db = null;
    try {
      ContentValues values = new ContentValues();
      values.put("name", userInfo.getName());
      values.put("age", userInfo.getAge);
      db = getConnection();
      if(db != null && db.isOpen()){
        db.insert("userInfo", null, values);
      }
    } catch (Exception e) {
    }finally{
       if (db != null && db.isOpen()) {
         db.close();
       }
       myLock.writeLock().unlock();   
    }
  }

  public List<UserInfo> query(){
    List<UserInfo> list = new ArrayList<UserInfo>();
    lock.readLock().lock(); 
    SQLiteDatabase db = null;
    try {
      db = getConnection();
      if(db != null && db.isOpen()){
        Cursor c = db.rawQuery("SELECT * FROM userInfo", null);
        UserInfo info;
        while (c.moveToNext()) {
          info = new UserInfo();
          info.setName(c.getString(c.getColumnIndex("name")));
          info.setAge(c.getInt(c.getColumnIndex("age")));
          list.add(info);
        }
      }
    } catch (Exception e) {
    }finally{
       if (db != null && db.isOpen()) {
         db.close();
       }
       lock.readLock().unlock();   
    }
    return list;
  }
}
```

###2. 多数据库方案：
每张table对应一个单独的db，由于SQLite在db层做了同步处理，故此方案支持读、写并发操作。

###3. 单数据库连接方案：
* 此方案采用整个Application保证只有一个数据库连接，利用SQLite单数据库连接支持同时读写的特性；
* 对数据库操作抽象：数据库连接获取/关闭、数据库连接是否可用、获取数据库操作DAO管理类。DaoSession是greenDAO中DAO管理类，包含DAO的注册、获取、清理。

```java
public interface DatabaseManager {

    /**
     * 获取数据库连接
     */
    void startup(Context mContext);

    /**
     * 关闭数据库
     */
    void shutdown();
    
    /**
     * 检查数据库状态是否可用
     * @return
     */
    boolean checkDBStatus();

    /**
     * 获取greenDAO DAO管理类
     * @return
     */
    DaoSession getDaoSession();
}
```

* 实现类DatabaseManagerImpl

```java
public class DatabaseManagerImpl implements DatabaseManager {
    private static final String DBNAME = "test_db";
    private OpenHelper helper;
    private SQLiteDatabase db;
    private DaoSession daoSession;
    private Context mContext;

    @Override
    public void startup(Context mContext) {
        this.mContext = mContext;
        if (LogUtils.DEBUG) {
            QueryBuilder.LOG_SQL = true;
            QueryBuilder.LOG_VALUES = true;
        }
        getOpenHelper();
        db = helper.getWritableDatabase();
        DaoMaster daoMaster = new DaoMaster(db);
        daoSession = daoMaster.newSession();
    }

    @Override
    public void shutdown() {
        if (daoSession != null) {
            daoSession.clear();
        }
        if (db != null && db.isOpen()) {
            db.close();
        }
    }

    private void getOpenHelper() {
        if (helper != null) {
            return;
        }
        // TODO: release版本请使用ReleaseOpenHelper
        if (LogUtils.DEBUG) {
            helper = new DaoMaster.DevOpenHelper(mContext, DBNAME, null);
        } else {
            helper = new ReleaseOpenHelper(mContext, DBNAME, null);
        }
    }

    @Override
    public boolean checkDBStatus() {
        if (db == null || !db.isOpen()) {
            getOpenHelper();
            db = helper.getWritableDatabase();
        }
        if (db.isOpen()) {
            return true;
        } else {
            if (LogUtils.DEBUG) {
                LogUtils.d("database open fail.");
            }
            return false;
        }
    }

    @Override
    public synchronized DaoSession getDaoSession() {
        if (!checkDBStatus()) {
            return null;
        }
        if (daoSession == null) {
            DaoMaster daoMaster = new DaoMaster(db);
            daoSession = daoMaster.newSession();
        }
        return daoSession;
    }
}
```

* 在Application onCreate方法中创建DatabaseManager，并调用初始化方法
startup，在此方法中获取数据库连接并保持，不释放，并通过数据库连接创建greenDao中DAO管理类;
* 在业务中需要数据库的地方，只需要调用getDaoSession获取DAO管理类调用相应的DAO即可。