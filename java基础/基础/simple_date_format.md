# SimpleDateFormat线程安全问题

>SimpleDateFormat是线程不安全的类，一般不要定义为static变量，如果定义为static，
必须通过加锁等方式保证线程安全。
---

```
import java.text.ParseException;
import java.text.SimpleDateFormat;
public class SimpleDateFormatDemo {

    // (1)创建单例实例
    static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    
    public static void test1() {
        // (2)创建多个线程，并启动
        for (int i = 0; i < 10; ++i) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        // (3)使用单例日期实例解析文本
                        System.out.println(sdf.parse("2019-03-07 15:17:27"));
                    } catch (ParseException e) {
                        e.printStackTrace();
                    }
                }
            });
            // (4)启动线程
            thread.start();
        }
    }

    public static void main(String[] args) {
        test1();
    }
}
```

上面这段代码由多线程共享一个非线程安全的sdf实例，结果可想而知。解决办法

- 1、sdf添加volatile关键字，由于volatile的可见性特性并且也适合读多写少的业务逻辑，可以很好的解决线程安全问题
- 2、ThreadLocal

看一下ThreadLocal的解决办法

>ThreadLocal，也叫做线程本地变量或者线程本地存储，ThreadLocal为变量在每个线程中都创建了一个副本，
那么每个线程可以访问自己内部的副本变量，这样就避免了SimpleDateFormat线程不安全的问题。
---

```
    // (1)创建localDateFormat实例
	static ThreadLocal<DateFormat> localDateFormat = new ThreadLocal<DateFormat>() {
		@Override
		public SimpleDateFormat initialValue() {
			return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		}
	};

	public static void test() {
		// (2)创建多个线程，并启动
		for (int i = 0; i < 10; ++i) {
			Thread thread = new Thread(new Runnable() {
				public void run() {
					try {
						// (3)解析文本
						System.out.println(localDateFormat.get().parse("2019-03-07 15:17:27"));
					} catch (Exception e) {
						e.printStackTrace();
					}
				}
			});
			// (4)启动线程
			thread.start();
		}
	}

	public static void main(String[] args) {
		test();
	}
```
