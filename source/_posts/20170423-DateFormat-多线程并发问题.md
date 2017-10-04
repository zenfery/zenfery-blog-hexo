---
title: DateFormat 多线程并发问题
date: 2017-04-23 22:22:22
updated: 2017-09-30 12:46:00
permalink: df-thread
categories:
  - JAVA
  - J2SE
author: Zenfery
tags:
  - 多线程
  - 并发
thumbnail:
blogexcerpt: 在日常开发中，java.text.DateFormat 应该算是使用频率比较高的一个工具类，经常会使用它 将 Date 对象转换成字符串日期，或者将字符串日期转化成 Date 对象。
---
在日常开发中，java.text.DateFormat 应该算是使用频率比较高的一个工具类，经常会使用它 将 Date 对象转换成字符串日期，或者将字符串日期转化成 Date 对象。先来看一段眼熟的代码：
{% codeblock lang:java %}
public abstract class DateUtils {
    private static final DateFormat dateFormatForDay = new SimpleDateFormat("yyyyMMdd");
    public static String formatForDay(Date date){
        return dateFormatForDay.format(date);
    }
}
{% endcodeblock %}

类 DateUtils 的方法 formatForDay() 在多线程的情况下，可能会得不到想要的结果。给一段多线程并发访问 formatForDay() 方法的示例：
{% codeblock lang:java %}
public class DateFormatExample {

    static final class RunThread implements Runnable{
        private int i;
        public RunThread(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            int count = 0;
            while(true){		
                // create a date
                Calendar c = Calendar.getInstance();
                c.add(Calendar.DAY_OF_MONTH, -i);
                Date d = c.getTime();
                // expect string
                String origianlDate = new SimpleDateFormat("yyyyMMdd").format(d);
                // DateUtils format string
                String afterDate = DateUtils.formatForDay(d);

                if(!origianlDate.equals(afterDate)){
                    System.out.println("RunThread\["+i+"\]\["+count+"] origianlDate = "+origianlDate
                        +", afterDate = "+afterDate);
                    System.exit(0);
                }
                count++;
            }
        }
    }

    public static void main(String[] args) {
        for(int i=0; i<100; i++){
            new Thread(new RunThread(i)).start();
        }
    }
{% endcodeblock %}

示例代码，创建了100个 RunThread 线程，每个线程根据 i 的值创建不同的日期，并将日期格式化成字符串，和原日期进行对比，若不相等，则打印退出。某次执行的结果见下：
{% codeblock lang:text %}
RunThread[85][1] origianlDate = 20170128, afterDate = 20170122
RunThread[91][5] origianlDate = 20170122, afterDate = 20170222
RunThread[32][7] origianlDate = 20170322, afterDate = 20170114
RunThread[60][3] origianlDate = 20170222, afterDate = 20170202
RunThread[1][0] origianlDate = 20170422, afterDate = 20170401
{% endcodeblock %}
从结果可以看出，格式化后的日期和传给 format() 的日期不一致。

## 原因分析
类 DateFormat 有个 Calendar 成员变量：
{% codeblock lang:java %}
public abstract class DateFormat extends Format {

    protected Calendar calendar;

    // ... other code
}
{% endcodeblock %}

调用 format() 方法，会调用实现类 SimpleDateFormat 的  format(Date date, StringBuffer toAppendTo, FieldDelegate delegate) 方法：
{% codeblock lang:java %}
public class SimpleDateFormat extends DateFormat {
    //... other code

    // Called from Format after creating a FieldDelegate
    private StringBuffer format(Date date, StringBuffer toAppendTo,
                                FieldDelegate delegate) {
        // Convert input date to time field list
        calendar.setTime(date);

        //... other code
    }
}
{% endcodeblock %}
每次调用 format() 方法，会将要格式化的日期设置到成员变量 calendar 中，然后再对其进行格式化，此类实现未进行线程同步，是非线程安全的。

当然，调用 parse() 方法，将 字符串转化成日期，也会有同样的非线程安全问题。

## 解决方案
- 每次进行格式化日期调用时，均 new 一个 SimpleDateFormat 对象；缺点是在高并发的情况下，就会频繁创建和销毁对旬，造成开销。
- 使用 synchronized 关键字 或 Lock 给静态方法加上同步；缺点是在高并发的情况下，所有的线程在此处会引起资源的竞争。
- 使用 ThreadLocal 对象创建静态 DateFormat 。这样在高并发情况下，有多少个线程，就会创建多少个 DateFormat 对象，既不会无限制创建、销毁对象，也不会引起对象的多线程竞争（此种方案适用于使用线程池的情况）。如下：
{% codeblock lang:java %}
public abstract class DateUtils {
    private static final ThreadLocal dateFormatForDay = new ThreadLocal();

    public static String formatForDay(Date date) {
        if (dateFormatForDay.get() == null) {
            dateFormatForDay.set(new SimpleDateFormat("yyyyMMdd"));
        }
        return dateFormatForDay.get().format(date);
    }
}
{% endcodeblock %}
- 根据格式化 pattern ，将 DateFormat 对象存储在内存中，使用时直接获取。如下：
{% codeblock lang:java %}
public abstract class PatternDateUtils {

	private static final Map<String, DateFormat> dfs = new HashMap<String, DateFormat>();
	private static final List<String> patterns = new ArrayList<String>();

	public static String format(Date date, String pattern){
		DateFormat df = dfs.get(pattern);
		if(df == null){
			df = new SimpleDateFormat(pattern);
			dfs.put(pattern, df);
		}
		return df.format(date);
	}
}
{% endcodeblock %}
