![题图.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/堆内存分析.jpg)

###前言

>通过Android Studio的Memory Monitor工具，对各种数据类型，如：boolean，int，float，long，SparseArray，HashMap等在内存的占用情况进行分析。对一些特定场景下的代码编写，如：String拼接，OnClickListener等所消耗的内存情况进行分析。通过分析，更好的了解了不同情况下堆内存是如何分配的，也确切验证了以往诸多的代码经验，为高效合理的利用内存奠定基础。

###Memory Monitor的基本使用

* 新建MainActivity，启动APP
```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

* 在 Android Monitor -> Monitors -> Memory 中，点击"initiate GC"，先手动GC一次，把没用的内存进行回收。
![step1_gc.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/step1_gc.png)

* 点击"Dump Java Heap"，生成.hprof（hprof文件为特定时间点，Java进程的内存快照）
![step2_dump.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/step2_dump.png)

以下是根据.hprof文件生成的内存分析表，本文主要关注Shallow Size和Retained Size，其他column含义可以参考[官方-HPROF Viewer and Analyzer](https://developer.android.com/studio/profile/am-hprof.html#hprof-analyzing)
![heap_nothing.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/heap_nothing.png)


###Shallow Size和Retained Size
>  Shallow Size：对象自身占用的内存大小，不包括它引用的对象
Retained Size：对象自身占用的内存大小，加上它直接或间接引用的对象大小
Dominating Size：管辖的内存大小，大部分情况和Retained一致

![shalow_and_retain.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/shalow_and_retain.png)

因为可以通过GC Roots直接访问，所以左图的obj3不是蓝色节点；而右图却是蓝色，因为它已经被包含在 Retained size 中。

|| Shallow Size | Retained Size（左） | Retained Size（右） |
| --- | --- | --- |--- |
| obj1 | obj1 | obj1+obj2+obj4 | obj1+obj2+obj3+obj4 |
| obj2 | obj2 | obj2+obj4 | obj2+obj3+obj4 |

###案例分析
>  如图[heap_nothing.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/heap_nothing.png)，在MainActivity在新建的时候，初始占用内存1776（以下案例分析基于红米note3机型）。

*  case 1：空对象TestModel+未初始化。

```
public class TestModel {
}

public class MainActivity extends AppCompatActivity {
    private TestModel mModel;
    ...onCreate()
}
```

![case1_TestModel.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case1_TestModel.png)

> 只定义TestModel成员变量的情况下，内存占用1780=初始内存+引用类型（4）。所以在项目发版前，要把一些没有使用到的变量都清理一遍，积少成多，免得造成内存浪费。

*  case 2：空对象TestModel+初始化。

```
public class MainActivity extends AppCompatActivity {
    private TestModel mModel = new TestModel();
    ...onCreate()
}
```

![case2_TestModel.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case2_TestModel.png)

> 内存占用1788=case1+类信息（8），说明调用new时，即使是空对象，也需要8字节左右的堆空间用于描述该对象的类信息。基于Java是在new的时候才去申请堆空间的特性，在开发中，可以考虑对象的延迟初始化，养成个好习惯，在使用到的时候才去new。

*  case3：TestModel以局部变量的方式进行定义。

```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TestModel mModel = new TestModel();
    }
}
```

![case3_TestModel.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case3_TestModel.png)

> 内存占用未变化，还是初始值1776，说明局部变量生命周期只存在于方法内部，方法结束后，即可被gc回收。除非必须，能使用局部变量的情况，就避免定义成员变量。

* case4：boolean基础类型。

```
public class MainActivity extends AppCompatActivity {
   private boolean mBoolean;
    ...onCreate()
}
```

![case4_boolean.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case4_boolean.png)

> 内存占用1777=初始状态+1，说明基础类型boolean的引用类型占用1字节。

* case5：Boolean封装类型。

```
public class MainActivity extends AppCompatActivity {
   private Boolean mBoolean;
    ...onCreate()
}
```

![case5_Boolean.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case5_Boolean.png)

> 内存占用1780=初始状态+4，装箱类型Boolean本质上也是一个对象，由case1可以推导出引用类型占用4字节。

* case6：Boolean封装类型+初始化。

```
public class MainActivity extends AppCompatActivity {
   private Boolean mBoolean = new Boolean(true);
    ...onCreate()
}
```

![case6_Boolean.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case6_Boolean.png)

![Boolean_source.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/Boolean_source.png)

> 内存占用1789=case5+9，如图，Boolean的源码中有个boolean基础类型的字段value，当调用"new Boolean(true)"的时候，根据case2可以推导，类描述信息8字节，根据case4可以推导，value基础类型占用1字节，所以总共增加9字节。

同理，可以推导出以下表格：

|| boolean/byte |short/char | int/float/String/引用类型/数组引用 | long/double/类信息 |  
| --- | --- | --- |--- |--- |
| 内存占用 | 1 | 2 | 4 | 8 |


* case7：TestModel内部类。

```
public class MainActivity extends AppCompatActivity {
    private TestModel mModel = new TestModel();
    ...onCreate()
    public class TestModel {
    }
}
```

![case7_TestModel.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case7_TestModel.png)

> 占用内存1792=case1（1780）+类信息（8）+this引用（4）。

* case8：TestModel静态内部类。

```
public class MainActivity extends AppCompatActivity {
    private TestModel mModel = new TestModel();
    ...onCreate()
    public static class TestModel {
    }
}
```

![case8_TestModel.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case8_TestModel.png)

> 占用内存1788=case1（1780）+类信息（8），静态内部类由于没有外部类的匿名this引用，少占用4字节。

* case9：HashMap和SparseArray的对比。

```
public class MainActivity extends AppCompatActivity {
    private Map<Integer, Integer> mMap = new HashMap<>();
    private SparseArray<Integer> mSparseArray = new SparseArray();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        for (int i = 0; i < 1000; i++) {
            mMap.put(i, i);
            mSparseArray.put(i, i);
        }
    }
}
```

![case9_map.png](https://github.com/listen2code/article/blob/master/性能优化（一）堆内存分析/screenshot/case9_map.png?raw=true)

> 各添加1000条数据，HashMap占用53168，SparseArray占用18653，说明使用SparseArray替代HashMap更节省内存。

* case10：OnClickListener三种写法的对比。从节省内存的角度考虑，通过方式3接口回调设置OnClickListener为最优。

写法1：匿名类
```
public class MainActivity extends AppCompatActivity {
    private Button mButton;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mButton = (Button) findViewById(R.id.button);
        mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(MainActivity.this, "hello", Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

![case10_1_listener.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case10_1_listener.png)

> 内存占用=MainActivity（1780）+MainActivity$1（12）=1792。

写法2：成员变量类
```
public class MainActivity extends AppCompatActivity {
    private Button mButton;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mButton = (Button) findViewById(R.id.button);
        mButton.setOnClickListener(mOnClickListener);
    }

    private View.OnClickListener mOnClickListener = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "hello", Toast.LENGTH_SHORT).show();
        }
    };
}
```

![case10_2_listener.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case10_2_listener.png)

> 内存占用=MainActivity（1784，包含4字节的成员变量）+MainActivity$1（12）=1796。

写法3：接口回调

```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mButton = (Button) findViewById(R.id.button);
        mButton.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        Toast.makeText(MainActivity.this, "hello", Toast.LENGTH_SHORT).show();
    }
}
```

![case10_3_listener.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case10_3_listener.png)

> 内存占用=1780（减少1个成员变量，避免通过new创建新的对象，内存占用最少）。

* case11：String的初始化。

case11_1：
```
public class MainActivity extends AppCompatActivity {
   private String mStr = "aaaaa";
    ...onCreate()
}
```

![case11_1_String.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case11_1_String.png)

case11_2：
```
public class MainActivity extends AppCompatActivity {
   private String mStr = new String("aaaaa");
    ...onCreate()
}
```

![case11_2_String.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case11_2_String.png)

> 
* "aaaaa"这个String为何占用26字节？按以上方式分析，至少占用内存30=类信息（8）+count（4）+hashCode（4）+char[]引用（4）+char[]数组（10），为何少了4字节？
* 直接赋值的方式会将"aaaaa"加入到字符串常量池，不占用堆空间；而case11_2的内存占用为 1806=case11_1+26，说明通过new String方式创建的字符串会在堆内存开辟空间。

* case12：String的拼接。

case12_1：基于case11_1，作字符串"+"拼接。

```
public class MainActivity extends AppCompatActivity {
   private String mStr = "aaaaa";
   protected void onCreate(Bundle savedInstanceState) {
       ...
      mStr += "c";
   }
}
```

![case12_1_String.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case12_1_String.png)

> 可以发现，拼接后内存占用1808=case11_1（1780）+28，而这28的空间正好是"aaaaac"的内存大小，也就是说在"+"拼接的时候，产生了一个临时的变量用于存储"aaaaac"的结果，并赋值给mStr。印证了《Effective in Java》的第51条中所说"由于字符串不可变，当2个字符串被连接在一起时，他们的内容都要被拷贝"。同时在[浅谈StringBuilder](http://www.jianshu.com/p/160c9be0b132)这篇文章中也讲到了"+"拼接的时候，会转化为StringBuilder，再通过toString创建一个新的String对象。

case12_2：用StringBuilder进行字符串拼接。

case12_2_1：初始化1个空的StringBuilder
```
public class MainActivity extends AppCompatActivity {
  private StringBuilder mStringBuilder = new StringBuilder();
    ...onCreate()
}
```

![case12_2_1_StringBuilder.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case12_2_1_StringBuilder.png)

> 一个空的StringBuilder就占49字节，类信息（8）+count（4）+shared（1）+value引用（4）+value[]数组（32）=49。value这个字符数组占用了32字节，而我们最多也就添加"aaaaac"6个字符，所以这里可以通过new StringBuilder(6)初始化字符数组的大小，避免浪费。

case12_2_2：使用StringBuilder进行"aaaaa"+"c"的字符串拼接。

```
public class MainActivity extends AppCompatActivity {
    private StringBuilder mStringBuilder = new StringBuilder(6);
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        mStringBuilder.append("aaaaa");
        mStringBuilder.append("c");
    }
}
```

![case12_2_2_StringBuilder.png](https://raw.githubusercontent.com/listen2code/article/master/性能优化（一）堆内存分析/screenshot/case12_2_2_StringBuilder.png)

> 首先在StringBuilder初始化的时候设置了字符数组大小为6，所以StringBuilder的初始内存占用就变小了，而在完成append("aaaaa")，append("c")之后，只要当前字符数组的容量够用，就不会继续扩容，避免了String拼接时，内存浪费的问题。当然前提是控制好StringBuilder的char[]初始容量，不然扩容后也会空余一些闲置内存。

###总结
> 
1.谨慎创建成员变量：不管有用没用，非基础类型的成员变量只要定义了，至少需要4字节，基础类型成员变量占用大小各不一样。尽量使用局部变量，缩短变量生命周期，促使GC更快回收。
2.谨慎new：如case2的TestModel，不管该对象是否为空，至少8字节的类信息占用。如case10的Listener，尽量避免不必要的new。考虑对象的延迟初始化，只有真正使用的时候才new。
3.除非必要，否则尽量使用基础类型，避免使用装箱类型。
4.少用内部类：内部类如果不需要访问到外部类的成员时，可以抽取成独立外部类，或加static，减少一个this引用（4字节），也可以避免内存泄漏。
5.使用google推荐的数据集合类型SparseArray，ArrayMap替代HashMap。
6.从节省内存的角度考虑，通过接口回调的方式设置OnClickListener为最优。
7.通过StringBuilder替代String进行字符串拼接，最好预先设置好StringBuilder的容量。

###未完，待续...

###参考
[官方-HPROF Viewer and Analyzer](https://developer.android.com/studio/profile/am-hprof.html#hprof-analyzing)