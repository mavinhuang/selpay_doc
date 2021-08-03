对接大概流程：提交订单号，生成二维码，扫码支付回调，轮询订单状态等流程。

#### **1.提交订单号**

商户自研平台完成订单以后，需要通过api提交订单信息到selpay平台，信息如下：

地址:  https://upweb.selpay.cn/prod-api/up/public/AddOrderInfo

请求方式：post

参数：

```json
{
  "appKey": "",  //平台中appKey
  "orderNo": "", //商户平台生成的订单编号，推荐uuid 长度32位
  "total": 0     //支付总额，请使用int类型，例如：0.01 写成 1
}
```

实例：

```json
{
  "appKey": "5f53481517b449b0a8c8dbe808d8bf5c",
  "orderNo": "asdfsdfsdfsdfds",
  "total": 12
}
```

返回结果：

```json
{
  "msg": "操作成功",
  "code": 200,    //状态码
  "data": {
    "scanUrl": "https://up.selpay.cn/#/pages/rt/index?appKey=5f53481517b449b0a8c8dbe808d8bf5c&no=asdfsdfsdfsdfds",  支付地址，用于生成二维码在ui展示，提供给客户扫码支付
    "order": {
      "appKey": "5f53481517b449b0a8c8dbe808d8bf5c",
      "orderNo": "asdfsdfsdfsdfds",
      "total": 12
    }
  }
}
```



#### **2.生成支付码**

从接口返回结果中获取scanUrl，生成二维码提供给客户扫码支付。UI上建议提供快速重建订单的功能，本质就是修改订单号，主要原因是微信不支持重复订单号的提交，这种情况就需要刷新订单号才能够支付。主要是考虑到用户支付过程中按错了，再次扫码就会提示订单已存在，导致无法支付的情况。

#### **3.支付回调**

当用户支付成功以后，系统会回调商户在平台中设置的回调地址。平台会把相关支付和用户信息发送到该 URL，商户需要接收处理信息。

 

对后台通知交互时，如果平台收到商户的应答不是纯字符串success或超过5秒后返回时，平台认为通知失败，平台会通过一定的策略（**通知频率为****0/15/15/30/180/1800/1800/1800/1800/3600，单位：秒**）间接性重新发起通知，尽可能提高通知的成功率，但不保证通知最终能成功。商户系统能够正确处理重复通知的机制。

建议流程： 首先检查对应业务数据的状态，  如果没有处理过再进行处理， 如果处理过直接返回结果成功。 在对业务数据进行状态检查和处理之前， 要采用数据锁进行并发控制， 以避免函数重入造成的数据混乱。

 

特别注意：商户后台接收到通知参数后，要对接收到通知参数里的订单号out_trade_no和订单金额total_fee和自身业务系统的订单和金额做校验，校验一致后才更新数据库订单状态

 平台采用 post方式给商户系统（通知参数内容为xml的字符串）

 

| 字段名                                              | 变量名             | 必填 | 类型         | 说明                                                         |
| --------------------------------------------------- | ------------------ | ---- | ------------ | ------------------------------------------------------------ |
| 版本号                                              | version            | 是   | String(8)    | 版本号，version默认值是2.0。                                 |
| 字符集                                              | charset            | 是   | String(8)    | 可选值 UTF-8 ，默认为 UTF-8。                                |
| 授权交易机构                                        | sign_agentno       | 否   | String(12)   | 授权交易的服务商机构代码，商户授权给服务商交易的情况下返回，签名使用服务商的密钥 |
| 连锁商户号                                          | groupno            | 否   | String(15)   | 连锁商户为其下门店发交易的情况返回，签名使用连锁商户的密钥   |
| 签名方式                                            | sign_type          | 是   | String(8)    | 签名类型，取值：MD5默认：MD5                                 |
| 返回状态码                                          | status             | 是   | String(16)   | 0表示成功，非0表示失败此字段是通信标识，非交易标识，交易是否成功需要查看 result_code 来判断 |
| 返回信息                                            | message            | 否   | String(128)  | 返回信息，如非空，为错误原因签名失败参数格式校验错误         |
| 以下字段在 status 为 0的时候有返回                  |                    |      |              |                                                              |
| 业务结果                                            | result_code        | 是   | String(16)   | 0表示成功非0表示失败                                         |
| 商户号                                              | mch_id             | 是   | String(15)   | 商户号，由平台分配                                           |
| 设备号                                              | device_info        | 否   | String(32)   | 终端设备号                                                   |
| 随机字符串                                          | nonce_str          | 是   | String(32)   | 随机字符串，不长于 32 位                                     |
| 错误代码                                            | err_code           | 否   | String(32)   | 参考错误码                                                   |
| 错误代码描述                                        | err_msg            | 否   | String (128) | 结果信息描述                                                 |
| 签名                                                | sign               | 是   | String(32)   | MD5签名结果，详见“安全规范”                                  |
| 以下字段在 status 和 result_code 都为 0的时候有返回 |                    |      |              |                                                              |
| 用户标识                                            | openid             | 是   | String(128)  | 用户在商户 appid 下的唯一标识                                |
| 交易类型                                            | trade_type         | 是   | String(32)   | pay.weixin.jspay                                             |
| 是否关注公众账号                                    | is_subscribe       | 是   | String(1)    | 用户是否关注公众账号，Y-关注，N-未关注，仅在公众账号类型支付有效 |
| 支付结果                                            | pay_result         | 是   | Int          | 支付结果：0—成功；其它—失败（支付结果以此字段为准）          |
| 支付结果信息                                        | pay_info           | 否   | String(64)   | 支付结果信息，支付成功时为空                                 |
| 平台订单号                                          | transaction_id     | 是   | String(32)   | 平台交易号                                                   |
| 第三方订单号                                        | out_transaction_id | 是   | String(32)   | 第三方订单号                                                 |
| 子商户是否关注                                      | sub_is_subscribe   | 否   | String(1)    | 用户是否关注子公众账号，Y-关注，N-未关注，仅在公众账号类型支付有效 |
| 子商户appid                                         | sub_appid          | 否   | String       | 子商户appid                                                  |
| 用户openid                                          | sub_openid         | 否   | String(128)  | 用户在商户 sub_appid 下的唯一标识                            |
| 商户订单号                                          | out_trade_no       | 是   | String(32)   | 商户系统内部的定单号，32个字符内、可包含字母                 |
| 总金额                                              | total_fee          | 是   | Int          | 总金额，以分为单位，不允许包含任何字、符号                   |
| 现金支付金额                                        | cash_fee           | 否   | Int          | [现金支付金额订单现金支付金额，详见支付金额](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=4_2) |
| 现金券金额                                          | coupon_fee         | 否   | Int          | 现金券支付金额<=订单总金额， 订单总金额-现金券金额为现金支付金额，以分为单位 |
| 免充值优惠金额                                      | mdiscount          | 否   | Int          | 免充值优惠金额，以分为单位                                   |
| 货币种类                                            | fee_type           | 否   | String(8)    | 货币类型，符合 ISO 4217 标准的三位字母代码，默认人民币：CNY  |
| 附加信息                                            | attach             | 否   | String(127)  | 商家数据包，原样返回                                         |
| 付款银行                                            | bank_type          | 是   | String(16)   | 银行类型                                                     |
| 银行订单号                                          | bank_billno        | 否   | String(32)   | 银行订单号，若为微信支付则为空                               |
| 支付完成时间                                        | time_end           | 是   | String(14)   | 支付完成时间，格式为yyyyMMddHHmmss，如2009年12月27日9点10分10秒表示为20091227091010。时区为GMT+8 beijing。该时间取自平台服务器 |

 

**后台通知结果反馈**

平台服务器发送通知，post发送XML数据流，商户notify_Url地址接收通知结，商户做业务处理后，需要以纯字符串的形式反馈处理结果，内容如下： 

| 返回结果       | 结果说明                                                     |
| -------------- | ------------------------------------------------------------ |
| success        | 处理成功，平台收到此结果后不再进行后续通知                   |
| fail或其它字符 | 处理不成功，平台收到此结果或者没有收到任何结果，系统通过补单机制再次通知 |

处理notify的逻辑，以java为例，其他类似：

```java
public void notify(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        try {
            req.setCharacterEncoding("utf-8");
            resp.setCharacterEncoding("utf-8");
            resp.setHeader("Content-type", "text/html;charset=UTF-8");

            String resString = XmlUtils.parseRequst(req);
            System.out.println("通知内容：" + resString);

            String respString = "success";
            if(resString != null && !"".equals(resString)){
                Map<String,String> map = XmlUtils.toMap(resString.getBytes(), "utf-8");
                String res = XmlUtils.toXml(map);
                //System.out.println("通知内容：" + res);
                
                String status = map.get("status");
                if(status != null && "0".equals(status)){
                    String result_code = map.get("result_code");
                    if(result_code != null && "0".equals(result_code)){
                       if(TestPayServlet.orderResult == null){
                           TestPayServlet.orderResult = new HashMap<String,String>();
                        }
                        String out_trade_no = map.get("out_trade_no");
                       /*
                       写自己的处理逻辑即可
                       */
                        
                     }
                      
                 respString = "success";
                   
                }
            }
            resp.getWriter().write(respString);
        } catch (Exception e) {
            e.printStackTrace();

        }
       
    }
```



#### **4.轮询订单支付状态**

商户系统需要轮询订单支付状态，如果订单已支付就触发发货等操作，如果没有就继续轮询，直到用户推出扫码支付界面。