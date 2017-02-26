###DataBinding整体使用流程
![整体流程图.png](https://github.com/listen2code/article/blob/master/DataBinding学习笔记（一）源码分析/screenshot/整体流程图.png?raw=true)

###开发阶段
#####UserModel.java
```
public class UserModel {
    public String name;
    public String nickName;
    public int age;

    public UserModel(String name, String nickName, int age) {
        this.name = name;
        this.age = age;
        this.nickName = nickName;
    }

    public boolean isAge18() {
        return age >= 18;
    }
}
```

#####activity_main.xml
>在xml中使用"@{}"标识符

```
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable name="user" type="com.listen.test_databinding.UserModel"/>
        <variable name="testClick" type="android.view.View.OnClickListener"/>
        <import type="android.view.View"/>
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        tools:context="com.listen.test_databinding.MainActivity">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{"名字" + user.name}'/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{user.nickName}'
            android:visibility="@{null == user.nickName ? View.VISIBLE : View.GONE}"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{user.isAge18() ? "man" : "boy"}'/>

        <Button
            android:id="@+id/btn_test"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:onClick="@{testClick}" android:text="测试"/>
    </LinearLayout>
</layout>
```

#####MainActivity.java

```
public class MainActivity extends AppCompatActivity {

    private ActivityMainBinding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);

        final UserModel user = new UserModel("listen", "ls", 18);
        mBinding.setUser(user);
        mBinding.setTestClick(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Toast.makeText(MainActivity.this, "testClick", Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```


###编译阶段

####1.Databinding会自动解析识别xml中的"@{}"标识符，并在以下目录生成2个xml文件

>1.build/intermediates/data-binding-layout-out/activity_main.xml
>2.build/intermediates/data-binding-info/debug/activity_main-layout.xml

######activity_main.xml
>带“@{}”的xml文件是android系统无法识别的，为了向后兼容，需要在编译期统一转换成系统能识别的标准xml布局，而原先在布局中添加的"@{}"，"@{三目运算符}"等信息，则会存储在activity_main-layout.xml中。

```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"    
    android:orientation="vertical"
    android:tag="layout/activity_main_0"
    tools:context="com.listen.test_databinding.MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:tag="binding_1"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:tag="binding_2"
    />

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:tag="binding_3"/>

    <Button
        android:id="@+id/btn_test"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:tag="binding_4" android:text="测试"/>
</LinearLayout>
```

######activity_main-layout.xml（xml描述文件）
>1.任何view只要用到了"@{}"标识，就会在activity_main-layout.xml中生成target描述，并根据该view在parent中的位置生成"binding_[index]"标识，并设置在tag中。
2.如果一个view即没有设置"android:id"，也没有使用"@{}"标识，则不会在activity_main-layout.xml中生成这个view的target描述。
3.LinearLayout比较特殊，并没有设置"android:id"，也没有使用"@{}"，但还是会生成一个默认的tag="layout/activity_main_0"，表示它是根布局，在ViewDataBinding.java实例化时，需要判断根布局的tag，后面源码会分析到。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Layout absoluteFilePath="/Users/lisong/Documents/AndroidStudioWorkSpace/Test_Databinding/app/src/main/res/layout/activity_main.xml" directory="layout"
        isMerge="false"
        layout="activity_main" modulePackage="com.listen.test_databinding">
    <Variables name="user" declared="true" type="com.listen.test_databinding.UserModel">
        ...
    </Variables>
    <Variables name="testClick" declared="true" type="android.view.View.OnClickListener">
        ...
    </Variables>
    <Imports name="View" type="android.view.View">
        ...
    </Imports>
    <Targets>
        <Target tag="layout/activity_main_0" view="LinearLayout">
            <Expressions/>
            ...
        </Target>
        <Target tag="binding_1" view="TextView">
            <Expressions>
                <Expression attribute="android:text" text=""名字" + user.name">
                    ...
                </Expression>
            </Expressions>
        </Target>
        <Target tag="binding_2" view="TextView">
            <Expressions>
                <Expression attribute="android:text" text="user.nickName">
                    ...
                </Expression>
                <Expression attribute="android:visibility"
                            text="null == user.nickName ? View.VISIBLE : View.GONE">
                    ...
                </Expression>
            </Expressions>
        </Target>
        <Target tag="binding_3" view="TextView">
            <Expressions>
                <Expression attribute="android:text"
                            text="user.isAge18() ? "man" : "boy"">
                    ...
                </Expression>
            </Expressions>
        </Target>
        <Target id="@+id/btn_test" tag="binding_4" view="Button">
            <Expressions>
                <Expression attribute="android:onClick" text="testClick">
                    ...
                </Expression>
            </Expressions>
        </Target>
    </Targets>
</Layout>
```

####2.生成ActivityMainBinding.java和BR.java
>DataBinding根据解析后的activity_main-layout.xml，和layout下的activity_main.xml文件，生成build/intermediates/classes/debug/[项目路径]/databinding/
ActivityMainBinding.java和BR.java

![ActivityMainBinding目录.png](https://github.com/listen2code/article/blob/master/DataBinding学习笔记（一）源码分析/screenshot/ActivityMainBinding目录.png?raw=true)

######ActivityMainBinding主要具备以下功能
>1.作为view和model的连接器，持有需要展示的数据和views的成员变量
2.将数据映射到view（就是setText，setOnClick等）
3.在UI线程更新数据

######BR.java就是一个常量类
>可以通过binding.setVariable(BRuser, new User())进行数据更新

```
public class BR {
    public static final int _all = 0;
    public static final int testClick = 1;
    public static final int user = 2;

    public BR() {
    }
}
```
```
public boolean setVariable(int variableId, Object variable) {
    switch(variableId) {
        case BR.testClick :
            setTestClick((android.view.View.OnClickListener) variable);
            return true;
        case BR.user :
            setUser((com.listen.test_databinding.UserModel) variable);
            return true;
    }
    return false;
}
```

###运行阶段
>Databinding框架最主要做的事，就是以上2步，接下来就是在代码中调用生成的ViewDataBinding，并进行数据绑定操作。

######DataBindingUtil是一切的入口

```
ActivityMainBinding mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
```
```
public static <T extends ViewDataBinding> T setContentView(Activity activity, int layoutId, DataBindingComponent bindingComponent) {
    activity.setContentView(layoutId);// 最终调用的还是activity.setContentView()，不过这里的layoutId是已经去掉"@{}"的标准xml布局
    View decorView = activity.getWindow().getDecorView();
    ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);//获取根顶级容器view
    return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
}
```
######取出布局的rootView，调用ActivityMainBinding.bind()

```
private static <T extends ViewDataBinding> T bindToAddedViews(android.databinding.DataBindingComponent component, ViewGroup parent, int startChildren, int layoutId) {
    final int endChildren = parent.getChildCount();
    final int childrenAdded = endChildren - startChildren;
    if (childrenAdded == 1) {
        // 从顶级容器view中获取当前布局的rootView，调用bind方法
        final View childView = parent.getChildAt(endChildren - 1);
        return bind(component, childView, layoutId);
    } else {
        ...
    }
}

static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
                                          int layoutId) {
    // sMapper = DataBinderMapper.java
    return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
}

/**  DataBinderMapper.java */
public android.databinding.ViewDataBinding getDataBinder(android.databinding.DataBindingComponent bindingComponent, android.view.View view, int layoutId) {
    switch(layoutId) {
        case com.listen.test_databinding.R.layout.activity_main:
            // 将rootView传递给ActivityMainBinding.bind()
            return com.listen.test_databinding.databinding.ActivityMainBinding.bind(view, bindingComponent);
    }
    return null;
}
```

此处做了rootView的判断，如果传递过来的不是当前ViewDataBinding绑定的布局，则抛异常。所以即使rootView没有设置id，及"@{}"，在info-layout.xml中也会生成相应的target描述。

```
public static ActivityMainBinding bind(View view, DataBindingComponent bindingComponent) {
    if(!"layout/activity_main_0".equals(view.getTag())) {
        throw new RuntimeException("view tag isn\'t correct on view:" + view.getTag());
    } else {
        return new ActivityMainBinding(bindingComponent, view); // ActivityMainBinding在此处初始化
    }
}
```

这里需要特别注意的是在编译期自动生成的activity_main.xml文件中自动添加了tag="binding_1"，"binding_2"等，其实在初始化完这些view后，都已经清空，是不影响我们在代码中设置tag的；不过rootView并没有清除tag（就是xml布局最外层的layout），如果>=14以上版本，在代码里设置setTag(R.id.databinding,"anything")，或，<14版本，在代码里设置setTag("anything")，则会报错，so，这个tag是由DataBinding占着的，使用上得小心。

```
public ActivityMainBinding(android.databinding.DataBindingComponent bindingComponent, View root) {
    super(bindingComponent, root, 0);
    // 遍历布局，找到所有views，并存储在bindings[]中，5表示布局一共有5个view，sIncludes存储被include进     // 来的布局，sViewsWithIds存储设置了"android:id"，但是没有用到"@{}"的view
    Object[] bindings = mapBindings(bindingComponent, root, 5, sIncludes, sViewsWithIds);

    // 将bindings[]中的view取出，赋值给当前各个view的成员变量，并清除tag，避免冲突
    this.btnTest = (Button)bindings[4];
    this.btnTest.setTag((Object)null);
    this.mboundView0 = (LinearLayout)bindings[0];
    this.mboundView0.setTag((Object)null);
    this.mboundView1 = (TextView)bindings[1];
    this.mboundView1.setTag((Object)null);
    this.mboundView2 = (TextView)bindings[2];
    this.mboundView2.setTag((Object)null);
    this.mboundView3 = (TextView)bindings[3];
    this.mboundView3.setTag((Object)null);
    this.setRootTag(root);
    /**
    ViewDatabBinding.java
    protected void setRootTag(View view) {
        //private static final boolean USE_TAG_ID = DataBinderMapper.TARGET_MIN_SDK >= 14;
        if (USE_TAG_ID) {
        view.setTag(R.id.dataBinding, this);
        } else {
        view.setTag(this);
        }
    }
    */

    
    // 请求刷新，实现数据与view的绑定
    this.invalidateAll();
}
```

mapBindings()，其实就是递归遍历view树的过程，不过不是byId，而是byTag，寻找以"binding_"开头的view，并取出"binding_[索引]"中的索引，赋值给binding[]数组。所有的view只在一次遍历中获得，而如果是用findViewById的方式，每次调用都需要遍历一次view树[性能对比]。需要特别注意的是binding数组的元素不一定都是view或viewGroup，如果有include布局的时候binding数组存储的可能是include布局的viewDataBinding对象。

```
private static void mapBindings(DataBindingComponent bindingComponent, View view, Object[] bindings, ViewDataBinding.IncludedLayouts includes, SparseIntArray viewsWithIds, boolean isRoot) {
    ViewDataBinding existingBinding = getBinding(view);
    if(existingBinding == null) {
        Object objTag = view.getTag();
        String tag = objTag instanceof String?(String)objTag:null;
        
        if(isRoot && tag != null && tag.startsWith("layout")) {
            // 如果是rootView，则从"layout/activity_main_0"中取出索引"0"，设置到bindings[0]中
            viewGroup = tag.lastIndexOf(95);
            count = parseTagInt(tag, viewGroup + 1);
            if(bindings[count] == null) {
                bindings[count] = view;
            }
            ...
        } else if(tag != null && tag.startsWith("binding_")) {
            // 同样判断tag，取出"binding_1"，"bingding_2"中的索引，赋值到bindings[]中
            viewGroup = parseTagInt(tag, BINDING_NUMBER_START);
            if(bindings[viewGroup] == null) {
                bindings[viewGroup] = view;
            }
            ...
        }
        
        // isBound=false，说明当前的view既不是根布局，也没有用到"@{}"（如果有用到就会生成"binding_"
        // 的tag）；则通过id获取该view，并设置到bingding[]
        // 如果存在设置了id，但是没有“@{}”的view会被添加到sViewsWithIds中，如果
        // "binding_[index]"的index最大为3，则view的起始index设置为4。
        // static {
        //    sIncludes = null;
        //    sViewsWithIds = new android.util.SparseIntArray();
        //    sViewsWithIds.put(R.id.btn_test, 4);
        //}
        if(!isBound) {
            viewGroup = view.getId();
            if(viewGroup > 0 && viewsWithIds != null && (count = viewsWithIds.get(viewGroup, -1)) >= 0 && bindings[count] == null) {
                bindings[count] = view;
            }
        }

        if(view instanceof ViewGroup) {
            // 如果是view是个viewGroup，则遍历子view
            ViewGroup var25 = (ViewGroup)view;
            count = var25.getChildCount();
            ...

            for(int i = 0; i < count; ++i) {
                View child = var25.getChildAt(i);
                boolean isInclude = false;
                if(indexInIncludes >= 0 && child.getTag() instanceof String) {
                    String childTag = (String)child.getTag();
                    if(childTag.endsWith("_0") && childTag.startsWith("layout") && childTag.indexOf(47) > 0) {
                        // 如果当前view也是一个rootView，则判断tag中是否有include标识信息
                        // 如果包含include标签，生成的info-layout文件应该是以下样式：
                        // <Target include="include_main" tag="layout/activity_main_0">
                        // </Target>
                        int includeIndex = findIncludeIndex(childTag, minInclude, includes, indexInIncludes);
                        if(includeIndex >= 0) {
                            isInclude = true;
                            ...
                            // 如果包含include信息，则重新调用DataBindingUtil.bind()生成ViewDataBinding，重复当前流程，
                            // 不过当前的bindings[index]就不是一个view，而是一个viewDataBinding
                            bindings[index] = DataBindingUtil.bind(bindingComponent, child, layoutId);
                            ...
                        }
                    }
                }

                if(!isInclude) {
                    // 如果只是一个viewGroup，不是include进来的布局，则重新调用mapBindings，只是isRoot=false，则会上面进入"binding_"的判断逻辑
                    mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false);
                }
            }
        }

    }
}

```
#####view遍历流程图
![view遍历流程.png](https://github.com/listen2code/article/blob/master/DataBinding学习笔记（一）源码分析/screenshot/view遍历流程.png?raw=true)

View都找到了，现在该是时候设置listener，data的时候了。这时候会通过invalidateAll()请求数据更新，层层调用后，还是回到了ActivityMainBinding的executeBindings()，在这个方法里将更新后的model数据，onclick等重新设置到Textview，Button上，完成了model->view的单向绑定。

```

// 子类：xxxViewDataBinding extends ViewDataBinding
public void invalidateAll() {
    synchronized(this) {
        this.mDirtyFlags = 4L;
    }

    this.requestRebind();
}

// 父类：ViewDataBinding.java
// 通过handler.post()执行mRebindRunnable
protected void requestRebind() {
    ...
    mUIThreadHandler.post(mRebindRunnable);
}

// mRebindRunnable调用了executePendingBindings()
private final Runnable mRebindRunnable = new Runnable() {
    @Override
    public void run() {
        ...
        executePendingBindings();
    }
};

// executePendingBindings调用了executeBindings()
public void executePendingBindings() {
    ...
    executeBindings();
    ...
}

// 子类：xxxViewDataBinding extends ViewDataBinding
protected void executeBindings() {
    ...
    // 当调用ViewDataBinding.setUser(new User())时，就是给成员变量mUser赋值，在这里获取this.mUser
    UserModel user = this.mUser;

    if((dirtyFlags & 6L) != 0L) {
        // 获取并构建数据，所以model中的字段要么为public，要么提供一个getter方法，不然这里无法获取
        // 可以看到，在xml中的@{}表达式，此时已经解析成对应的方法isAge18，user.name等
        if(user != null) {
            userIsAge18User = user.isAge18();
            nameUser = user.name;
            nickNameUser = user.nickName;
        }

        ...
        // 获取并构建数据
        userIsAge18UserStrin = userIsAge18User?"man":"boy";
        stringNameUser = "名字" + nameUser;
        ObjectnullNickNameUs1 = null == nickNameUser;
        ...

        objectnullNickNameUs = ObjectnullNickNameUs1?0:8;
    }

    // 设置listener
    if((dirtyFlags & 5L) != 0L) {
        this.btnTest.setOnClickListener(testClick);
    }

    // 通过TextViewBindingAdapter将数据设置到TextView上
    if((dirtyFlags & 6L) != 0L) {
        TextViewBindingAdapter.setText(this.mboundView1, stringNameUser);
        TextViewBindingAdapter.setText(this.mboundView2, nickNameUser);
        this.mboundView2.setVisibility(objectnullNickNameUs);
        TextViewBindingAdapter.setText(this.mboundView3, userIsAge18UserStrin);
    }

}
```

######以上便是当我们通过DataBindingUtil.setContentView()对Databinding进行初始化，以及当我们获取到最新数据，通过Binding.setModel进行数据更新时的操作流程。
![数据绑定流程.png](https://github.com/listen2code/article/blob/master/DataBinding学习笔记（一）源码分析/screenshot/数据绑定流程.png?raw=true)

###参考

英文官方文档
https://developer.android.com/topic/libraries/data-binding/index.html

Google开发团队介绍DataDinding使用
https://realm.io/cn/news/data-binding-android-boyar-mount/?utm_source=tuicool&utm_medium=referral

QQ音乐团队分享，比较贴近源码的介绍
http://gold.xitu.io/entry/57e48e7ba22b9d006139c60b

