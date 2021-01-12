# 动态植入java Class 并运行

动态植入java代码的方式有多种，这里只介绍GroovyClassLoader方法

## 1、GroovyClassLoader导入

```
//在build.gradle中导入（AndroidStudio的导入方式，其他编译工具可以使用其他方式）
implementation 'org.codehaus.groovy:groovy-all:2.2.2'
```

```
//国内可能需要使用镜像
maven { url 'https://maven.aliyun.com/repository/google'}
maven { url 'https://maven.aliyun.com/repository/jcenter'}
maven { url 'http://maven.aliyun.com/nexus/content/groups/public'}
```

## 2、GroovyClassesTest类搭建

```java
        //groovy提供了一种将字符串文本代码直接转换成Java Class对象的功能
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        //里面的文本是Java代码,但是我们可以看到这是一个字符串我们可以直接生成对应的Class<?>对象,而不需要我们写一个.java文件
        Class<?> clazz = groovyClassLoader.parseClass("package com.xxl.job.core.glue;\n" +
                "\n" +
                "public class Main {\n" +
                "\n" +
                "    public int age = 22;\n" +
                "    \n" +
                "    public void sayHello() {\n" +
                "        System.out.println(\"年龄是:\" + age);\n" +
                "\n" +
                "    }\n" +
                "\n" +
                "}\n"
        );
//反射获取实例
        Object obj = clazz.newInstance();
//反射获取方法对象
        Method method = clazz.getDeclaredMethod("sayHello");
//调用方法
        method.invoke(obj);
    }
```



## 3、升级使用

```java
        //groovy提供了一种将字符串文本代码直接转换成Java Class对象的功能
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        //里面的文本是Java代码,但是我们可以看到这是一个字符串我们可以直接生成对应的Class<?>对象,而不需要我们写一个.java文件
        Class<?> clazz = groovyClassLoader.parseClass("package com.xxl.job.core.glue;\n" +
                "\n" +
                "public class Main {\n" +
                "\n" +
                "    public int age = 22;\n" +
                "    \n" +
                "    public void sayHello() {\n" +
                "        System.out.println(\"年龄是:\" + age);\n" +

                "\n" +
                "Thread thread1 = new Thread(new Runnable() {\n" +
                "            @Override\n" +
                "            public void run() {\n" +
                "        System.out.println(\"年龄是:\" + age);\n" +
                "                System.out.println(\"hhhhh\");\n" +
                "            }\n" +
                "        });\n" +
                "\n" +
                "        thread1.run();"+

                "        System.out.println(\"年龄是:\" + age);\n" +
                "getResult(1, 5);\n"+
                "    }\n" +
                "\n" +
                "public static int getResult(int a, int b){\n" +
                "        int c = a + b;\n" +
                "        System.out.println( \" c =  \"+c);\n" +
                "        return c;\n" +
                "    }\n"+
                "}\n"
        );
//反射获取实例
        Object obj = clazz.newInstance();
//反射获取方法对象
        Method method = clazz.getDeclaredMethod("sayHello");
//调用方法

//获取含有参数的方法对象（int.class, int.class是两个参数的参数类型）
        Method method1 = clazz.getDeclaredMethod("getResult", int.class, int.class);
        List<Object> listValue = new ArrayList<Object>();
//        method1.getParameterTypes();//获取参数类型，按照类型添加
        listValue.add(2);//与上面参数类型对应
        listValue.add(5);//与上面参数类型对应
//调用方法并返回结果
        Object invoke = method1.invoke(obj, listValue.toArray());
        System.out.println(invoke);//打印结果（强转定义类型）

//返回由此 方法实例表示的注释成员的默认值。
        Object val = method.getDefaultValue();
        System.out.println(val);

    }
```