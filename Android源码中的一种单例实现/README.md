> 单例模式的实现方式有懒汉，饿汉，双重校验锁，枚举，内部类等等，写法就不全部列举了。Android源码中有一个单例辅助类/frameworks/base/core/java/android/util/Singleton.java，可以实现懒汉式单例，写法挺奇特的，虽然是个hide类，不过拷贝出来就可以用了。


```
package android.util;

/**
 * Singleton helper class for lazily initialization.
 *
 * Modeled after frameworks/base/include/utils/Singleton.h
 *
 * @hide
 */
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
} 
    
```


正常懒汉单例
 
```
public class SingletonDemo {
    private static SingletonDemo mInstance;

    public static final SingletonDemo get() {
        synchronized (SingletonDemo.class) {
            if (mInstance == null) {
                mInstance = new SingletonDemo();
            }
            return mInstance;
        }
    }
} 
```

懒汉+双重校验单例
 
```
public class SingletonDemo {
    private static SingletonDemo mInstance;

    public static final SingletonDemo get() {
        if (mInstance == null) {
            synchronized (SingletonDemo.class) {
                if (mInstance == null) {
                    mInstance = new SingletonDemo();
                }
            }
        }
        return mInstance;
    }
} 
```
       
     
变种懒汉单例

```
public class SingletonDemo {

    public static final SingletonDemo get() {
        return INSTANCE.get();
    }

    private static final Singleton<SingletonDemo> INSTANCE = new Singleton<SingletonDemo>() {
        protected SingletonDemo create() {
            return new SingletonDemo();
        }
    };
}
```
   
懒汉式单例一般都会再加个双重校验的判断，避免每次调用get()都加锁，影响性能，Android源码中Singleton.java工具类并没有做双重校验（看了下googlesource中的[Singleton.java](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/Singleton.java)，也是6年前提交的代码了），所以我们在将Singleton.java拷贝出来使用的时候可以加个双重校验优化下。Singleton.java封装了create，get模版，及get中的校验逻辑，从而SingletonDemo中的实现代码就可以更加的简单：1.create()一个单例对象，2.在需要的时候get()。