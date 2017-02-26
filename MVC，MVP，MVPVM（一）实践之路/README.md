#####文章目录
1.MVC，MVP，MVPVM（一）实践之路
2.[MVC，MVP，MVPVM（二）提升效率之Templates](http://www.jianshu.com/p/a2976caf7649)

#####简介
>分别使用MVC，MVP，MVP+VM，实践具体需求，对比优劣，逐步优化。

#####需求
>实现我的押金页面，包含未缴纳，已缴纳，免押金3种状态
1.顶部title：3种状态展示不同文案；
2.金额：已缴纳，未缴纳状态金额字号，色值不同；免押金状态不展示；
3.底部tips：已缴纳，免押金状态展示不同文案；已缴纳状态，不展示；
4.按钮：未缴纳，已缴纳状态，文案，及点击事件都不相同；

![我的押金页面.png](http://upload-images.jianshu.io/upload_images/2157048-73973e274b68ed22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####MVC的实现方式
activity_main.xml

```
<LinearLayout
            xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:tools="http://schemas.android.com/tools"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical"
            android:paddingLeft="30dp"
            android:paddingRight="30dp"
            tools:context="com.listen.test_mvc.MainActivity">
        
            <TextView
                android:id="@+id/tv_title"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="50dp"
                android:textColor="@android:color/black"
                android:textSize="20sp"
                tools:text="您需要缴纳押金"/>
        
            <TextView
                android:id="@+id/tv_money"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginBottom="50dp"
                android:layout_marginTop="50dp"
                android:textColor="@android:color/darker_gray"
                android:textSize="40sp"
                tools:text="¥200"/>
        
            <TextView
                android:id="@+id/tv_tips"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textColor="@android:color/black"
                android:textSize="16sp"
                tools:text="押金随时可退"/>
        
            <Button
                android:id="@+id/btn_pay_or_return"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginLeft="30dp"
                android:layout_marginRight="30dp"
                android:layout_marginTop="100dp"
                android:textColor="@android:color/black"
                android:textSize="16sp"
                tools:text="缴纳押金"/>
        </LinearLayout>
```     

在MainActivity中通过butterKnife框架初始化view

```
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.tv_title)
    TextView mTvTitle;
    @BindView(R.id.tv_money)
    TextView mTvMoney;
    @BindView(R.id.tv_tips)
    TextView mTvTips;
    @BindView(R.id.btn_pay_or_return)
    Button mBtnPayOrReturn;

    ///////////////////////////////////////////////////////////////////////////
    // 缴纳押金，退还押金的点击事件
    ///////////////////////////////////////////////////////////////////////////
    private View.OnClickListener mDepositPayClickListener = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "缴纳押金", Toast.LENGTH_SHORT).show();
        }
    };

    private View.OnClickListener mDepositReturnClickListener = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "退还押金", Toast.LENGTH_SHORT).show();
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
    }
}
```

定义IDepositRepository封装数据获取逻辑（server，sqlite），此处模拟网络请求


```
public interface IDepositRepository {
    void getDepositInfo();
}
```

```
public interface OnDepositLoadListener {
    void onLoadDepositSuccess(MyDepositModel model);
}
```

```
public class DepositRepositoryImpl implements IDepositRepository {

    private OnDepositLoadListener mOnDepositLoadListener;

    public DepositRepositoryImpl(OnDepositLoadListener onDepositLoadListener) {
        mOnDepositLoadListener = onDepositLoadListener;
    }

    public void getDepositInfo() {
        new HttpTask() {
            @Override
            public void onRequestSuccess(MyDepositModel model) {
                mOnDepositLoadListener.onLoadDepositSuccess(model);
            }
        }.path("http://xxxx/getMydeposit").execute();
    }
}
```

MyDepositModel用于存储数据

```
public class MyDepositModel {
    public String moneyPaied;// 已经缴纳押金时，该字段表示已经缴纳的金额
    public String moneyNeed; // 未缴纳押金时，该字段表示需要缴纳的金额
    public String isDepositPay;// 是否缴纳押金，1：是，0：否
    public String isAuth; // 是否实名认证，1：是，0：否

    public static MyDepositModel mock() {
        MyDepositModel model = new MyDepositModel();
        model.moneyPaied = "200.00";
        model.moneyNeed = "300.00";
        model.isDepositPay = "0";
        model.isAuth = "0";
        return model;
    }

    public boolean isDepositPay() {
        return "1".equals(isDepositPay);
    }

    public boolean isAuth() {
        return "1".equals(isAuth);
    }
}
```

在MainActivity中调用IDepositRepository请求数据，通过OnDepositLoadListener获取请求成功后的数据，根据数据展示不同的view

```
public class MainActivity extends AppCompatActivity implements OnDepositLoadListener {

    @BindView(R.id.tv_title)
    TextView mTvTitle;
    @BindView(R.id.tv_money)
    TextView mTvMoney;
    @BindView(R.id.tv_tips)
    TextView mTvTips;
    @BindView(R.id.btn_pay_or_return)
    Button mBtnPayOrReturn;
    
    private IDepositRepository mIDepositRepositoryImpl;

    ///////////////////////////////////////////////////////////////////////////
    // 缴纳押金，退还押金的点击事件
    ///////////////////////////////////////////////////////////////////////////
    private View.OnClickListener mDepositPayClickListener = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "缴纳押金", Toast.LENGTH_SHORT).show();
        }
    };

    private View.OnClickListener mDepositReturnClickListener = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Toast.makeText(MainActivity.this, "退还押金", Toast.LENGTH_SHORT).show();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        mIDepositRepositoryImpl = new DepositRepositoryImpl(this);
        requestData();
    }

    ///////////////////////////////////////////////////////////////////////////
    // 模拟请求网络数据
    ///////////////////////////////////////////////////////////////////////////
    private void requestData() {
        mIDepositRepositoryImpl.getDepositInfo();
    }

    ///////////////////////////////////////////////////////////////////////////
    // 请求数据后的回调
    ///////////////////////////////////////////////////////////////////////////
    @Override
    public void onLoadDepositSuccess(MyDepositModel model) {
        showMydepositView(model);
    }

    ///////////////////////////////////////////////////////////////////////////
    // 根据数据的不同状态展示不同的view
    ///////////////////////////////////////////////////////////////////////////
    private void showMydepositView(MyDepositModel model) {
        if (model.isAuth()) {
            // 已经实名认证
            showAuthView();
        } else if (model.isDepositPay()) {
            // 已经缴纳押金
            showDepositPaiedView(model);
        } else {
            // 未缴纳押金
            showDepositNoPaiedView(model);
        }
    }

    ///////////////////////////////////////////////////////////////////////////
    // 展示未缴纳押金view
    ///////////////////////////////////////////////////////////////////////////
    private void showDepositNoPaiedView(MyDepositModel model) {
        // title
        mTvTitle.setText("您需要缴纳押金");

        // money
    mTvMoney.setTextColor(getResources().getColor(android.R.color.darker_gray));
        mTvMoney.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 30);
        mTvMoney.setText("¥ " +model.moneyNeed);

        // tips
        mTvTips.setText("押金随时可退");

        //button
        mBtnPayOrReturn.setText("缴纳押金");
        mBtnPayOrReturn.setOnClickListener(mDepositPayClickListener);
    }

    ///////////////////////////////////////////////////////////////////////////
    // 展示缴纳押金view
    ///////////////////////////////////////////////////////////////////////////
    private void showDepositPaiedView(MyDepositModel model) {
        // title
        mTvTitle.setText("您当前押金");

        // money
mTvMoney.setTextColor(getResources().getColor(android.R.color.holo_red_light));
        mTvMoney.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 40);
        mTvMoney.setText("¥ " + model.moneyPaied);

        // tips
        mTvTips.setVisibility(View.INVISIBLE);

        //button
        mBtnPayOrReturn.setText("退还押金");
        mBtnPayOrReturn.setOnClickListener(mDepositReturnClickListener);
    }

    ///////////////////////////////////////////////////////////////////////////
    // 展示已实名认证view
    ///////////////////////////////////////////////////////////////////////////
    private void showAuthView() {
        // title
        mTvTitle.setText("您已享受免押金服务");

        // money
        mTvMoney.setVisibility(View.INVISIBLE);

        // tips
        mTvTips.setText("您已完成实名认证");

        //button
        mBtnPayOrReturn.setVisibility(View.INVISIBLE);
    }
}
```
效果图
![免押金.png](http://upload-images.jianshu.io/upload_images/2157048-76841eba5136dbc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![未缴纳.png](http://upload-images.jianshu.io/upload_images/2157048-580d8bbed16d8384.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![已缴纳.png](http://upload-images.jianshu.io/upload_images/2157048-458787755bcf9257.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

项目结构：
model：MydepositModel作为数据的载体，Repository负责从网络获取数据，两者共同承担着model的职责；
view：activity_main.xml负责view的展示形式；
control：MainActivity负责接收view的交互请求，提交给model；当model发生变化时操作view，更新展示逻辑。

![mvc1.png](http://upload-images.jianshu.io/upload_images/2157048-c006a3de6c35fa7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![mvc2.png](http://upload-images.jianshu.io/upload_images/2157048-2429be5ab0682d99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Activity：view的容器，控制生命周期，页面交互与事件处理
xml：view展示与布局
view逻辑：操作view进行更新，如setText，setVisible等
业务逻辑：model更新，根据返回数据，执行逻辑主线，如：已/未缴纳/已认证
Repository：数据中心（server，sqlite）
model：存储数据
交互逻辑：用户操作view，产生事件与数据，反向传递给model进行处理，如setOnclick，或在EditText中输入内容提交server等

######总结：
xml作为view层，控制能力太弱，如果要去动态的改变一个Textview的字号，色值，或者隐藏/显示一个按钮，这些都没办法在xml中做，只能把代码写在Activity中。MyDepositModel以后，需要根据isAuth，isDepositPaied等业务逻辑，控制view的展示。造成了Activity既是view层，又是controller层，导致代码膨胀，当业务复杂度继续增加时，一个Activity上千行代码是很常见的，大量逻辑参与其中，维护及代码阅读难度将不断提升。
view和model直接交互，如：mTvMoney.setText(model.moneyNeed)，耦合较重，无法独立变化。mTvMoney作为一个Textview，只需要提供通过setText方法将String设置到TextView上进行展示的一种能力，至于这个String是从model1，还是model2中获取的，mTvMoney并不关心，而model作为数据源，也同样不需要关心当前是展示在mTvMoney上，还是mTvTips上。mBtnPayOrReturn按钮也是一样，只需提供一种点击响应的能力，至于点击后是操作缴纳押金，还是退还押金，mBtnPayOrReturn并不关心。


#####MVP的实现方式1
>在view和model之间新增presenter作为沟通的桥梁，presenter从model获取数据后，更新view的展示，使得view和model之间没有耦合，也将业务逻辑从view上抽离出来。


实现MainPresenter，持有IMainView，IDepositRepository成员变量，获取数据，根据业务逻辑更新view的展示。

```
public interface IMainPresenter {
    /**
     * @desc 进入页面后刷新数据
     */
    void requestData();

    /**
     * @desc 点击按钮
     */
    void onButtonClickAction();
}

```

```
/**
 * @author listen
 * @desc 主页面的presenter
 */
public class MainPresenter implements IMainPresenter, OnDepositLoadListener {

    private IMainView mIMainView;
    private IDepositRepository mIDepositRepositoryImpl;
    private MyDepositModel mModel;

    public MainPresenter(IMainView iMainView) {
        mIMainView = iMainView;
        mIDepositRepositoryImpl = new DepositRepositoryImpl(this);
    }

    @Override
    public void requestData() {
        mIDepositRepositoryImpl.getDepositInfo();
    }

    /**
     * @desc 实现OnDepositLoadListener回调，获取MyDepositModel，并根据业务逻辑更新view的展示
     */
    @Override
    public void onLoadDepositSuccess(MyDepositModel model) {
        if (model.isAuth()) {
            // 已经实名认证
            showAuthView();
        } else if (model.isDepositPay()) {
            // 已经缴纳押金
            showDepositPaiedView(model);
        } else {
            // 未缴纳押金
            showDepositNoPaiedView(model);
        }
    }

    /**
     * @desc 未支付状态view
     */
    private void showDepositNoPaiedView(MyDepositModel model) {
        // title
        mIMainView.setTitleText("您需要缴纳押金"); // 不暴露mTvTitle，只提供设置title文案的能力

        // money
        mIMainView.setMoneyTextVisible();// 提供操作MoneyText显示/隐藏的能力
        mIMainView.setMoneyTextColorGray();// 提供MoneyText字体设置为灰色的能力
        mIMainView.setMoneyTextSizeSmall();// 提供MoneyText字号设置小的能力
        mIMainView.setMoneyText("¥ " + model.moneyNeed);// 提供设置MoneyText文案的能力

        // tips
        mIMainView.setTipsTextVisible();// 提供操作TipsText显示/隐藏的能力
        mIMainView.setTipsText("押金随时可退");// 提供设置TipsText文案的能力

        //button
        mIMainView.setButtonVisible();// 提供操作Button显示/隐藏的能力
        mIMainView.setButtonText("缴纳押金");// 提供设置Button文案的能力
    }

    /**
     * @desc 支付状态view
     */
    private void showDepositPaiedView(MyDepositModel model) {
        // title
        mIMainView.setTitleText("您当前押金");

        // money
        mIMainView.setMoneyTextVisible();
        mIMainView.setMoneyTextColorRed();
        mIMainView.setMoneyTextSizeBig();
        mIMainView.setMoneyText("¥ " + model.moneyNeed);

        // tips
        mIMainView.setTipsTextInvisible();

        //button
        mIMainView.setButtonVisible();
        mIMainView.setButtonText("退还押金");
    }

    /**
     * @desc 实名认证状态view
     */
    private void showAuthView() {
        // title
        mIMainView.setTitleText("您已享受免押金服务");

        // money
        mIMainView.setMoneyTextInvisible();

        // tips
        mIMainView.setTipsTextVisible();
        mIMainView.setTipsText("您已完成实名认证");

        //button
        mIMainView.setButtonInvisible();
    }

    /**
     * @desc 当点击Button时触发的操作
     */
    @Override
    public void onButtonClickAction() {
        if (mModel.isDepositPay()) {
            mIMainView.showToast("退还押金");
        } else {
            mIMainView.showToast("缴纳押金");
        }
    }
}
```

实现view层

```
/**
 * @author listen
 * @desc 主页面view层接口
 */
public interface IMainView {
    void setTitleText(String text);
    
    void setMoneyTextColorGray();
    void setMoneyTextSizeSmall();
    void setMoneyText(String text);
    void setMoneyTextInvisible();
    void setMoneyTextVisible();
    void setMoneyTextColorRed();
    void setMoneyTextSizeBig();
    
    void setTipsText(String text);
    void setTipsTextInvisible();
    void setTipsTextVisible();
    
    void setButtonText(String text);
    void setButtonInvisible();
    void setButtonVisible();
    
    void showToast(String text);
}
```
```
public class MainActivity extends AppCompatActivity implements IMainView {
    @BindView(R.id.tv_title)
    TextView mTvTitle;
    @BindView(R.id.tv_money)
    TextView mTvMoney;
    @BindView(R.id.tv_tips)
    TextView mTvTips;
    @BindView(R.id.btn_pay_or_return)
    Button mBtnPayOrReturn;

    private IMainPresenter mPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        mPresenter = new MainPresenter(this);
        mPresenter.requestData();

        mBtnPayOrReturn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.onButtonClickAction();
            }
        });
    }

    @Override
    public void setTitleText(String text) {
        mTvTitle.setText(text);
    }

    @Override
    public void setMoneyTextColorGray() {
        mTvMoney.setTextColor(getResources().getColor(android.R.color.darker_gray));
    }

    @Override
    public void setMoneyTextColorRed() {
        mTvMoney.setTextColor(getResources().getColor(android.R.color.holo_red_light));
    }

    @Override
    public void setMoneyTextSizeSmall() {
        mTvMoney.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 30);
    }

    @Override
    public void setMoneyTextSizeBig() {
        mTvMoney.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 40);
    }

    @Override
    public void setMoneyText(String text) {
        mTvMoney.setText(text);
    }

    @Override
    public void setMoneyTextInvisible() {
        mTvMoney.setVisibility(View.VISIBLE);
    }

    @Override
    public void setMoneyTextVisible() {
        mTvMoney.setVisibility(View.VISIBLE);
    }

    @Override
    public void setTipsText(String text) {
        mTvTips.setText(text);
    }

    @Override
    public void setTipsTextInvisible() {
        mTvTips.setVisibility(View.INVISIBLE);
    }

    @Override
    public void setTipsTextVisible() {
        mTvTips.setVisibility(View.VISIBLE);
    }

    @Override
    public void setButtonText(String text) {
        mBtnPayOrReturn.setText(text);
    }

    @Override
    public void setButtonInvisible() {
        mBtnPayOrReturn.setVisibility(View.INVISIBLE);
    }

    @Override
    public void setButtonVisible() {
        mBtnPayOrReturn.setVisibility(View.VISIBLE);
    }

    @Override
    public void showToast(String text) {
        Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
    }
}
```
![mvp项目结构.png](http://upload-images.jianshu.io/upload_images/2157048-fa7b9941b45e35ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![mvp2.png](http://upload-images.jianshu.io/upload_images/2157048-208c7539307eef2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######总结
presenter处理业务逻辑并更新view，Activity只提供基础的操作view的能力，2者互相独立，view与业务分离。
业务变化1：不管已缴纳，未缴纳，免押金任何状态tipsText都不显示，此时就不需要去修改Activity，直接在presenter设置mIMainView.setTipsTextVisible()即可；
业务变化2：新增一种状态，已实名认证，不过某些条件不满足，押金不能全免，只能减半，titleText显示"已认证，还需缴纳押金"，moneyText大号字体，红色，tipsText显示"押金已减半"，Button显示文案"补足押金"，这种场景下，就不用去修改Activity的任何代码，只要在presenter新增逻辑分支，根据view提供的能力进行更新即可。

```
/**
 * @desc 押金减半状态view
 */
private void showDepositHalfPayView(MyDepositModel model) {
   // title
   mIMainView.setTitleText("已认证，还需缴纳押金");

   // money
   mIMainView.setMoneyTextVisible();
   mIMainView.setMoneyTextColorRed();
   mIMainView.setMoneyTextSizeBig();
   mIMainView.setMoneyText("¥ " + model.moneyNeed);

   // tips
   mIMainView.setTipsTextVisible();
   mIMainView.setTipsText("押金已减半");

   //button
   mIMainView.setButtonVisible();
   mIMainView.setButtonText("补足押金");
}
```

 
```
/**
 * @desc 押金减半时，button的点击响应
 */
public void onButtonClickAction() {
        if ("押金减半") {
            mIMainView.showToast("补足押金");
        } 
    }
```

业务变化3：presenter依赖的是IMainView，不管是MainActivity，还是Main1Activity，只要是实现了IMainView即可复用当前presenter。

当我们把业务逻辑抽取到presenter后，Activity基本上只剩下一些view的逻辑，真正实现了减负，变成了一个相对纯净的view。当我们需要修改view的逻辑时，就去找Activity，需要修改数据逻辑时，就去找Repository，修改业务逻辑时就去找presenter，每个模块职责分明。
缺点：
1.view与presenter之间交互过于频繁，Activity中都是一些setText，setVisibility等方法。这时很容易让人想到使用Databinding可以很好的简化这部分代码。


#####MVP的实现方式2
>通过DataBinding实现model到view的单向绑定，减少view与model之间因频繁交互而产生的冗余代码。

在<data>标签中引入data=MyDepositModel，presenter=IMainPresenter。当model变化时，通过data将数据映射到view上。当Button产生点击事件时交由presenter响应并处理。使用Databinding以后，开发流程上省略了findView，setView的过程，在写xml的时候就可以直接将model进行关联及映射。

```
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable name="data" type="com.listen.test_mvvm.model.data.MyDepositModel"/>
        <variable name="presenter" type="com.listen.test_mvvm.presenter.IMainPresenter"/>
        <import type="android.view.View"/>
    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:paddingLeft="30dp"
        android:paddingRight="30dp"
        tools:context="com.listen.test_mvvm.view.MainActivity">

        <TextView
            android:id="@+id/tv_title"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="50dp"
            android:textColor="@android:color/black"
            android:textSize="20sp"
            android:text="@{data.title}"/>

        <TextView
            android:id="@+id/tv_money"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginBottom="50dp"
            android:layout_marginTop="50dp"
            android:textColor="@{data.isDepositPay ? @android:color/holo_red_light : @android:color/darker_gray}"
            android:textSize="@{data.isDepositPay ? @dimen/sp_40 : @dimen/sp_30}"
            android:visibility="@{data.isAuth ? View.INVISIBLE : View.VISIBLE}"
            android:text="@{data.money}"/>

        <TextView
            android:id="@+id/tv_tips"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@android:color/black"
            android:textSize="16sp"
            android:visibility="@{data.showTips ? View.VISIBLE : View.INVISIBLE}"
            android:text="@{data.tips}"/>

        <Button
            android:id="@+id/btn_pay_or_return"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginLeft="30dp"
            android:layout_marginRight="30dp"
            android:layout_marginTop="100dp"
            android:onClick="@{presenter.onButtonClickAction}"
            android:visibility="@{data.isAuth ? View.INVISIBLE : View.VISIBLE}"
            android:textColor="@android:color/black"
            android:textSize="16sp"
            android:text='@{data.isDepositPay ? "退还押金" : "缴纳押金"}'/>
    </LinearLayout>
</layout>
```

将业务逻辑转移到MyDepositModel

```
public class MyDepositModel {
    public String moneyPaied;// 已经缴纳押金时，该字段表示已经缴纳的金额
    public String moneyNeed; // 未缴纳押金时，该字段表示需要缴纳的金额
    public String isDepositPay;// 是否缴纳押金，1：是，0：否
    public String isAuth; // 是否实名认证，1：是，0：否

    public boolean isDepositPay() {
        return "1".equals(isDepositPay);
    }

    public boolean isAuth() {
        return "1".equals(isAuth);
    }

    public String getTitle() {
        if (isAuth()) {
            return "您已享受免押金服务";
        }

        if (isDepositPay()) {
            return "您当前押金";
        } else {
            return "您需要缴纳押金";
        }
    }

    public String getMoney() {
        if (isDepositPay()) {
            return "¥ " + moneyPaied;
        } else {
            return "¥ " + moneyNeed;
        }
    }

    public String getTips() {
        if (isAuth()) {
            return "您已完成实名认证";
        }

        if (!isDepositPay()) {
            return "押金随时可退";
        }

        return "";
    }

    public boolean isShowTips() {
        if (isAuth() || !isDepositPay()) {
            return true;
        }
        return false;
    }
}
```

MainPresenter不再与view频繁的交互，仅仅是作为view和model的连接器，主干逻辑更为清晰

```
public class MainPresenter implements IMainPresenter, OnDepositLoadListener {
    private IMainView mIMainView;
    private IDepositRepository mIDepositRepository;
    private MyDepositModel mModel;

    public MainPresenter(IMainView iMainView) {
        mIMainView = iMainView;
        mIDepositRepository = new DepositRepository(this);
    }

    // 请求数据
    @Override
    public void requestData() {
        mIDepositRepository.getDepositInfo();
    }

    // 获取数据，通知view更新
    @Override
    public void onLoadDepositSuccess(MyDepositModel model) {
        mModel = model;
        mIMainView.updateData(model);
    }

    // 接收并处理view的点击事件
    @Override
    public void onButtonClickAction(View v) {
        if (mModel.isDepositPay()) {
            mIMainView.showToast("退还押金");
        } else {
            mIMainView.showToast("缴纳押金");
        }
    }
}
```

MainActivity中不再需要fingViewById，也不用定义Textview，Button的成员变量，全部交由DataBinding进行处理，相较MVP的实现，MainActivity进一步简化

```
public class MainActivity extends AppCompatActivity implements IMainView {
    private IMainPresenter mPresenter;
    private ActivityMainBinding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        mPresenter = new MainPresenter(this);
        mBinding.setPresenter(mPresenter);
        
        // 初始化页面数据
        mPresenter.requestData();
    }

    // 更新数据绑定
    @Override
    public void updateData(MyDepositModel model) {
        mBinding.setData(model);
    }

    // 提供Toast提示的能力
    @Override
    public void showToast(String text) {
        Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
    }
}
```

IMainView接口也不再需要提供那么多操作view的方法

```
public interface IMainView {
    void updateData(MyDepositModel model);
    void showToast(String text);
}
```


问题：
1.xml中参杂了一些业务逻辑，如：data.isDepositPay，data.isAuth，xml中应该尽量只是简单的view逻辑，与业务逻辑隔离。
2.由于使用databinding是model->view的单向绑定，不得不将大部分逻辑搬移到model中，例如：MyDepositModel中即有数据处理逻辑，isDepositPay，isAuth（如果model中存在list<Item>等，经常会对外提供getItemById(int id)等方法，做遍历查询）。同时还存在view的展示逻辑，例：isShowTips，getTitle，getMoney，这些方法都是根据数据变化控制view的展示，两者之间其实还是有比较明确的分界线，可以进一步分离，解耦，避免model过重。

#####MVPVM的实现方式
>通过viewModel作为model和view的适配层，model只负责数据存储

activity_main.xml中将原先的model.isDepositPay()，model.isAuth()改成viewModel.moneyTextVisible()，viewModel.moneyTextSizeLarge()等。在xml中依赖viewModel，只关心view显示/隐藏，字号变大/小，色值高亮/正常，至于什么情况下展示高亮，是否显示由viewModel中适配的model逻辑决定。

```
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable name="data" type="com.listen.test_mvvm.model.viewmodel.IMyDepositViewModel"/>
        <variable name="presenter" type="com.listen.test_mvvm.presenter.IMainPresenter"/>
        <import type="android.view.View"/>
    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:paddingLeft="30dp"
        android:paddingRight="30dp"
        tools:context="com.listen.test_mvvm.view.MainActivity">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="50dp"
            android:text="@{data.title}"
            android:textColor="@android:color/black"
            android:textSize="20sp"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginBottom="50dp"
            android:layout_marginTop="50dp"
            android:text="@{data.money}"
            android:textColor="@{data.moneyTextColorHightLight ? @android:color/holo_red_light : @android:color/darker_gray}"
            android:textSize="@{data.moneyTextSizeLarge ? @dimen/sp_40 : @dimen/sp_30}"
            android:visibility="@{data.moneyTextVisible ? View.VISIBLE : View.INVISIBLE}"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{data.tips}"
            android:textColor="@android:color/black"
            android:textSize="16sp"
            android:visibility="@{data.tipsVisible ? View.VISIBLE : View.INVISIBLE}"/>

        <Button
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginLeft="30dp"
            android:layout_marginRight="30dp"
            android:layout_marginTop="100dp"
            android:onClick="@{presenter.onButtonClickAction}"
            android:text="@{data.buttonText}"
            android:textColor="@android:color/black"
            android:textSize="16sp"
            android:visibility="@{data.buttonVisible ? View.VISIBLE : View.INVISIBLE}"/>
    </LinearLayout>
</layout>

```

IMyDepositViewModel接口，定义view提供的能力

```
public interface IMyDepositViewModel {
    String getTitle();

    boolean isMoneyTextColorHightLight();
    boolean isMoneyTextSizeLarge();
    boolean isMoneyTextVisible();
    String getMoney();

    boolean isTipsVisible();
    String getTips();

    boolean isButtonVisible();
    String getButtonText();
}
```

MyDepositBaseViewModel实现IMyDepositViewModel的默认展示逻辑

```
public abstract class MyDepositBaseViewModel implements IMyDepositViewModel {

    private MyDepositModel mModel;

    public MyDepositBaseViewModel(MyDepositModel model) {
        mModel = model;
    }

    public MyDepositModel getModel() {
        return mModel;
    }

    @Override
    public String getTitle() {
        return "";
    }

    @Override
    public boolean isMoneyTextColorHightLight() {
        return false;
    }

    @Override
    public boolean isMoneyTextSizeLarge() {
        return false;
    }

    @Override
    public boolean isMoneyTextVisible() {
        return false;
    }

    @Override
    public String getMoney() {
        return "";
    }

    @Override
    public boolean isTipsVisible() {
        return false;
    }

    @Override
    public String getTips() {
        return "";
    }

    @Override
    public boolean isButtonVisible() {
        return false;
    }

    @Override
    public String getButtonText() {
        return "";
    }
}
```

已缴纳押金时viewModel的展示逻辑

```
public class MyDepositPayViewModel extends MyDepositBaseViewModel {
    public MyDepositPayViewModel(MyDepositModel model) {
        super(model);
    }

    @Override
    public String getTitle() {
        return "您当前押金";
    }

    @Override
    public String getMoney() {
        return "¥ " + getModel().moneyPaied;
    }

    @Override
    public boolean isMoneyTextVisible() {
        return true;
    }

    @Override
    public boolean isMoneyTextColorHightLight() {
        return true;
    }

    @Override
    public boolean isMoneyTextSizeLarge() {
        return true;
    }

    @Override
    public boolean isButtonVisible() {
        return true;
    }

    @Override
    public String getButtonText() {
        return "退还押金";
    }
}

```

未缴纳押金时viewModel的展示逻辑

```
public class MyDepositNoPayViewModel extends MyDepositBaseViewModel {

    public MyDepositNoPayViewModel(MyDepositModel model) {
        super(model);
    }

    @Override
    public String getTitle() {
        return "您需要缴纳押金";
    }

    @Override
    public String getMoney() {
        return "¥ " + getModel().moneyNeed;
    }

    @Override
    public boolean isMoneyTextVisible() {
        return true;
    }

    @Override
    public boolean isMoneyTextColorHightLight() {
        return false;
    }

    @Override
    public boolean isMoneyTextSizeLarge() {
        return false;
    }

    @Override
    public boolean isButtonVisible() {
        return true;
    }

    @Override
    public boolean isTipsVisible() {
        return true;
    }

    @Override
    public String getTips() {
        return "押金随时可退";
    }

    @Override
    public String getButtonText() {
        return "缴纳押金";
    }
}
```

已认证时viewModel的展示逻辑

```
public class MyDepositAuthViewModel extends MyDepositBaseViewModel {

    public MyDepositAuthViewModel(MyDepositModel model) {
        super(model);
    }

    @Override
    public String getTitle() {
        return "您已享受免押金服务";
    }

    @Override
    public boolean isTipsVisible() {
        return true;
    }

    @Override
    public String getTips() {
        return "您已完成实名认证";
    }
}
```

MainPresenter获取数据后，根据不同业务逻辑展示
MyDepositAuthViewModel，MyDepositPayViewModel，MyDepositNoPayViewModel。此处有点像设计模式中的策略模式，这3个viewModel就是view的不同展示策略的封装。

```
public class MainPresenter implements IMainPresenter, OnDepositLoadListener {

    private IMainView mIMainView;
    private IDepositRepository mIDepositRepositoryImpl;
    private MyDepositModel mModel;

    public MainPresenter(IMainView iMainView) {
        mIMainView = iMainView;
        mIDepositRepositoryImpl = new DepositRepositoryImpl(this);
    }

    @Override
    public void requestData() {
        mIDepositRepositoryImpl.getDepositInfo();
    }

    @Override
    public void onLoadDepositSuccess(MyDepositModel model) {
        mModel = model;
        if (mModel.isAuth()) {
            mIMainView.updateData(new MyDepositAuthViewModel(model));
        } else if (mModel.isDepositPay()) {
            mIMainView.updateData(new MyDepositPayViewModel(model));
        } else {
            mIMainView.updateData(new MyDepositNoPayViewModel(model));
        }
    }

    @Override
    public void onButtonClickAction(View v) {
        if (mModel.isDepositPay()) {
            mIMainView.showToast("退还押金");
        } else {
            mIMainView.showToast("缴纳押金");
        }
    }
}
```


MainActivity.java，只做基本的数据请求，DataBinding初始化，toast提示等操纵。

```
public class MainActivity extends AppCompatActivity implements IMainView {
    private IMainPresenter mPresenter;
    private ActivityMainBinding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        mPresenter = new MainPresenter(this);
        mBinding.setPresenter(mPresenter);
        mPresenter.requestData();
    }

    @Override
    public void updateData(IMyDepositViewModel viewModel) {
        mBinding.setData(viewModel);
    }

    @Override
    public void showToast(String text) {
        Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
    }
}
```


![mvpvm.png](http://upload-images.jianshu.io/upload_images/2157048-9f76f2696e03adf4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![mvpvm2.png](http://upload-images.jianshu.io/upload_images/2157048-119d62e1d7e42238.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图：用户操作view，触发事件响应，通过presenter中转，传递给model进行数据处理，获取新数据后处理业务逻辑，并适配成不同状态的viewModel展示策略，view根据不同的viewModel进行更新。

总结：
从mvc到mvpvm，项目中类虽然变多了，不过模块之间职责更加明确清晰。大部分情况，使用mvp结合databinding就可以较好的对view和model进行解耦，且代码冗余较少，当然在页面逻辑简单的情况下，可能连Presenter都没有用上的必要。不过如果是类似本文中的需求，view状态相对复杂的情况下，最好还是经过一层viewModel适配，也可以释放model的压力，xml布局中只依赖抽象的IMyDepositViewModel（model->view的数据输入）和IMainPresenter（view->model的事件输出），不依赖具体。
本文并非按照传统的MVC，MVP，MVVM的路线实现架构，而是采用循序渐进的方式，在MVC中发现Activity过重，所以引入MVP，Presenter作为View和Model的中转，达到解耦的目的。后来发现Activity提供view能力时冗余代码过多，所以引入DataBinding，虽然代码简化了，不过xml中引入了部分业务逻辑，model中同时参杂数据处理逻辑和view展示逻辑，故而引入viewModel，将xml与model进一步解耦，同时减轻model负担，不过此时并不算是mvvm，本质上在mvp的基础上，引入vm，因此presenter的中转作用还在，所以才演变成了现在的mvpvm。同时强调下，架构无绝对的好坏与绝对的标准，大家应该在项目中根据实际场景选择最合适的架构方式。本文中如有说明，解释不到位的地方，还请指出，互相学习共勉。

最终版本项目地址：https://github.com/listen2code/Test_MVPVM

