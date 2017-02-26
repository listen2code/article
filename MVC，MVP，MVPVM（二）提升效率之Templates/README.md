#####文章目录
1.[MVC，MVP，MVPVM（一）实践之路](http://www.jianshu.com/p/8ea5868d11f1)
2.MVC，MVP，MVPVM（二）提升效率之Templates

#####遗留问题
>《MVC，MVP，MVPVM（一）实践之路》中讲到如何从MVC到MVPVM的转变，将各个模块分离，职责划清。不过有个缺点，就是类爆炸，为什么代码写着写着就MVC了，就是因为一个Activity搞定，写着爽。但是，如果要实现解耦，就一定意味着会有很多不同的职能类。如果采用mvp，或mvpvm的实现方式，每次在新建一个页面就需要差不多10+个文件，虽然逻辑简单，不过全都手动创建的话，是不是觉得还是MVC好。

![so many class.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/so%20many%20class.png?raw=true)

#####解决方案
>套用定义好的代码Templates，自动生成各个职能类。

![mvpvm_templates.gif](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/gif/auto%20create%20classes.gif?raw=true)

模版地址：
https://github.com/listen2code/Test_MVPVM/tree/master/doc/MvpvmComponent

#####使用介绍
1.将写好的templates拷贝到本地
Android Studio.app/Contents/plugins/android/lib/templates/activities/
2.重启AndroidStudio，右键项目->New->Activity->MvpvmComponent
3.在打开的编辑页面输入ActivityName和layoutName，包名为"com.listen.test_mvpvm"
![templates_input.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/how%20to%20use-2.png?raw=true)

根据Templates自动生成的类
![classes_generate.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/classes_generate.png?raw=true)


```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable name="data" type="com.listen.test_mvpvm.model.viewmodel.ITestMvpvmViewModel"/>
        <variable name="presenter" type="com.listen.test_mvpvm.presenter.ITestMvpvmPresenter"/>
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/tv_test"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:text="@{data.text}"/>

        <Button
            android:id="@+id/btn_test"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_below="@id/tv_test"
            android:onClick="@{presenter.onClickAction}"
            android:text="@{data.buttonText}"/>

    </LinearLayout>
</layout>
```


```
public class TestMvpvmActivity extends AppCompatActivity implements ITestMvpvmView {

    private ActivityTestMvpvmBinding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_test_mvpvm);
        mBinding.setPresenter(new TestMvpvmPresenter(this));
    }

    @Override
    public void updateView(ITestMvpvmViewModel viewModel) {
        mBinding.setData(viewModel);
    }
}
```


```
public class TestMvpvmPresenter implements OnTestMvpvmListener, ITestMvpvmPresenter {

    private ITestMvpvmView mITestMvpvmView;
    private ITestMvpvmRepository mTestMvpvmRepository;

    public TestMvpvmPresenter(ITestMvpvmView iTestMvpvmView) {
        this.mITestMvpvmView = iTestMvpvmView;
        this.mTestMvpvmRepository = new TestMvpvmRepositoryImpl(this);
    }

    public void onClickAction(View v) {
        mTestMvpvmRepository.method("");
    }

    public void onTestMvpvmLoadSuccess(TestMvpvmModel data) {
        mITestMvpvmView.updateView(new TestMvpvmViewModel(data));
    }

    public void onTestMvpvmLoadFail(String errorMessage) {
    }
}
```


```
public class TestMvpvmRepositoryImpl implements ITestMvpvmRepository {

    private OnTestMvpvmListener mOnTestMvpvmListener;

    public TestMvpvmRepositoryImpl(OnTestMvpvmListener onTestMvpvmListener) {
        this.mOnTestMvpvmListener = onTestMvpvmListener;
    }

    public void method(String param) {
        new HttpTask() {
            @Override
            public void onRequestSuccess(TestMvpvmModel model) {
                mOnTestMvpvmListener.onTestMvpvmLoadSuccess(model);
            }
        }.path("http://xxxx/getMydeposit").execute();
    }
}
```


#####Templates介绍
Templates由3个xml描述文件，和一堆.flt文件组成。

![template_1.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/template_1.png?raw=true)
![template2.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/template2.png?raw=true)


globals.xml.ftl：主要用于定义一些全局变量，正常如果要自定义模版的话，globals文件可以直接拷贝过去，不需要什么改动。

```
<?xml version="1.0"?>
<globals>
    <global id="hasNoActionBar" type="boolean" value="false" />
    <global id="parentActivityClass" value="" />
    <global id="excludeMenu" type="boolean" value="true" />
    <global id="generateActivityTitle" type="boolean" value="false" />
    <#include "../common/common_globals.xml.ftl" />
</globals>

```

template.xml：我们在新建模版时，会有个填写信息的页面，template文件主要用于描述该页面上的元素信息。
![templates_input.png](https://github.com/listen2code/article/blob/master/MVC，MVP，MVPVM（二）提升效率之Templates/screenshot/how%20to%20use-2.png?raw=true)

```
<template
    format="5"
    revision="8"
    name="MvpvmComponent"
    minApi="7"
    minBuildApi="14"
    description="Creates mvpvm components">

    <category value="Activity" />
    <formfactor value="Mobile" />

    <!-- 上图中第一个"Component Name"输入框的描述 -->
    <parameter
        id="activityClass"// 唯一性ID，后面会在.flt中引用
        name="Component Name"// 输入框前的描述提示语
        type="string"// 输入值类型
        constraints="class|unique|nonempty"
        suggest="${layoutToActivity(activityLayoutName)}"// 输入约束
        default="Mvpvm"// 默认输入值
        help="The name of the component" />//底部提示语

    <parameter
        id="activityLayoutName"
        name="Layout Name"
        type="string"
        constraints="layout|unique|nonempty"
        suggest="${activityToLayout(activityClass)}"
        default="activity_mvp"
        help="The name of the layout" />

    <parameter
        id="isLauncher"
        name="Launcher Activity"
        type="boolean"
        default="false"
        help="If true, this activity will have a CATEGORY_LAUNCHER intent filter, making it visible in the launcher" />
    
    <parameter
        id="packageName"
        name="Package name"
        type="string"
        constraints="package"
        suggest="com.listen.test_mvpvm"
        default="com.listen.test_mvpvm" />

    <globals file="globals.xml.ftl" />
    <execute file="recipe.xml.ftl" />

</template>
```

recipe.xml.flt：将root目录下的.flt文件解析成.java文件，并打开。

```
<?xml version="1.0"?>
<recipe>

    <!-- 处理模版 -->
    <instantiate from="root/src/app_package/MvpActivity.java.ftl"
        to="${escapeXmlAttribute(srcOut)}/view/${activityClass}Activity.java" />

    <!-- 打开xin -->
    <open file="${escapeXmlAttribute(srcOut)}/view/${activityClass}Activity.java" />

</recipe>
```

MvpActivity.java.flt文件示例，引用template中的
"@{activityClass}"，"${activityLayoutName}"，就是在输入页面填写的ActivityName，和layoutName。将"@{}"解析后生成完整的Activity.java。其实Templates的原理，就是将你需要生成的java文件的公用部分抽取出来，用"@{}"占位符，替代会变化的keyWord，写成一个个.flt模版文件，同时在输入面板中将keyWord输入，并通过template.xml，recipe.xml等描述文件，最终合成我们需要的java文件。

```
public class ${activityClass}Activity extends AppCompatActivity implements I${activityClass}View {

    private Activity${activityClass}Binding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.${activityLayoutName});
        mBinding.setPresenter(new ${activityClass}Presenter(this));
    }

    @Override
    public void updateView(I${activityClass}ViewModel viewModel) {
        mBinding.setData(viewModel);
    }
}
```

通过定义Templates，在每次新建一个页面时节省了大量重复的类创建工作，我们可以运用Templates定义各种页面模版，可以大幅度的提升开发效率。在日常工作中，应该尽可能减少一些纯体力的，重复的代码拷贝。不过为优化，为解耦而增加再多类，分再多模块都是值得的。

>Templates更详细介绍可以参考鸿洋的文章，这里不做赘述：http://blog.csdn.net/lmj623565791/article/details/51635533

