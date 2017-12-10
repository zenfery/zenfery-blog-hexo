---
title: Java注解(2)-注解处理器(运行时|RetentionPolicy.RUNTIME)
permalink: annotation-2
categories:
  - JAVA
  - Annotation
date: 2015-02-02 06:22:22
updated: 2015-02-02 20:46:00
author: Zenfery
tags:
  - Java-Core
thumbnail:
blogexcerpt: 如果没有用来读取注解的工具，那注解将没有任何作用，它也不会比注释更有用。读取注解的工具叫作注解处理器。Java提供了两种方式来处理注解：第一种是利用运行时反射机制；另一种是使用Java提供的API来处理编译期的注解。
---

如果没有用来读取注解的工具，那注解将没有任何作用，它也不会比注释更有用。读取注解的工具叫作注解处理器。Java提供了两种方式来处理注解：第一种是利用运行时反射机制；另一种是使用Java提供的API来处理编译期的注解。

## 反射机制方式的注解处理器
仅当定义的注解的@Retention为RUNTIME时，才能够通过运行时的反射机制来处理注解。下面结合例子来说明这种方式的处理方法。

Java中的反射API（如java.lang.Class、java.lang.reflect.Field等）都实现了接口java.lang.reflect.AnnotatedElement，来提供获取类、方法和域上的注解的实用方法。

## 示例：通过JavaBean上定义的注解来动态生成相应的SQL

### 定义注解

- **类注解映射表名**。定义注解@TableSQL，只定义一个value值来映射表名，默认值为空，如果程序不给此值，将使用类名（小写）来作为表名。

``` java
package com.zenfery.example.annotation.sql;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)//定义注解应用于类
@Retention(RetentionPolicy.RUNTIME)//定义注解在JVM运行时保留
public @interface TableSQL {
    String value() default "";//指定对应的表名
}
```


- **属性与字段对应注解**。定义注解@TableColumnSQL的目标为FIELD，仅能在类的属性上使用；value()属性定义对应的字段名；constraint()定义字段的约束，它是由注解@Constraint定义。

``` java
package com.zenfery.example.annotation.sql;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)//定义注解应用于成员变量
@Retention(RetentionPolicy.RUNTIME)//定义注解在JVM运行时保留
public @interface TableColumnSQL {
 String value() default "";
 Constraint constraint() default @Constraint();
}
```

@Constraint注解仅定义了两个注解元素，allowNull()指定字段是否允许为空值；isPrimary()指定字段是否是主键。
``` java
package com.zenfery.example.annotation.sql;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)//定义注解应用于成员变量
@Retention(RetentionPolicy.RUNTIME)//定义注解在JVM运行时保留
public @interface Constraint {
 boolean allowNull() default true; //是否允许为空
 boolean isPrimary() default false; //是否为主键
}
```

### 使用注解

- 定义User类。类上使用@TableSQL来注解映射表名；字段id和name使用@TableColumnSQL来注解映射字段名。

``` java
package com.zenfery.example.annotation.clazz;

import com.zenfery.example.annotation.sql.Constraint;
import com.zenfery.example.annotation.sql.TableColumnSQL;
import com.zenfery.example.annotation.sql.TableSQL;

/**
 * 定义User类与数据库映射
 * @author zenfery
 */
@TableSQL()
public class User {

 //定义id字段，与表user的列id相映射，指定约束为：不为空，为主键。
 @TableColumnSQL(value="id",constraint=@Constraint(allowNull=false,isPrimary=true))
 String id;

 //只为注解指定value字段，可省略value。
 @TableColumnSQL("name")
 String name;
}
```

### 编写注解处理器

- 在以上工作做完后，注解没有任何意义（它没有做任何事情），下面来为RUNTIME级别的注解来编写处理器，在此编写的处理器仅为演示理解用。
    1. 获取使用了注解的User类。
    2. 根据类上的注解@TableSQL获取表名。
    3. 根据类中所有的字段上的注解@TableColumnSQL来获取字段名，并获取字段的特性。
    4. 根据获取的表名和字段名拼接SQL。
    5. 打印SQL。

``` java
package com.zenfery.example.annotation.clazz;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

import com.zenfery.example.annotation.sql.Constraint;
import com.zenfery.example.annotation.sql.TableColumnSQL;
import com.zenfery.example.annotation.sql.TableSQL;

public class TableSQLHandler {

  /**
   * 注解处理器：读取User类中注解，生成对应的SQL并打印出来
   * 在此假设表的所有字段均为varchar(10)
   * @since JDK 1.6
   * @param args
   */
  public static void main(String[] args) {
    /**
     * 注解的@Retention均为JVM保留注解（RetentionPolicy.RUNTIME）
     * ，在此直接使用main方法启动JVM，通过Java提供的反射机制来处
     * 理。
     */
    try {
      //指定在JVM需要处理注解的类
      Class userClass = null;
      //userClass = Class.forName("com.zenfery.example.annotation.clazz.User");
      userClass = User.class;

      //打印所有类的注解
      Annotation[] annotations = userClass.getDeclaredAnnotations();
      for(int i=0; i<annotations.length; i++){
        System.out.println("注解["+(i+1)+"] = "+annotations[i].toString());
      }

      //检查类是否有@TableSQL注解
      if( userClass.isAnnotationPresent(TableSQL.class) ){
        //sql
        String sql = "\nCREATE TABLE ";
        //注解了TableSQL注解
        TableSQL ts = (TableSQL)userClass.getAnnotation(TableSQL.class);
        String tableName = ts.value();
        if("".equals(tableName)){//如果获取的值为TableSQL的默认值，则使用类名来做为表名
          tableName = userClass.getSimpleName().toLowerCase();
        }
        System.out.println("获取"+userClass.getName()+"对应的表名为："+tableName);
        sql += tableName + " ( \n";

        //从User类的属性中获取需要与数据库映射的字段
        Field[] fields = userClass.getDeclaredFields();

        List<String> primaryKeys = new ArrayList<String>();//存储主键
        for(int i=0; i<fields.length; i++){
          Field field = fields[i];
          if( field.isAnnotationPresent(TableColumnSQL.class) ){
            TableColumnSQL tcs = (TableColumnSQL)field.getAnnotation(TableColumnSQL.class);
            String fieldName = tcs.value();//表中的字段名
            Constraint c = tcs.constraint();//字段对应的约束
            boolean allowNull = c.allowNull();//是否可为空
            boolean isPrimary = c.isPrimary();//是否为主键

            //拼接SQL
            sql += "\t" + fieldName +" VARCHAR(10)";
            if(!allowNull) sql += " NOT NULL";//不允许为空
            if(i<fields.length-1) sql+= ",\n";

            //主键
            if(isPrimary) primaryKeys.add(fieldName);
          }else{
            System.out.println("字段"+field.getName()+"未使用注解@TableColumnSQL！");
          }
        }
        if(primaryKeys.size()>0){
          StringBuilder keys = new StringBuilder();
          for(int k=0; k<primaryKeys.size(); k++){
            keys.append(primaryKeys.get(k));
            if(k<primaryKeys.size()-1)keys.append(",");
          }

          sql += ",\n\tPRIMARY KEY "+keys.toString();
        }
        sql += "\n) DEFAULT CHARSET=utf8";
        // ====> 打印SQL
        System.out.println("生成的SQL："+sql);

      }else{
        System.out.println("警告："+userClass.getName()+"未使用@TableSQL注解！");
      }
    } catch (Exception e) {
      e.printStackTrace();
    }

  }

}
```

运行TableSQLHandler的main()方法，输出：
```
注解[1] = @com.zenfery.example.annotation.sql.TableSQL(value=)
获取com.zenfery.example.annotation.clazz.User对应的表名为：user
生成的SQL：
CREATE TABLE user (
 id VARCHAR(10) NOT NULL,
 name VARCHAR(10),
 PRIMARY KEY id
) DEFAULT CHARSET=utf8
```
