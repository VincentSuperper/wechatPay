# 微信支付(公众号支付)

## 准备工作

### 一、申请商户平台

1. 登录公众平台，点击左侧菜单中的微信支付

    ![左侧菜单](https://5b0988e595225.cdn.sohucs.com/images/20171102/a7730d06c918416a90571a6af7ff8d98.png)

2. 进入支付申请，按照指引填写相关的信息并提交

    ![支付申请](https://5b0988e595225.cdn.sohucs.com/images/20171102/d8de402e3c054424b4d6f0830364dc5e.png)

3. 资料审核通过后，支付宝会向登记的银行帐号打一笔款，收到款后，两次登录到公众平台填写正确的数字，签约成功

    ![资料审核](https://5b0988e595225.cdn.sohucs.com/images/20171102/b9b5172aecc74cda85d079b00c775e0e.png)

>*注意：在申请支付功能的时候，不要申请成了服务商，需要的是商户。虽然服务商也具备支付功能，但在调用 接口时会比商户复杂一点，申请过程也繁琐一点。另外服务商和商户是具备不同的业务功能的，常见的微商场用的更多是商户平台。对于服务商和商户的区别我理解的是，服务商是对商户提供技术支持的第三方团队。*

### 二、配置支付参数

1. 设置支付目录

    在[微信商户平台](pay.weixin.qq.com)设置您的公众号支付支付目录，设置路径：商户平台-->产品中心-->开发配置，如下图所示。公众号支付在请求支付的时候会校验请求来源是否有在商户平台做了配置，所以必须确保支付目录已经正确的被配置，否则将验证失败，请求支付不成功。

    *请确保实际支付时的请求目录与后台配置的目录一致，否则将无法成功唤起微信支付。*

    支付授权目录要求：
    1. 所有使用公众号支付方式发起支付请求的链接地址，都必须在支付授权目录之下
    2. 最多设置3个支付授权目录，且域名必须通过ICP备案
    3. 头部要包含http或https，须细化到二级或三级目录，以左斜杠“/”结尾

    ![设置支付目录](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_3_1.png)

2. 设置授权域名

    开发公众号支付时，在统一下单接口中要求必传用户openid，而获取openid则需要您在公众平台设置获取openid的域名，只有被设置过的域名才是一个有效的获取openid的域名，否则将获取失败。具体界面如下图所示:

    ![设置授权域名](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_3_2.png)

## 开发步骤

1. 订单页面开发

    微信网页支付的核心就在这一步，这里也是坑最多，流程最复杂的地方，如果把这里搞明白了，微信网页支付基本上就算完成了80%

    订单支付分为两部分，一个是前端页面，一个是后端业务接口。微信网页支付主要是依赖前端js方法实现，后端接口仅是给前端js方法提供必要的参数

    在订单页面，当用户确认支付信息无误，就点击确认付款按钮，这时调用后端接口，后端接口进行“封装订单，封装微信支付参数”的处理后，将微信支付参数返回到前端页面。如果参数无误，则会自动调起微信支付，用户输入支付密码后，完成支付

    这里，主要是wx.config和wx.chooseWXPay两个方法，第一个方法进行微信支付验证，验证用户身份，支付参数是否合法。当第一个方法验证通过后，就可以调用wx.chooseWXPay，如果这个方法的参数也都准确无误，则会自动弹出微信支付页面

    wx.config所需参数：

        debug: false, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。

        appId: payReq.appid, // 必填，公众号的唯一标识（开通微信支付的公众号APPID）

        timestamp: payReq.timestamp , // 必填，生成签名的时间戳

        nonceStr: payReq.noncestr, // 必填，生成签名的随机串

        signature: payReq.signature,// 必填，签名，见附录1

        jsApiList: ['chooseWXPay'] //必填，需要使用的JS接口列表，所有JS接口列表见附录2

    这里需要我们获取的参数有timestamp，nonceStr，signature 其他参数都是现成的，直接用就可以

    wx.chooseWXPay所需参数：

        timestamp:payReq.timestamp,//支付签名时间戳，注意微信jssdk中的所有使用timestamp字段均为小写。但最新版的支付后台生成签名使用的timeStamp字段名需大写其中的S字符

        nonceStr:payReq.noncestr, // 支付签名随机串，不长于 32 位

        package: payReq.prepayId,//统一支付接口返回的prepayid参数值，提交格式如：prepayid=***）

        signType: 'MD5', // 签名方式，默认为'SHA1'，使用新版支付需传入'MD5'

        paySign: payReq.paySign, // 支付签名

    这里需要我们获取的参数有：timestamp,nonceStr,package,paySign其他参数都是现成的，直接用就可以

    合并起来，我们总共要获取的参数是：

        timestamp,nonceStr,package,signature,paySign

    这些参数，我们通过后端接口获取，然后传递给前端页面。当用户在订单页面点击确认支付后，请求后端接口，后端接口可以接到参数:openid,订单金额。封装订单信息根据自身的业务封装对象即可。重点说前端页面所需参数的获取

2. 调用“统一下单”接口，获取参数：timestamp,nonceStr,package

    接口文档：https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=9_1

    **timestamp,nonceStr**：按照文档中“统一下单”接口中参数的算法，可以直接生成。

    **package**：这个参数获取比较麻烦，文档中需要用到两个特殊的参数：mch_id（商户号），key（商户密码），这两个参数要去商户平台拿。另外需要一个签名参数sign，这个需要用其他参数合并生成

    商户号：

    ![商户号](http://images.gitbook.cn/015868a0-67a7-11e7-8182-1d8aa3f22897)

    商户KEY：

    ![商户KEY](http://images.gitbook.cn/18e6aea0-67a7-11e7-8182-1d8aa3f22897)

    拿到这些参数后，按照签名算法，可以得到sign，注意，此处是MD5加密

    最后，按照统一下单接口的文档，传递参数调用接口，在返回结果中可以拿到参数名为：prepay_id的参数值，就是package

    **signature**：获取这个签名参数，sha1加密，需要参数jsapi_ticket，所有的签名算法都是一样的，只是所需要的参数不一样，所以，这里的关键就是获取jsapi_ticket。这块比较绕，通过查阅大量文档最终确定了下面的关系：

    获取signature需要用到参数`jsapi_ticket` → 获取参数`jsapi_ticket`需要调用前面说的第三个接口，并且用到参数`access_token` → 获取参数`access_token`需要调用前面说的第二个接口

    所以，按照这个顺序依次获取，当拿到`jsapi_ticket`参数后，我们来生成签名signature

    这里payReq是上面统一下单接口的返回结果集，url是当前接口的URL路径。将签名拼出来之后，直接加密，就拿到了我们要的签名

    **paySign**：签名，MD5加密。

    整个签名生成相对容易些，因为需要用到的参数现在已经全部具备了，只需要拼接签名，然后加密即可

        String paySignStr = "appId=" + APPID + "&nonceStr=" + payReq.get("noncestr").toString() + "&package=" + payReq.get("prepayId").toString() + "&signType=MD5" + "&timeStamp=" + payReq.get("timestamp").toString() + "&key=" + APIKEY;
    
    *关于timeStamp中S大小写的问题，很多文档说的模糊不清，这里做一下总结*

    *只需要记住一点，所有使用timeStamp的地方，只有在生成paySignStr签名时，用的到timeStamp作为key的S是大写，其他地方，都小写*

    至此，微信网页支付所需要的所有参数获取完毕，只需要将这些参数从服务端返回给前端页面的两个JS方法（wx.config和wx.chooseWXPay），当在页面点击支付按钮时，调用这两个方法，就可以看到期待已久的微信支付弹层

## 总结

- 几个关键点

    >两个方法(wx.config和wx.chooseWXPay)

    >五个参数(timestamp,nonceStr,package,signature,paySign)

    >三个接口
    
    >三个页面以及三个文档。

- 签名的生成，整个过程用到了三个签名

  **第一个签名sign**: 在统一下单接口中的请求参数中，sign做为这个接口的请求参数之一，需要用其他参数生成之后，才能作为请求参数再去请求，这里用的是MD5加密

  **第二个签名signature**: 这个签名是给wx.config用的，加密方式是SHA256
  
  **第三个签名paySign**: 这个签名是给wx.chooseWXPay用的，这里用的是MD5加密
