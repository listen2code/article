![运行时权限.png](https://github.com/listen2code/article/blob/master/运行时权限/screenshot/运行时权限.jpg?raw=true)

####为什么需要6.0运行时权限
* 更友好

> 6.0以前的安装时权限，会在应用安装时列出所有需要的权限，当列出一些危险权限时，用户不知每个权限的具体用途，可能因为这些权限警告而放弃安装应用。对于一些非装不可的应用，用户则不得不被迫接受所有权限，很容易安装了一些流氓APP，体验不佳。
> 6.0以后的运行时权限，可以在调用相关功能之前判断权限授权状态，并自定义提示弹框告知用户权限用途，使用户清楚了解之后，再授权使用。

![5.0和6.0的安装页面.png](https://github.com/listen2code/article/blob/master/运行时权限/screenshot/5.0和6.0的安装页面.png?raw=true)

* 更稳定

> 6.0系统的手机对于每个应用，都有个权限设置页面，可以手动开关权限，如果用户在设置页面误关了某个权限，若没在程序运行时做判断，则会导致相关功能的调用失败，引起崩溃等。

![5.0和6.0的权限设置页面.png](https://github.com/listen2code/article/blob/master/运行时权限/screenshot/5.0和6.0的权限设置页面.png?raw=true)

####如何实现运行时权限

* 设置targetSdkVersion

> 只有targetSdkVersion>=23，且安装在6.0以上的手机时，运行时权限机制才能正常运作

| 手机系统 | targetSdkVersion<23 | targetSdkVersion>=23 |
| --- | --- | --- |
| 5.0+ | 安装时权限 | 安装时权限 |
| 6.0+ | 安装时权限 | **运行时权限** |

* 代码实现

> 在需要使用某项权限时，通过V4包的checkSelfPermission判断权限是否授权，通过requestPermissions申请某项权限

```
// 需要执行某项需要权限的操作时
if (ContextCompat.checkSelfPermission(this, "某项权限") == PackageManager.PERMISSION_GRANTED) {
   // 执行操作
} else {
   // 请求权限
   ActivityCompat.requestPermissions(this, new String[]{"某项权限"}, requestCode);
}
```
 
> 在Activity的onRequestPermissionsResult回调方法中，处理权限授权结果

```
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
   if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
       // 授权成功,执行操作
   } else {
       // 权限被拒绝
       if (ActivityCompat.shouldShowRequestPermissionRationale(this, "权限名称")) {
           // shouldShowRequestPermissionRationale=true: 表示权限被拒绝,且没有勾选"never ask again"
           // 正常的权限被拒绝流程,可以继续申请权限,重复以上流程
       } else {
           // shouldShowRequestPermissionRationale=false: 在权限被拒绝,且勾选"never ask again"的情况下，返回false
           // 继续申请权限的时候,不会再弹出默认的系统弹框,需要自定义提示弹框,并引导用户去权限设置页面,手动开启权限
       }
   }
}
```

####LsPermission工具类的使用

* 主要实现逻辑参考[PermissionGen](https://github.com/lovedise/PermissionGen)，封装了权限判断，请求，结果处理等通用逻辑，目前只支持context=AppCompatActivity，如果在Fragment中使用时可以调用getActivity()获取上层AppCompatActivity。

![demo演示.gif](https://github.com/listen2code/article/blob/master/运行时权限/screenshot/运行时权限.gif?raw=true)

```
private void normal() {
   final String[] permissions =
       new String[] {Manifest.permission.CALL_PHONE, Manifest.permission.ACCESS_FINE_LOCATION};

   PermissionUtil.request(this, permissions, new OnPermissionAdapter() {
       /**
        * @desc 申请的权限全被授权
        */
       @Override
       public void onGrant() {
           showToast("权限被同意");
           callPhone();
       }

       /**
        * @desc 权限被拒绝
        */
       @Override
       public void onDeny(List<String> permissions) {
           showToast("用户拒绝授权" + permissions.toString());
       }

       /**
        * @desc 权限被拒绝,且勾选"never ask again"
        */
       @Override
       public void onNeverAsk(List<String> permissions) {
           showToast("用户拒绝授权, 并勾选 never ask again " + permissions.toString());
           PermissionUtil.showNeverAskDialog(MainActivity.this, "这个权限很重要");
       }
       
       /**
        * @desc 无论权限授权成功还是失败，都会回调
        */
       @Override
       public void always(List<String> grantPermissions, List<String> denyPermissions,
           List<String> foreverDenyPermissions) {
           showToast("授权: " + grantPermissions.toString() + "\n 拒绝: " + denyPermissions.toString()
               + "\n never ask: " + foreverDenyPermissions.toString());
       }
   });
}

/**
* @desc 将权限回调转发给PermissionUtil处理
* @author listen
* @date 2017/2/24 13:47
*/
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
   PermissionUtil.onRequestPermissionsResult(this, requestCode, permissions, grantResults);
   super.onRequestPermissionsResult(requestCode, permissions, grantResults);
}
```

* PermissionUtil内部通过SparseArray保存了requestCode和权限回调的键值对，避免了连续点击时重复回调情况，对于不同的权限申请，推荐使用不同的requestCode，避免回调覆盖的问题。

```
private static SparseArray<OnPermissionListener> mPermissionRequestList = new SparseArray<>();

public static void request(final Context context, int requestCode, String[] permissions,
                          OnPermissionListener listener) {
   /** 存在未授权的权限 */
   synchronized (mPermissionRequestList) {
       if (null != mPermissionRequestList.get(requestCode)) {
           /** 当前权限请求已经存在,不重复添加 */
           log("the same permission is requesting");
       } else {
           /** 将当前权限请求加入队列 */
           /** 如果一个页面中存在分别触发A, B多个权限的情况, 则最好将不同权限申请对应不同的requestCode, 存入SparseArray分别处理 */
           mPermissionRequestList.put(requestCode, listener);

           /** 执行权限申请 */
           ActivityCompat.requestPermissions();
       }
   }
}
```

在权限回调的时候从SparseArray移除Listener

```
public static void onRequestPermissionsResult(Context context, int requestCode, String[] permissions,
                                                  int[] grantResults) {
   final OnPermissionListener listener;
   synchronized (mPermissionRequestList) {
       listener = mPermissionRequestList.get(requestCode);
       mPermissionRequestList.remove(requestCode);
   }

   if (null != listener) {
       /** 执行回调 */
       listener.onGrant();
       listener.onDeny();
       listener.onNeverAsk();
       listener.always();
   } else {
       log("request is not exists");
   }
}
```

代码地址：[LsPermission](https://github.com/listen2code/Test_Mogu_View)

除了基本的权限申请逻辑的封装以外，还写了类似微信，支付宝，百度地图等在启动页的权限申请Demo，算是PermissionUtil的简单运用。


参考：
[Android 6.0 运行时权限处理完全解析](http://blog.csdn.net/lmj623565791/article/details/50709663)
[Android 6.0 运行时权限管理最佳实践](https://gold.xitu.io/post/57d5de3e2e958a00546a7465)
[聊一聊Android 6.0的运行时权限](http://droidyue.com/blog/2016/01/17/understanding-marshmallow-runtime-permission/index.html)

官方：
[在运行时请求权限](https://developer.android.com/training/permissions/requesting.html?hl=zh-cn)
[权限最佳做法](https://developer.android.com/training/permissions/best-practices.html?hl=zh-cn)

