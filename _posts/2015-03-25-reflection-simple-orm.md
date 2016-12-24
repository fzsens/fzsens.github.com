## 基于反射实现简单的ORM
面向对象语言的一个优势在于抽象，将现实的业务需求抽象和构造成为程序的可用实体对象，使程序员在编程时候的思考方式更加贴近现实业务的运转方式。另一方面数据在使用过程中又经常会需要持久化，一般可以简单理解成将数据存储到数据库中。现在流行的关系数据库(NoSQL另做讨论)主要以表(具有行和列)为对外提供展示和保存数据的基础结构，行表述存储的数据量，列标识数据的模式(`Schema`)。如果使用面向对象语言和关系型数据库进行数据对应的话，即为列(模式)，对应抽象编程对象中的属性。

![来自wikipedia](https://i.imgur.com/hZwW64i.png)

ORM(`Object Relection Mapping`)，关系对象映射，用于实现面向对象编程语言里不同类型系统的数据之间的转换。在Java社区中有很多成熟的ORM框架，诸如Mybatis,Hibernate等，通过配置或者注解的形式，或自动或手动，将对象和数据库建立映射。从而简化开发中对数据持久问题的处理。

spring jdbctemplate是由spring项目派生出来的一个组件，对jdbc进行了轻度封装，用于简化Java对数据库的操作。一个常用的jdbctemplate使用例子如下：

````java
  Actor actor = this.jdbcTemplate.queryForObject(
  "select first_name, last_name from t_actor where id = ?",
  new Object[]{1212L},
  new RowMapper<Actor>() {
      public Actor mapRow(ResultSet rs, int rowNum) throws SQLException {
          Actor actor = new Actor();
          actor.setFirstName(rs.getString("first_name"));
          actor.setLastName(rs.getString("last_name"));
          return actor;
      }
  });
````

查询语句执行的结果存储在`ResultSet`中与`JDBC`一致，查询之后的`ResultSet`即模拟了关系数据库中的表的结构。通过`getXXX(“columnName”)`; 可获得对应行的列值，这部分关于`ResutSet`的操作过程实际上就是完成了从关系到对象的映射过程。如果对象中自带获取关系的元数据(参考元编程)，这个ORM过程就可以通过程序自动构建完成。

类似Lisp或者Ruby，通过自身极强的可拓展性，可以很方便地实现自我管理。而Java对于运行时`Rumtime`代码提供更改的方式就显得比较笨重，较为常用的方法即为放射`Reflection`,放射提供了一整套方法可以在代码运行时候进行访问和更改。注解`Annotation`则提供了可以向代码中附着元数据的方式。下面就使用放射和注解的方式，尝试在spring jdbctemplate构建一个简单的ORM框架。关系数据库选择Oracle，其中的一些语句迁移到其他的数据库上，可能需要进行一些更改，本文主要以说明问题为主。

- 首先定义元数据，即为Annotation，对于简单的数据库操作(CRUD)来讲，关注的主要有：表名，字段名，主键(用于插入和删除)，主键生成规则(Oracle中可选取Sequence)等。这些原数据搭建了代码对象到数据库的桥梁。

````java
  //表名
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  public @interface TableName {
    public String value() default "";
    
  }

  //标识是否主键
  @Target(ElementType.FIELD)
  @Retention(RetentionPolicy.RUNTIME)
  public @interface PrimaryKey {
  }

  //用于主键生成的序列Oracle关联
  @Target(ElementType.FIELD)
  @Retention(RetentionPolicy.RUNTIME)
  public @interface OracleSEQ {
    public String value() default "";
  }

  //字段名
  @Target(ElementType.FIELD)
  @Retention(RetentionPolicy.RUNTIME)
  public @interface Column {
    public String value() default "";
  }
````

- 创建业务对象，将元数据和编码绑定。

````java
  @TableName("T_USR")
  public class TUsrModel extends BaseModel{
    
    @PrimaryKey
    @Column("A_ID")
    @OracleSEQ("SQ_T_USR")
    private Long id;
    
    @Column("A_PID")
    private String pid;
    
    @Column("A_USERNAME")
    private String username;
    
    @Column("A_PWD")
    private String pwd;

    /**
    省略getter和setter方法
    **/
  }
````

- 如何将一个具有元数据的Java实体对象持久化到数据库中，最简单的方式即为通过放射构建出插入语句，将实体对象中的值作为插入条件

````java
  public class SQLContext {
      /**
       * 执行的sql
       * */
      private StringBuilder sql;

      /**
       * 主键名称
       * */
      private String primaryKey;

      /**
       * 参数，对应sql中的?号
       * */
      private List<Object> params;

      public SQLContext(StringBuilder sql, String primaryKey, List<Object> params) {
          this.sql = sql;
          this.primaryKey = primaryKey;
          this.params = params;
      }
      
  }
````

SQLContext存储需要执行的SQL语句以及对应的参数

````java
public class EntityUtils {
  
  //表名缓存
  private static final Map<String,String> tableNameCaches = 
      Collections.synchronizedMap(new WeakHashMap<String,String>());
  
  //表主键缓存
  private static final Map<String,String> tablePrimaryKeyCaches = 
      Collections.synchronizedMap(new WeakHashMap<String,String>());
  
  //字段名缓存
  private static final Map<String, String> columnNameCaches = 
      Collections.synchronizedMap(new WeakHashMap<String,String>());
  //字段类型缓存
  private static final Map<String, Class<?>> columnTypeCaches = 
      Collections.synchronizedMap(new WeakHashMap<String,Class<?>>());
  /**
   * 获取表名
   * @param entityClass
   * @return
   */
  public static String getTableName(Class<?> clazz) {
    String key = clazz.getName();
    if(tableNameCaches.get(key)!= null) {
      return tableNameCaches.get(key);
    }
    if (clazz.isAnnotationPresent(TableName.class)) {
      String tableName = clazz.getAnnotation(TableName.class).value();
      tableNameCaches.put(key, tableName);
      return tableName;
    } else {
      return null;
    }
  }
  
  /**
   * 获取主键
   * @param entityClass
   * @return
   */
  public static String getPrimaryKey(Class<?> clazz) {
    String key = clazz.getName();
    if(tablePrimaryKeyCaches.get(key)!= null) {
      return tablePrimaryKeyCaches.get(key);
    }
    
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
      if(field.isAnnotationPresent(PrimaryKey.class)) {
        String primaryKey = field.getAnnotation(Column.class).value();
        tablePrimaryKeyCaches.put(key, primaryKey);
        return primaryKey;
      }
    }
    return null;
  }
  
  /**
   * 获取SEQ
   * @param clazz
   * @return
   */
  public static String getOracleSEQ(Class<?> clazz) {
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
      if(field.isAnnotationPresent(OracleSEQ.class)) {
        return field.getAnnotation(OracleSEQ.class).value();
      }
    }
    return null;
  }
  
  public static List<String> getFieldNames(Class<?> clazz) {
    List<String> fieldNames = new ArrayList<String>();
    BeanInfo beanInfo = ClassUtils.getSelfBeanInfo(clazz);
    PropertyDescriptor[] properties = beanInfo.getPropertyDescriptors();
    for (PropertyDescriptor pd : properties) {
      fieldNames.add(pd.getName());
    }
    return fieldNames;
  }
  
  /**
   * 获取ColumnName
   * @param clazz
   * @param fieldName
   * @return
   */
  public static String getColumnName(Class<?> clazz,String fieldName) {
    String key = clazz.getName()+":"+fieldName;
    if(columnNameCaches.get(key)!=null) {
      return columnNameCaches.get(key);
    }
    Field field = null;
    String fielName = null;
    while(clazz != null && field == null) {
      try {
        field = clazz.getDeclaredField(fieldName);
      } catch (SecurityException e) {
        e.printStackTrace();
        throw new RuntimeException("安全设置问题");
      } catch (NoSuchFieldException e) {
        clazz = clazz.getSuperclass();
      }
    }
    if(field == null) {
      throw new RuntimeException("没有找到字段名为 "+ fieldName + " 的字段");
    } else {
      fielName = field.getAnnotation(Column.class).value();
      columnNameCaches.put(key,fielName);
    }
    return fielName;
  }
  
  public static boolean isPrimaryKey(Class<?> clazz,String fieldName) {
    Field field = null;
    while(clazz != null && field == null) {
      try {
        field = clazz.getDeclaredField(fieldName);
      } catch (SecurityException e) {
        e.printStackTrace();
        throw new RuntimeException("安全设置问题");
      } catch (NoSuchFieldException e) {
        clazz = clazz.getSuperclass();
      }
    }
    if(field == null) {
      throw new RuntimeException("没有找到字段名为 "+ fieldName + " 的字段");
    }
    return field.isAnnotationPresent(OracleSEQ.class);
  }
  
  public static Class<?> getColumnType(Class<?> clazz,String fieldName) {
    String key = clazz.getName()+":"+fieldName;
    if(columnTypeCaches.get(key)!=null) {
      return columnTypeCaches.get(key);
    }
    Field field = null;
    Class<?> fieldType = null;
    
    while(clazz != null && field == null) {
      try {
        field = clazz.getDeclaredField(fieldName);
      } catch (SecurityException e) {
        e.printStackTrace();
        throw new RuntimeException("安全设置问题");
      } catch (NoSuchFieldException e) {
        clazz = clazz.getSuperclass();
      }
    }
    if(field == null) {
      throw new RuntimeException("没有找到字段名为 "+ fieldName + " 的字段");
    }else {
      fieldType = field.getType();
      columnTypeCaches.put(key, fieldType);
    }
    return fieldType;
  }
  
  /**
   * 获取操作方法
   */
  public static String getOperationName(Class<?> clazz,String fieldName) {
  
    Field field = null;
    
    while(clazz != null && field == null) {
      try {
        field = clazz.getDeclaredField(fieldName);
      } catch (SecurityException e) {
        e.printStackTrace();
        throw new RuntimeException("安全设置问题");
      } catch (NoSuchFieldException e) {
        clazz = clazz.getSuperclass();
      }
    }
    if(field == null) {
      throw new RuntimeException("没有找到名为 "+ fieldName + " 的方法");
    }
    return field.getAnnotation(OperationName.class).value();
  }
}
````

EntityUtils 提供获取对象元数据的方法封装，并提供并发安全的缓存处理。

````java
  public class ClassUtils {
    private static final Map<Class<?>, BeanInfo> classCache = 
        Collections.synchronizedMap(new WeakHashMap<Class<?>, BeanInfo>());
      public static BeanInfo getSelfBeanInfo(Class<?> clazz) {
          try {
              BeanInfo beanInfo;
              if (classCache.get(clazz) == null) {
                  beanInfo = Introspector.getBeanInfo(clazz, BaseModel.class.getSuperclass());
                  classCache.put(clazz, beanInfo);
                  Class<?> classToFlush = clazz;
                  do {
                      Introspector.flushFromCaches(classToFlush);
                      classToFlush = classToFlush.getSuperclass();
                  } while (classToFlush != null);
              } else {
                  beanInfo = classCache.get(clazz);
              }
              return beanInfo;
          } catch (IntrospectionException e) {
            //修改修改
              return null;
          }
      }
      public static Object newInstance(Class<?> clazz) {
          try {
              return clazz.newInstance();
          } catch (Exception e) {
            //修改修改
            return null;
          }
      }
  }
````

ClassUtils获取Java对象的属性即为`BeanInfo`

````java
  public static SQLContext buildInsertSql(Object entity) {
      Class<?> clazz = entity.getClass();
      String tableName = EntityUtils.getTableName(clazz);
      String primaryName = EntityUtils.getPrimaryKey(clazz);
      String seqName = EntityUtils.getOracleSEQ(clazz);
      StringBuilder sql = new StringBuilder("insert into ");
      List<Object> params = new ArrayList<Object>();
      sql.append(tableName);
      //获取属性信息
      BeanInfo beanInfo = ClassUtils.getSelfBeanInfo(clazz);
      PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
      sql.append("(");
      StringBuilder args = new StringBuilder();
      args.append("(");
      for (PropertyDescriptor pd : pds) {
          Object value = getReadMethodValue(pd.getReadMethod(), entity);
          /*default param*/
          value = getDefaultValue(value, pd.getName());
          //deal with oracle seq
          if(EntityUtils.isPrimaryKey(clazz, pd.getName())) {
            args.append(seqName+".Nextval");
          } else {
                args.append("?");
                params.add(value);
          }
          sql.append(EntityUtils.getColumnName(clazz,pd.getName()));
          sql.append(",");
          args.append(",");
      }
      sql.deleteCharAt(sql.length() - 1);
      args.deleteCharAt(args.length() - 1);
      args.append(")");
      sql.append(")");
      sql.append(" values ");
      sql.append(args);
      return new SQLContext(sql, primaryName, params);
  }
````

通过对对象属性的操作和构建，便可获得插入语句

````java
 public Long insert(Object entity) {
    final SQLContext sqlContext = SQLHelper.buildInsertSql(entity);
    KeyHolder keyHolder = new GeneratedKeyHolder();
    jdbcTemplate.update(new PreparedStatementCreator() {
        public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
            PreparedStatement ps = con.prepareStatement(sqlContext.getSql().toString(),
                new String[] { sqlContext.getPrimaryKey() });
            int index = 0;
            for (Object param : sqlContext.getParams()) {
                index++;
                if(param instanceof Date) {
                  Date date = (Date)param;
                  //对插入条件的Date需要转换为sql的TimeStamp类型
                  ps.setObject(index, new java.sql.Timestamp(date.getTime()));
                } else {
                  ps.setObject(index, param);
                }
            }
            return ps;
        }
    }, keyHolder);
    return keyHolder.getKey().longValue();
 }
````

实际使用中，另外一个使用频率较高的方法为查询方法，如之前jdbctemplate的使用例子所示。需要手工进行关系对象的映射，而现在对象本身就带有映射信息。因此这个动作也可以自动构建完成。

````java
public class DefaultRowMapper<T> implements RowMapper<T>{

    /** 转换的目标对象 */
    private Class<?>    clazz;

    public DefaultRowMapper(Class<?> clazz) {
        this.clazz = clazz;
    }

    @Override
    public T mapRow(ResultSet resultSet, int i) throws SQLException {
        Object entity = ClassUtils.newInstance(this.clazz);
        BeanInfo beanInfo = ClassUtils.getSelfBeanInfo(this.clazz);
        PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
        for (PropertyDescriptor pd : pds) {
            String column = EntityUtils.getColumnName(this.clazz,pd.getName());
            Class clazzType = EntityUtils.getColumnType(this.clazz,pd.getName());
            Method writeMethod = pd.getWriteMethod();
            if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
                writeMethod.setAccessible(true);
            }
            try {
              Object object = resultSet.getObject(column);
              if(clazzType.equals(Long.class)) {
                object = resultSet.getLong(column);
              }
              if(clazzType.equals(Double.class)) {
                object = resultSet.getDouble(column);
              }
              if(clazzType.equals(Integer.class)) {
                object = resultSet.getInt(column);
              }
              if(clazzType.equals(Float.class)) {
                object = resultSet.getFloat(column);
              }
                writeMethod.invoke(entity, object);
            } catch (Exception e) {
              e.printStackTrace();
                throw new SQLException(e);
            }
        }
        return (T)entity;
    }
}
````

实现spring jdbctemplate的`RowMapper`接口，其中的mapRow类即为处理对象和关系映射的方法。构建的基础也是基于元数据之上。

````java
public static final String SELECT_USER_BY_USER_PWD = ""
      + " select * from t_usr t "
      + " where t.A_USERNAME = :username"
      + " and t.A_PWD = :pwd ";

MapSqlParameterSource param = new MapSqlParameterSource();
    param.addValue("username", username);
    param.addValue("pwd", password);
    List<TUsrModel> userlist = new ArrayList<TUsrModel>();
    userlist = this.getNamedJdbcTemplate().query(
        SELECT_USER_BY_USER_PWD,
        param,
        new DefaultRowMapper<TUsrModel>(TUsrModel.class));
````

即可实现查询时候的映射。
其中涉及到的事务控制和其他安全控制，此处就不一一列举，通过spring的IOC，以及`DataSourceTransactionManager`，可以构建出自己的事务管理系统。

总结：相比较Ruby的元编程，使用Java来处理程序自管理显得额外繁琐，异常机制也带来了大量不必要的代码冗余。但是如果能够善用反射和注解，也能够给工作带来极大的便利。  

基于上面的原理，实现的`ficus`工具，可以在 [https://github.com/fzsens/ficus](https://github.com/fzsens/ficus "ficus") 获取