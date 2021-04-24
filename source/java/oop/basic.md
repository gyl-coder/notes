---
title: 基础
type: guide
order: 2
---

## Java 如何高效进行数组拷贝

- System.arraycopy()
- Arrays.copyOf()
- clone
- for循环逐一赋值

> java.util.Arrays类中提供了一个copyOf()方法，通过这个方法可以实现数组复制。这个方法的原理是内部创建了一个数组对象，然后通过System.arraycopy来实现数组复制.
> System.arraycop方法是通过native方法,Java虚拟机内部实现的。

<code>参考文章：</code>

- https://blog.csdn.net/weixin_43896318/article/details/104645939
- https://zhuanlan.zhihu.com/p/37346162