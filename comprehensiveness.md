**如何定位问题？ 解决思路和处理方法？**



**最近在做的比较熟悉的项目是哪个？画一下项目技术架构图**



**单机上一个线程池正在处理服务，如果忽然断电怎么办（正在处理和阻塞队列里的请求怎么办）**



**接口如何处理重复请求？**



**知道osgi吗 它是如何实现的**



**知道字节码吗？字节码获取方式都有哪些？**

字节码也就是class文件。
在程序中获取类对象的方法就是获取字节码的方式。

```
1.类名.class,例如System.class;
2.对象.getClass(),例如New Date().getClass()
3.Class.forName("类名")，例如Class.forName("Java.Util.Date");
123
```

字节码角度讲int y=5单纯压入int类型变量，赋值为5，而Integer x=5则会调用Integer.valueOf()的方法。x==y会使用到if_icmp方法比较。

Integer x = 5， int y = 5 比较x == y 都经过了哪些步骤?



