---
title: SimpleDateFormat的线程安全问题
date: 2017-11-26 20:02:33
tags: SimpleDateFormat
categories: java基础
---
> SimpleDateFormat在进行日期格式转换时用的很多，但是 DateFormat 和 SimpleDateFormat 类不都是线程安全的，在多线程环境下调用 format() 和 parse() 方法应该使用同步代码来避免问题
<!-- more -->

多线程测试
----

```java
 * <p>
 * Date formats are not synchronized.
 * It is recommended to create separate format instances for each thread.
 * If multiple threads access a format concurrently, it must be synchronized
 * externally.
 *
 * @see          <a href="http://java.sun.com/docs/books/tutorial/i18n/format/simpleDateFormat.html">Java Tutorial</a>
 * @see          java.util.Calendar
 * @see          java.util.TimeZone
 * @see          DateFormat
 * @see          DateFormatSymbols
 * @author       Mark Davis, Chen-Lieh Huang, Alan Liu
 */
public class SimpleDateFormat extends DateFormat {
...................
｝
```
在注视中，明确说明If multiple threads access a format concurrently, it must be synchronized
测试代码：
```java
public static final String PATTEN = "yyyy-MM-dd hh:mm:ss";

    public static final SimpleDateFormat sdf = new SimpleDateFormat(PATTEN);

    public static final CountDownLatch countDownLatch = new CountDownLatch(100);

    @Test
    public void testJDKSimpleDateFormat() {
        //jdk的实现方式，会有线程安全的问题
        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println(sdf.parseObject("2013-05-24 06:02:20"));
                    } catch (ParseException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
```
上面的测试方法在多线程下会出现如下异常：
```java
Exception in thread "Thread-1" Exception in thread "Thread-3" java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Long.parseLong(Long.java:453)
	at java.lang.Long.parseLong(Long.java:483)
	at java.text.DigitList.getLong(DigitList.java:194)
	at java.text.DecimalFormat.parse(DecimalFormat.java:1316)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1793)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1455)
	at java.text.DateFormat.parseObject(DateFormat.java:415)
	at java.text.Format.parseObject(Format.java:243)
	at com.smart.tools.SimpleDateFormatTest$1.run(SimpleDateFormatTest.java:32)
	at java.lang.Thread.run(Thread.java:745)
Exception in thread "Thread-8" java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Long.parseLong(Long.java:453)
	at java.lang.Long.parseLong(Long.java:483)
	at java.text.DigitList.getLong(DigitList.java:194)
	at java.text.DecimalFormat.parse(DecimalFormat.java:1316)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2088)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1455)
	at java.text.DateFormat.parseObject(DateFormat.java:415)
	at java.text.Format.parseObject(Format.java:243)
	at com.smart.tools.SimpleDateFormatTest$1.run(SimpleDateFormatTest.java:32)
	at java.lang.Thread.run(Thread.java:745)
java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Long.parseLong(Long.java:453)
	at java.lang.Long.parseLong(Long.java:483)
	at java.text.DigitList.getLong(DigitList.java:194)
	at java.text.DecimalFormat.parse(DecimalFormat.java:1316)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1793)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1455)
	at java.text.DateFormat.parseObject(DateFormat.java:415)
	at java.text.Format.parseObject(Format.java:243)
	at com.smart.tools.SimpleDateFormatTest$1.run(SimpleDateFormatTest.java:32)
	at java.lang.Thread.run(Thread.java:745)
```

根本原因
----

SimpleDateFormat继承了DateFormat,在DateFormat中定义了一个protected属性的 Calendar类的对象：calendar。只是因为Calendar累的概念复杂，牵扯到时区与本地化等等，Jdk的实现中使用了成员变量来传递参数，这就造成在多线程的时候会出现错误。

在format方法里，有这样一段代码：
```java
 private StringBuffer format(Date date, StringBuffer toAppendTo,
                                FieldDelegate delegate) {        // Convert input date to time field list  
```
calendar.setTime(date)这条语句改变了calendar，稍后，calendar还会用到（在subFormat方法里），而这就是引发问题的根源。想象一下，在一个多线程环境下，有两个线程持有了同一个SimpleDateFormat的实例，分别调用format方法：
　　线程1调用format方法，改变了calendar这个字段。
　　中断来了。
　　线程2开始执行，它也改变了calendar。
　　又中断了。
　　线程1回来了，此时，calendar已然不是它所设的值，而是走上了线程2设计的道路。如果多个线程同时争抢calendar对象，则会出现各种问题，时间不对，线程挂死等等。
　　分析一下format的实现，我们不难发现，用到成员变量calendar，唯一的好处，就是在调用subFormat时，少了一个参数，却带来了这许多的问题。其实，只要在这里用一个局部变量，一路传递下去，所有问题都将迎刃而解。
　　这个问题背后隐藏着一个更为重要的问题--无状态：无状态方法的好处之一，就是它在各种环境下，都可以安全的调用。衡量一个方法是否是有状态的，就看它是否改动了其它的东西，比如全局变量，比如实例的字段。format方法在运行过程中改动了SimpleDateFormat的calendar字段，所以，它是有状态的。

解决办法
----

 1. 每次需要的时候都创建一个新的实例
 2. 使用同步：同步SimpleDateFormat对象
 3. 使用common-lang中的api中的FastDateFormat
```java
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.CountDownLatch;

import org.apache.commons.lang3.time.FastDateFormat;
import org.junit.Test;

/**
 * 测试时间处理类的线程安全问题
 * @Description 
 * @author gaowenming
 */
public class SimpleDateFormatTest {

    public static final String PATTEN = "yyyy-MM-dd hh:mm:ss";

    public static final SimpleDateFormat sdf = new SimpleDateFormat(PATTEN);

    public static final CountDownLatch countDownLatch = new CountDownLatch(100);

    @Test
    public void testJDKSimpleDateFormat() {
        //jdk的实现方式，会有线程安全的问题
        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println(sdf.parseObject("2013-05-24 06:02:20"));
                    } catch (ParseException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }

    @Test
    public void testCommonLang() {
        //CommonLang第三方jar包实现
        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        FastDateFormat fdf = FastDateFormat.getInstance(PATTEN);
                        System.out.println(fdf.format(new Date()));
                        Date date = fdf.parse("2013-05-24 06:02:20");
                        System.out.println(date);
                        countDownLatch.countDown();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        try {
            //等待100个线程都执行完
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^");
    }
}
```