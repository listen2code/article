![客户端接口设计规范.png](https://github.com/listen2code/article/blob/master/从客户端的角度设计后端的接口/screenshot/客户端接口设计规范.png?raw=true)

####前言
> 兵马未动，粮草先行。在一款APP产品的各个版本迭代中，兵马的启动指的是真正开始敲代码的时候，粮草先行则是指前期的需求，交互，UI等评审准备阶段，还有本文要说的接口的设计与评审。虽然很多时候一个api接口的业务，数据逻辑是后端提供的，但真正使用这个接口的是客户端，一个前端功能的实现流程与逻辑，有时候只有客户端的RD才清楚，从某种意义来说，客户端算是接口的需求方。所以建议在前期接口设计和评审时，客户端的RD应该更多的思考和参与，什么时机调什么接口？每个接口需要哪些字段？数据含义怎么给？只有这些都考虑清楚，且达成一致并产出接口文档后，当项目真正启动时，根据接口协议进行开发，才能尽量避免各种不确定因素对项目整体进度的影响。本文介绍了接口设计中常见的规范，以及个人的一些思考与总结，水平有限，权当是抛砖引玉，如果有更好的设计，请在文章下方留言告诉我，谢谢。

####接口设计规范
一.  接口示例
>以下是一个用户信息接口的文档示例，包含接口描述，请求参数，响应参数，json示例等。

接口描述：用户登陆成功后，或进入个人中心时会获取一次用户信息

| URI  | 方法| 
| --- | ---| 
| /userinfo | GET| 
    
请求参数

| 名称 | 必填 | 备注 |
| --- | --- | --- |
| id | 是 | 用户id |
    
响应参数
    
| 名称 | 类型 | 备注 |
| --- | --- | --- |
| id | String | 用户id |
| name | String | 姓名，例：张三 |
| age | String | 年龄，例：20 |
    
json示例
```
{
    "code":200,
    "msg":"成功",
    "time":"1482213602000",
    "data": {
        "id":"1001",
        "name":"张三",
        "age":"20"
    }
}
```


二.  基本规范 

1.通用请求参数

>每个请求都要携带的参数，用于描述每个请求的基本信息，后端可以通过这些字段进行接口统计，或APP终端设备的统计，一般放到header或url参数中。
    
| 字段名称 | 说明 |
| --- | --- | --- |
| version | 客户端版本version，例：1.0.0 |
| token | 登陆成功后，server返回的登陆令牌token |
| os | 手机系统版本（Build.VERSION.RELEAS）例：4.4，4.5 |
| from | 请求来源，例：android/ios/h5 |
| screen | 手机尺寸，例：1080*1920 |
| model | 机型信息（Build.MODEL），例：Redmi Note 3 |
| channel | 渠道信息，例：com.wandoujia |
| net | APP当前网络状态，例：wifi，mobile；部分接口可以根据用户当前的网络状态，下发不同数据策略，如：wifi则返回高清图，mobile情况则返回缩略图 |
| appid | APP唯一标识，有的公司一套server服务多款APP时，需要区分开每个APP来源 |
    
2.请求Path，http://www.online.com/api/ [path]

> 原则：在以下命名规范的基础上尽量保持良好的可读性，见名知意。另外这里需要额外提下restful规范，个人理解restful规范是通过path表示当前请求的资源，通过method表示当前请求的操作动作（post=增，delete=删，put=改，get=查），例：GET  /userinfo/{id}，通过这个path就可以清楚的知道当前请求的意图是根据id获取用户信息，而APP开发中很多时候一个页面是需要同时获取，如，用户，订单，营销各种信息，这时候就很难用一个path来表示当前请求的真正意图，restful规范就很难得到实现，有不同见解的欢迎交流。故本文介绍的接口设计方法，只区分get和post，通过path命名定义请求行为，
        
| 操作行为 | Method | Path |
| --- | --- | --- |
| 查找 | GET | getXxx |
| 增加 | POST | addXxx/submitXxx |
| 修改 | POST | modifyXxx |
| 删除 | POST | delXxx |

示例
    
| 操作行为 | Method | Path |
| --- | --- | --- |
| 获取用户信息 | GET | getUserInfo |
| 增加收货地址 | POST | addAddress |
| 修改密码 | POST | modifyPwd |
| 删除收货地址 | POST | delAddress |
| 登陆 | GET | login |
| 发送短信验证码 | GET | sendSms |
| 订单支付 | POST | orderPay |

3.响应数据
    
| 字段名称 | 说明 |
| --- | --- | --- |
| code | 响应状态码，200：成功；非200：失败 |
| msg | 请求失败时的message|
| time | 服务端时间戳，单位：毫秒。用于同步时间 |
| data | 数据实体 |

code=200时，msg=登陆成功/修改成功/提交成功；如果需要Toast，可以直接使用msg。
code!=200时，msg=错误提示信息；比如login接口，"账号或密码错误"，"账号不存在"类似这些的业务提示文案放在msg字段，客户端直接Toast就可以了。不过需要提醒后端同学，错误提示不能自己觉的什么合适就提示什么，要按需求文档来提供，或和PM确认。

>object类型数据

```
// json
{
  "code":200,
  "msg":"成功",
  "time":"1482213602000",
  "data": {
    "name":"张三",
    "age":"20"
  }
}  
     
// model.java
public class Model {
     public String name;
     public String age;
} 
```
    
>array类型数据，正常情况下在解析json的时候，1.先解析code和msg，判断code==200的情况下继续解析data。2.将data下面的json串解析成当次请求需要的model数据结构。对于array类型的数据，即使只有1个list字段，也要保证data下是个完整的object结构，这样我们在用Gson解析model的时候，统一将data层级下的数据当object解析就可以了，不用区分object或array的情况。
    
```
// json
{
    "code":200,
    "msg":"成功",
    "time":"1482213602000",
    "data": {
        "list":["张三","李四"]
    }
}   
  
// model.java
public class Model {
    public List<String> list;
} 
```

>array+分页类型数据，需要额外返回total字段，客户端需要通过total判断本地加载的list是否还有更多可以加载。
            
请求参数

| 名称 | 必填 | 备注 |
            | --- | --- | --- |
            | pageNum | 是 | 当前第几页，例：1，2，3 |
            | pageSize | 是 | 每页条数，例：10 |
          
响应数据
            
```
// json
{
    "code":200,
    "msg":"成功",
    "time":"1482213602000",
    "data": {
        "list":["张三","李四"],
        "total":"10"
    }
}   
  
// model.java
public class Model {
    public List<String> list;
    public String total;
} 
```

>不论列表页面是支持分页加载，还是一次加载全部数据，都建议将接口设计成支持分页的，如果要实现一次性加载只要把pageSize改成类似Integer.Max的值。这样设计的好处是客户端和后端可以设计一套统一的分页列表模版代码，即使需求变更，也可以很好的支持。
            
4.命名规范
     
* **统一命名：**与后端约定好即可（php和js在命名时一般采用下划线风格，而Java中一般采用的是驼峰法），无绝对标准，不要同时存在驼峰"userName"，下划线"phone_number"两种形式就可以了。
        
* **避免冗余字段：**每次在新增接口字段时，注意是否已经存在同一个含义的字段，保持命名一致，不要同时存在"userName"，"username"，"uName"多种同义字段。

* **注释清晰（重要）：**每个接口/字段都需要有详细的描述信息，很多时候接口体现业务逻辑，是团队中很重要的文档沉淀，同时，详细的接口文档，可以帮助新人快速熟悉业务。具体示例如下：
       
> 接口描述：用户登陆成功后会获取一次用户信息，每次进入个人中心也会重新获取一遍

| URI | 方法 |
               | --- | --- | --- |
               | /userinfo | GET |
            
>字段描述：数值要有单位，时间要有格式，状态字段要有状态描述，以及不同状态下对于其他字段返回逻辑的关联关系。
        
| 字段类型 | 字段名称 | 说明 |
                | --- | --- | --- |
                | Boolean | isVip | 是否时Vip用户，1：是，0：否 |
                | 金额 | realPay | 订单实际付款金额，单位：元 |
                | 时间 | payTime | 订单付款时间，单位：毫秒 |
                | 日期 | payDate | 订单付款日期，格式"yyyy-MM-dd" |
                | 状态 | status | 订单状态，1：进行中（payDate不返回），2：待支付（payDate返回），3：已支付（payDate不返回）；（bool以1/0表示，状态从1+开始） |
    
5.统一定义String字段类型

```
// json
{
    "name":"张三",
    "isVip": true,
    "age":20,
    "money": 10.5
}
   
// Model.java
public class Model {
    String name;
    boolean isVip;
    int age;
    float money;
}
```
   
>如果使用的是Gson库的话，正常情况下这么定义model是可以正常解析，但是会有以下异常情况：
    
* Boolean型字段
        
```
{
    //如果传true，false以外的数据，就会解析失败
    "isVip": 20
    "isVip": 
}
```
解析报错：
        
```
（1）java.lang.IllegalStateException: Expected a boolean but was NUMBER
（2）com.google.gson.stream.MalformedJsonException: Unexpected value
```
        
* Int类型字段
        
```
{
    "age": 20.5
    "age": abc
    "age": ""
    "age": 
}
```
            
解析报错：
            
```
（1）java.lang.NumberFormatException: Expected an int but was 20.5
（2）java.lang.IllegalStateException: Expected an int but was STRING
（3）java.lang.NumberFormatException: empty String
（4）com.google.gson.stream.MalformedJsonException: Expected value
```
        
* Float类型字段
        
```
{
    "money": abc
    "money": ""
}
```
  
解析报错：
  
```
（1）java.lang.NumberFormatException: For input string: "abc"
（2）java.lang.NumberFormatException: empty String
```

>Gson库在解析到某个非法字段时，会抛出各种异常，导致整个model的解析失败。客户端没处理好的话，会因为这种时不时的脏数据引发各种奇怪的bug。解决方案：
        
* 修改Gson源码，对于字段解析失败的异常进行捕获，保证model解析完成，非正常解决方案，修改源码后Gson库就不能随便更新了，获取替换其他json解析库也变的不方便。
* 自定义JsonDeserializer，比较正常的解决思路。
        
```
public class IntegerDefaultAdapter implements JsonDeserializer<Integer> {
  @Override
  public Integer deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context)
      throws JsonParseException {
          
      // 如果integer类型的字段，进行一次类型转换
      try {
          return Integer.parseInt(json.getAsString());
      } catch (NumberFormatException e) {
      }
      return -1;
  }
}
   
String json = "{name:listen,isVip:true,age:abc,money:1.0}";
Gson gson = 
new GsonBuilder().registerTypeAdapter(int.class, new IntegerDefaultAdapter())
.create();
Model model = gson.fromJson(json, Model.class);// age字段解析出来为-1
```

* 将APP接收数据的类型定义为容错能力更强的String（推荐）。
    
```
{
    "name": "abc"
    "name": "20"
    "name": "10.2"
    "name": "true"
}
```
    
**优点：**
  * 容错性强，规避因脏数据引起的数据解析失败。
  * age，money这些字段大部分情况下都是直接展示，此时便可省去拼接 ""，或String.valueOf()等步骤。另外假设此时将age字段定义为int类型，很容易就会直接调用textView.setText(age)，那么这个age就会当成resId去执行，导致资源找不到报错，定义为String可以避免此类错误。
        
**注意事项：**
            
  * Boolean类型数据，统一返回1（true）和0（false），客户端做一层容错判断，只有1才为true，其他非1，解析失败的情况均为false，例：
       
  ```
  if(!TextUtils.isEmpty(isVip) && "1".equals(isVip)) {
        return true;
   }
        return false;
```
       
  * status类型字段从1+开始，和Boolean类型（0否，1是）区分开。"0"的含义有2种，（1）非0即为真，所以0即表示false；（2）"0"是一种未赋值的默认状态。假设此时用0表示状态1，那么就很难判断出到底时数据解析失败，使用默认值0，还是说逻辑走通并赋值为0。例：orderStatus，1:进行中，2:待支付，3:已完成。
            
  * int，float类型数据，如果不是直接展示的话，需要做一次类型转换，注意捕获异常，在解析失败的情况下，使用default值。
            
  ```
int defaultInt = -1;
try {
    defaultInt = Integer.parseInt(age);
} catch (NumberFormatException e) {
    e.printStackTrace();
}
return defaultInt;
 ```
        
6.上传/下载接口，根据md5校验数据完整性
    
* 上传，下载文件/图片时，除了file本身，还要携带该file的md5，在传输过程中可能丢失部分数据，导致文件损毁，所以需要通过md5值进行完整性校验。
* 上传成功后，正常情况后端只需要返回code表示成功/失败，在开发阶段，可以让后端将上传成功后的图片url返回，这样当我们调用完接口以后，就可以通过该url字段查看图片是否上传成功，存储的尺寸大小，模糊度等，就不用每次粘着后端帮忙看请求结果了，这个思路同样通用于其他接口，不过上线后需要将这个不必要的字段去掉。
    
  ```
{
    "code":200,
    "msg":"成功",
    "time":"1482213602000",
    "data": {
        "url":"http://www.online.com/path/pic.jpg"
    }
}
```

7.避免浮点型计算
>浮点型计算可能导致精度丢失，为了避免，可以缩小单位进行存储。例：1.5元，后端会以150分存到数据库，1.5km会存成1500m。同理，如果一个类似距离的字段，如果是展示用，则直接返回"1.5km"，如果涉及到逻辑判断与计算（如：>1000m，执行逻辑A，>1500m，执行逻辑B），可以返回"1500，单位(m)"，至少比传1.5来的方便。当然如果要计算浮点型也是可以的，需要用到BigDecimal，这么设计只是为了减少出错的可能性。

8.json数据保持良好结构
    
```
{
    "userId"...
    "userName"...
    "userPhoto"...
    "orderId"...
    "orderType"...
    "addressId"...
    "addressName"...
    "addressDetail"...
}
```

>json的3类信息user，order，address，全部堆在一起，字段多了以后，对于接口信息的读取很不直观；客户端在定义model的时候，会将全部字段定义在一个model中，如果其他地方也有用到addressId，Name，Detail等字段信息，则需要重新定义address的model，无法实现model的复用。

      
```
{
    "user":{
        "id"...
        "name"...
        "photo"...
    }
    "order":{
        "id"...
        "type"...
    }
    "address":{
        "id"...
        "name"...
        "detail"...
    }        
}
```
    
>经过优化后user，order，address字段在各自的结构体内，一眼就可以看出这个接口有哪些类型的数据。还有点要注意，如果放在同一级别，id字段就需要用userId，orderId，addressId区分开，而现在根据不同结构体区分字段类型后，直接使用id就可以了，如果还使用userId，写代码的时候就会出现
data.getUser().getUserId()的写法，就会很奇怪。


三. 瘦客户端
> 众所周知，客户端任何的修改都是需要发版的，特别是IOS需要走AppStore的审核流程。为了修一个bug，仅仅改几行代码，而重新走一轮发版流程，是很劳民伤财的。所以在接口设计的时候，也需要适当考虑这点，将业务重心交由后端，客户端保持逻辑简单。有时候，一个功能，客户端，后端都可以做，那么为什么客户端就是不做，要后段拼好提供呢？还是那句话，后端一天可以发n个版，客户端一个版本却只能发一次，有些团队一开始并没意识到这点，总觉后端就是重度业务逻辑的所在，管那么多前端的展示，字符串拼接逻辑干嘛，可是，真正到了出问题（bug或需求变更）需要发版的时候，虽然70%的锅是客户端背，但是，剩余30%也会对当初重客户端的选择而后悔，不过重点不是谁背锅，而是产品不出问题。so，为了大局，后端的RD们，我们得聊聊。

1. 客户端尽量只负责展示逻辑，不处理业务逻辑
  > 例如：客户端有个TextView，后端只给个status字段，status=1时，展示文案1；status=2时，展示文案2；这样设计的缺点是，如果以后要修改status=3时，展示文案1，那么这个status判断逻辑时写死在客户端，就没办法支持这种修改，且这种设计限定死了TextView只能展示2种文案。推荐方案是后端直接将TextView需要展示的文案下发，这样不管是status的判断，还是文案的展示，后期都是可变的。
    
2. 客户端不处理金额的计算
>例如：外卖APP，用户在下单的时候，需要选择收货地址，支付类型，优惠券等，任何一个选项的修改，都可能影响用户最后需要支付的金额。所以这里比较常见的接口设计是在每次选择完回到订单支付页面后，再发送一次请求，后端根据当前选项重新计算金额。金额永远是一款产品最重要，最敏感的信息，如果交由客户端计算，万一出错，即使少1分，都是毁灭性的，所以，关于金额，展示就好。
    
3. 客户端少处理请求参数的校验与约束提示
>例如：修改密码功能，密码规则"6-12字母，数字，下划线"，有3种做法：

 * 在发送请求前，客户端校验密码规则，如果不符合，则不发送请求。优点：规则不满足时，可以减少不必要的请求。缺点：客户端写死校验逻辑，密码规则变化时，客户端需要发版。
 *  客户端只判断null，和最短位数限制，其他校验规则交由后端处理。优点：灵活性最好。缺点：后端压力大，校验请求多。
 * 后端在通用配置的接口返回正则表达式，客户端获取后进行正则校验。优点：具有一定灵活性。缺点：开发，调试成本较高。（推荐：即使出问题，也可以清除配置，回退到第2个方案）
    
四.扩展性
> 接口的设计要具有一定的扩展性，考虑到后续版本变化，对于接口，字段的影响及变化。
    
1. 文案与图片
>对于界面上的文案，图片，特别是"xxx20分钟之内"，"xxx7天到期"这些带数字的文案，不可能永远不变的，即使和PM确认了打死不变，也最好通过常量配置接口进行下发（未下发时使用APP本地默认文案，下发时使用下发的文案），我们的原则是：变与不变都能支持。
    
2. 数据列表化：尽量用List(key, value)的数据格式定义类似列表的界面
        
 ![list.png](https://github.com/listen2code/article/blob/master/从客户端的角度设计后端的接口/screenshot/list.png?raw=true)
    
方案1：客户端在写xml的时候将左侧的"姓名"，"性别"，"年龄"写死，右侧的具体数据从json解析获得
    
```
{
    "name": "张三",
    "sex": "男",
    "age": "20岁",
    "nickName": "小张"
}
```        

方案2（推荐）：将左侧的title和右侧的value，以list(key-value)的数据形式进行下发，优点：左，右侧文案灵活配置，后期如果需要扩展，新增或删除一个条目，都可以通过后端控制。不过采用这种形式，也需要考虑实际场景，对于变化不那么频繁，数据item较少，较固定的情况下其实没有必要设计的太灵活，只会增加开发成本。
            
```
{
    "userInfos":[
    {
        "key":"姓名",
        "value":"张三"
    },{
        "key":"性别",
        "value":"男"
    },{
        "key":"年龄",
        "value":"20岁"
    },{
        "key":"昵称",
        "value":"小张"
    }]
}
```        


3.用flag替换boolean：一般情况下，一款APP都会有config接口，用于获取一些常量文案，通用配置等信息，会有很多类似开关的字段，如："isNew"，"isVip"，"isShowBalance"等等。
    
```
{
    "isNew":"1",// 是否是新用户
    "isVip":"1",// 是否是VIP用户
    "isShowBalance":"1",//是否显示侧边栏余额模块
}
```

优化方案：通过二进制第1位表示"isNew"，二进制第2位表示"isVip"，二进制第3位表示"isShowBalance"。如果有其他新增状态，不需要新增字段，就需要改变返回的数据即可。
        
```
{
    "flag":"7"// 二进制：111，表示3个状态都为true
    "flag":"5"// 二进制：101，表示isNew，isShowBalance为true，isVip为false
}
```
```
long flag = 5;
System.out.println("bit=" + Long.toBinaryString(flag));
System.out.println("isNew=" + ((flag & 1) == 1));
System.out.println("isVip=" + ((flag & 2) == 2));
System.out.println("isShowBalance=" + ((flag & 4) == 4));
  
bit=101
isNew=true
isVip=false
isShowBalance=true
```
    
五.安全性

1. 响应数据中包含用户隐私的字段数据，需要加*号。如：手机号，身份证，用户邮箱，支付账号，邮寄地址等。
    
  ```
{
    "phone":"150*****000",
    "idCard":"3500**********0555",  
    "email":"40*****00@qq.com"     
}
```
        
2. 请求参数中包含用户隐私的字段参数，如：登陆接口的密码字段，需要进行加密传输，避免被代理捕捉请求后获取明文密码。
    
3. 客户端和服务器通过约定的算法，对传递的参数值进行签名匹配，防止参数在请求过程中被抓取篡改。密钥记得放到so中，放在java层太不安全，so中要进行keystore反向签名校验，避免so被获取后直接调用获取算法。
    
    * so中要进行keystore反向签名校验
    
        > Java层在进行参数签名计算的时候需要获取app本地存储的密钥，调用NativeHelper.getKey()，在so中通过反射调用java层的getSignature()，比较是否和so中存储的keyStore哈希值一致，如果是则返回密钥，不是则返回空字符串。
    
        Java层的NativeHelper.java
        
        ```
        package com.listen.test;
        public class NativeHelper {
            static {
                System.loadLibrary("native-lib");
            }
            // 调用so获取密钥
            public native String getKey();
        
            // 获取当前keyStore的hash值
            public String getSignature() {
                final String packname = BaseApplication.getInstance().getPackageName();
                PackageInfo packageInfo;
                try {
                    packageInfo =
                        BaseApplication.getInstance()
                            .getPackageManager()
                            .getPackageInfo(packname, PackageManager.GET_SIGNATURES);
                    Signature[] signs = packageInfo.signatures;
                    Signature sign = signs[0];
                    return sign.hashCode() + "";
                } catch (Throwable t) {
                    if (null != t) {
                        t.printStackTrace();
                    }
                }
                return "";
            }
        }
        ```
        
         so层的native-lib.c
         
         ```
         // 字符串转字符
         char* _JString2CStr(JNIEnv* env, jstring jstr) {
            char* rtn;
            jclass clsstring = (*env)->FindClass(env, "java/lang/String");
            jstring strencode = (*env)->NewStringUTF(env, "GB2312");
            jmethodID mid = (*env)->GetMethodID(env, clsstring, "getBytes",
                    "(Ljava/lang/String;)[B");
            jbyteArray barr = (jbyteArray)(*env)->CallObjectMethod(env, jstr, mid,
                    strencode); // String .getByte("GB2312");
            jsize alen = (*env)->GetArrayLength(env, barr);
            jbyte* ba = (*env)->GetByteArrayElements(env, barr, JNI_FALSE);
            if (alen > 0) {
                rtn = (char*) malloc(alen + 1); //"\0"
                memcpy(rtn, ba, alen);
                rtn[alen] = 0;
            }
            (*env)->ReleaseByteArrayElements(env, barr, ba, 0);
            return rtn;
        }
    
         char* storeKeyHash = "1234567890";// 该值可以通过java层的getSignature获取
         
         JNIEXPORT jstring JNICALL Java_com_listen_test_NativeHelper_getKey(
                JNIEnv *env, jobject obj, jbyteArray array) {
            // 反射获取当前keyStore的hash值
            jclass jClazz = (*env)->FindClass(env, "com/listen/test/NativeHelper");
            jmethodID jmethodid = (*env)->GetMethodID(env, jClazz, "getSignature",
                    "()Ljava/lang/String;");
            jstring appSign = (jstring)(*env)->CallObjectMethod(env, obj, jmethodid);
        
            // 判断是否是本程序的签名哈希值
            char* charAppSign = _JString2CStr(env, appSign); //将jstring转换为cha*
            if (strcmp(charAppSign, storeKeyHash) != 0) {
                return (*env)->NewStringUTF(env, "");//keyStore的hash不一致，不是在当前app种调用该so
            }
            return (*env)->NewStringUTF(env, "秘钥值");//keyStore的hash一致，返回密钥
        }
         ```
        
六.兼容性
> APP1.0在使用接口A，如果此时在开发1.1的时候修改了接口A的逻辑，在1.1发版的时候线上就会出现2个版本的客户端访问同一个接口A，为了保证1.0客户端调用接口A不会出错，就需要通过version字段或path中的"v1/login"，"v2/login"进行区分，不同版本客户端访问同一接口时处理逻辑要各自独立。

1. 接口/字段的删除，修改要谨慎：
>对于已经存在的接口进行修改，需要考虑对线上版本的影响，尽量是数据含义，和新增字段，而不是去修改。
2. md5缓存的兼容性：
>如果1.0的接口A存在md5缓存，正常都是后端上线后再发布1.1客户端的顺序，如果在后端上线后，1.1还没发布的情况下，此时1.0的客户端就缓存了1.1后端逻辑的md5，在更新成1.1的时候，md5没有变，就有可能缓存的还是1.0的数据，所以比较推荐后端在计算md5的时候把version加上，这样更新APP可以保证md5是不一样的。 


七.性能优化

1. 合并接口
>为了减少客户端和服务器建立连接和断开连接消耗的时间，资源，电量，尽量避免频繁的间隔网络请求。业务场景允许的情况下，尽量1个页面对应1个接口。原先一个页面要通过多个请求获取多种类型数据的情况，最好能通过一个接口全部获取得到。又如：在调用B接口前需要A接口的前置数据的情况，可以让后端支持下，在调用A接口时直接返回B接口的数据，减少类似这种的连续请求。
    
2. 字段精简
    >定义字段名时，在保证良好可读性的前提下，尽量精简，减少流量的消耗

     ```
    {
      "orderDescription" >> "orderDesc"
      "oldPassword" >> "oldPwd"
      "longitude" >> "lng"
      "latitude" >> "lat"
    }
    ```
    
3. md5缓存
>对于频繁调用，且数据不常变化的接口（config配置接口），可以在返回的数据中添加md5字段（用于校验除md5外其他数据是否变化），在下次请求的时候将这个md5作为参数传给后端，md5没有变化的情况下，不返回data，客户端可以直接使用上次请求缓存在本地的data。
    
     ![md5.png](https://github.com/listen2code/article/blob/master/从客户端的角度设计后端的接口/screenshot/md5.png?raw=true)
    
4. 无用字段清理
    >每个版本的接口更新后，需要将无用字段进行清理。或者同个接口不同状态下需要返回的字段各不相同的时候，当次请求不需要的字段需要提醒后端不必下发，避免传输无用数据浪费用户流量。

5. 图片裁剪服务
    > 客户端上传图片后，当需要在列表这些图片区域较小的地方展示的时候，没必要直接加载原图，可以先在后端通过图片裁剪服务处理后再进行展示。例：
http://image-demo.img-cn-hangzhou.aliyuncs.com/example.jpg@100h_100w_1e_1c?spm=5176.doc32223.2.3.jmkKF9&file=example.jpg@100h_100w_1e_1c， 这是阿里云的图片裁剪服务，在url后面直接拼上裁剪参数，就可以实现将原图居中裁剪成100*100的缩略图。当需要展示高清图的时候，再加载原图的url。
    
6. 局部刷新
    > 一个页面，如果之前已经加载了20%的数据，那么就不需要每次都返回100%数据，只要返回剩余80%即可。例：订单列表页面，每个item已经具有类似orderId，orderDesc等字段，那么点击进入订单详情的时候，orderId，orderDesc就可以从订单列表传递过来即可，详情页的请求只需要返回订单相关的剩余数据，客户端需要额外处理数据组装逻辑，将前一个页面传递过来的字段和详情页请求到的字段组装成完整的model数据。
    
7. wifi与移动网络的区别对待
    >WiFi连接下，网络传输的电量消耗要比移动网络少很多，应该尽量减少移动网络下的数据传输，多在WiFi环境下传输数据。例：crash日志上报，数据统计接口等，可以在移动网络的情况下请求频率降低，或缓存，在wifi网络时上调请求频率，或将缓存的数据统一上报。还有上文提到的，如果是wifi网络状态下，就下发高清图提升用户体验，移动网络状态就下发缩略，或裁剪图。


八.体验优化
> 设计接口时，不能只考虑减少流量消耗，性能优化等，特定场景下用户体验的优化才是最高优先级的。

1. 通过预加载降低对网络的依赖
> 使用APP的场景为网络较差的情况。例：配送员在使用配送APP的时候，商家地址如果在地下室，或配送员进入电梯的时候，这时候常要查看订单详情，网络信号又比较差的，就会影响正常工作。可以考虑在订单列表的接口中，将订单详情的数据一起请求下来，并通过md5判断详情页面数据是否变化，避免重复加载，这样其实用户在网络比较好的情况下请求一次列表后，再进入详情页，就不再需要重新请求，对网络的依赖也是最小的。同理，对于一些阅读类APP，前几页的文章，用户查看详情的概率较高，可以在返回文章列表的时候携带正文内容，则可以实现秒开详情，也可以判断网络状态，wifi场景下可以将详情数据都返回。

  ```
       {
            "md5"... // 校验所有item的detail，只有在新订单，或订单完成后移除的情况下，md5才会变化
            "orderList":[{
                "id"...
                "status"...
                "detail":{ // detail中尽量只保留变化情况较少的字段，避免md5频繁变化，如status就移出到item中存放
                    "type"...
                    "desc"...
                }
            },{
                "id"...
                "status"...
                "detail":{
                    "type"...
                    "desc"...
                }
            }]
       }
       ```


* 总结
    > 暂时先这么多吧，水平有限，权当是抛砖引玉，如果有更好的设计，请在文章下方留言告诉我，交流想法，互相学习。谢谢支持～

