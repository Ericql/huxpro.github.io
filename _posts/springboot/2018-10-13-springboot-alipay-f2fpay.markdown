---
layout:       post
title:        "SpringBoot之支付宝支付 -- 当面付"
subtitle:     "当面付"
date:         2018-10-13 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - SpringBoot
---
# SpringBoot之支付宝支付 -- 当面付
## 内容目录
[TOC]
## 当面付介绍
&emsp;&emsp;当面付产品包括条码支付、扫码支付和声波支付,当面付就是买家和卖家进行面对面支付.其中条码支付和扫码支付使用的最多,所以主要介绍这两种.
### 条码支付
&emsp;&emsp;条码支付是支付宝给到线下传统行业的一种收款方式.商家使用扫码枪等条码识别设备扫描用户支付宝钱包上的条码/二维码,完成收款.用户仅需出示付款码,所有收款操作由商家端完成.
其业务流程如下  
![条码支付业务流程](/img/in-post/SpringBoot/Alipay/条码支付业务流程.png)
### 扫码支付
&emsp;&emsp;扫码支付指用户打开支付宝钱包中的"扫一扫"功能,扫描商家展示在某收银场景下的二维码并进行支付的模式.该模式适用于线下实体店支付、面对面支付等场景
其业务流程如下:  
![扫码支付业务流程](/img/in-post/SpringBoot/Alipay/扫码支付业务流程.png)
## 当面付流程接入
### 第一步:创建应用并获取APPID
&emsp;&emsp;我们现在集成第三方的功能(无论是支付宝还是微信)都需要到对应的开放平台去进行账号注册、创建应用然后提交审核,从而获取到应用的配置信息,如AppId等  
&emsp;&emsp;此处由于本人并不具有企业支付宝账号,从而有些功能无法开启,只能去采取沙箱环境去测试.打开[蚂蚁金服开放平台](https://docs.open.alipay.com/),用自己的支付宝"扫一扫"进行登录,然后下拉"开发者中心"的"沙箱",进入沙箱环境.即可获取到AppId及网关等,具体如下图:  
![沙箱进入步骤](/img/in-post/SpringBoot/Alipay/进入沙箱.jpg)
### 第二步:配置密钥
&emsp;&emsp;开发者调用接口前需要先生成RSA密钥,RSA密钥包含应用私钥(APP_PRIVATE_KEY)、应用公钥(APP_PUBLIC_KEY).生成密钥后在开放平台开发者中心进行密钥配置,配置完成后就会显示出支付宝公钥(ALIPAY_PUBLIC_KEY).
具体可参考支付宝的[RSA密钥生成教程](https://docs.open.alipay.com/291/105971)
### 第三步:设置应用网关和回调地址
&emsp;&emsp;应用网关: 一般是项目上线对应的域名,即必须是外网可以访问的地址,此处本人是采取购买隧道的方式实现,也可使用阿里云等方式.  
&emsp;&emsp;授权回调地址:是我们自己项目一个访问接口请求,当支付宝支付成功后会异步通知到这个地址上,通知此次支付结果是成功还是失败,例如：http://www.eric.com/pay/alipay/notify
## 当面付代码示例
### 搭建和配置开发环境
1.下载服务端的SDK  
&emsp;&emsp;蚂蚁金服开放平台提供了[开放平台服务端SDK](https://docs.open.alipay.com/54/103419),内部封装了签名&验签、HTTP接口请求等基础功能,我们只需将SDK以maven方式引入我们自己的项目即可    
&emsp;&emsp;当然SDK对应的代码和示例demo,蚂蚁金服也均有提供,具体可参考[SDK&Demo](https://docs.open.alipay.com/194/105201/)  
2.接口调用配置
&emsp;&emsp;在SDK调用前需要进行初始化,初始化,初始化(重要的事情说三遍),以JAVA代码为例:
```
AlipayClient alipayClient = new DefaultAlipayClient(URL, APP_ID, APP_PRIVATE_KEY, FORMAT, CHARSET, ALIPAY_PUBLIC_KEY, SIGN_TYPE);
```
关键参数说明:  

| 配置参数 | 参数解释 | 获取方式/值 |  
|------ |----- | ----- |  
| URL | 支付宝网关(固定) | https://openapi.alipay.com/gateway.do |
| APP_ID | APPID即创建应用后生成 | 获取见上面的创建应用并获取APPID |
| APP_PRIVATE_KEY | 开发者应用私钥 | 获取见上面配置密钥 |
| FORMAT | 参数返回格式,只支持json | json(固定) |
| CHARSET | 请求和签名使用的字符编码格式,支持GBK和UTF-8 | 根据实际情况配置 |
| ALIPAY_PUBLIC_KEY | 支付宝公钥,由支付宝生成 | 获取见上面配置密钥 |
| SIGN_TYPE | 商户生成签名字符串所使用的签名算法类型,目前支持RSA2和RSA | RSA2 |
### 条码支付示例
#### 场景介绍
&emsp;&emsp;条码支付是支付宝给到线下传统行业的一种收款方式.商户使用扫码枪等条码识别设备扫描用户支付宝钱包上的条码/二维码,完成收款.用户仅需出示付款码,所有操作由商户端完成
#### 调用流程
![条码支付调用流程](/img/in-post/SpringBoot/Alipay/条码支付调用流程.jpg)  
1.商户系统将用户付款码与订单信息一起通过交易支付接口[alipay.trade.pay](https://docs.open.alipay.com/api_1/alipay.trade.pay)请求到支付宝,并从接口同步返回中获取支付结果  
2.根据公共返回参数中的code,这笔交易可能有四种状态:支付成功(10000),支付失败(40004),等待用户付款(10003)和未知异常(20000)

| 结果码 | 说明 | 处理方式 |
|------ |----- | ----- |
| 10000 | 支付成功 |记录交易结果并在客户端显示支付成功，进入后续的业务处理 |
| 40004 | 支付失败 | 记录交易结果并在客户端显示错误信息 |
| 10003 | 等待用户付款 | 发起轮询流程：等待5秒后调用交易查询接口[alipay.trade.query](https://docs.open.alipay.com/api_1/alipay.trade.query)通过支付时传入的商户订单号(out_trade_no)查询支付结果(返回参数TRADE_STATUS),如果仍然返回等待用户付款(WAIT_BUYER_PAY),则再次等待5秒后继续查询,直到返回确切的支付结果(成功TRADE_SUCCESS或已撤销关闭TRADE_CLOSED),或是超出轮询时间.在最后一次查询仍然返回等待用户付款的情况下,必须立即调用[交易撤销接口alipay.trade.cancel](https://docs.open.alipay.com/api_1/alipay.trade.cancel)将这笔交易撤销,避免用户继续支付 |
| 20000 | 未知异常 | 调用查询接口确认支付结果,详见[异常处理](https://docs.open.alipay.com/api_1/alipay.trade.cancel) |
#### 条码示例代码
##### 利用SDK快速接入
```
AlipayClient alipayClient = new DefaultAlipayClient("https://openapi.alipay.com/gateway.do", APP_ID, APP_PRIVATE_KEY, "json", CHARSET, ALIPAY_PUBLIC_KEY, "RSA2"); //获得初始化的AlipayClient
AlipayTradePayRequest request = new AlipayTradePayRequest(); //创建API对应的request类
request.setBizContent("{" +
"    \"out_trade_no\":\"20150320010101001\"," +
"    \"scene\":\"bar_code\"," +
"    \"auth_code\":\"28763443825664394\"," +
"    \"subject\":\"Iphone6 16G\"," +
"    \"store_id\":\"NJ_001\"," +
"    \"timeout_express\":\"2m\"," +
"    \"total_amount\":\"88.88\"" +
"  }"); //设置业务参数
AlipayTradePayResponse response = alipayClient.execute(request); //通过alipayClient调用API，获得对应的response类
System.out.print(response.getBody());
// 根据response中的结果继续业务逻辑处理
```
SDK快速接入方式完成上述代码即可条码支付
##### 利用官方Demo使用SpringBoot方式接入
1.pom文件引入Alipay依赖
```
<dependency>
	<groupId>com.alipay</groupId>
	<artifactId>alipay-trade-sdk</artifactId>
	<version>20161215</version>
</dependency>
<dependency>
	<groupId>com.alipay</groupId>
	<artifactId>alipay-sdk-java</artifactId>
	<version>3.3.0</version>
</dependency>

<dependency>
	<groupId>commons-configuration</groupId>
	<artifactId>commons-configuration</artifactId>
	<version>1.10</version>
</dependency>
```
2.properties文件设置Alipay配置
```
pay:
  alipay:
    gatewayUrl: https://openapi.alipaydev.com/gateway.do
    appid: 2016092200568796
    privateKey: MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCHPm+7TPpe7W3H9entjOvGLi/vAXhjd4JkUCd2uqlLW3hyJ6JyUPq+ydZkaUGQW6mg+VWY+v08Cx20wyqfMOklNf6IbuoC4f947XTOOfAY6x9DNKOPIdTFl2fMYcSV0weZEHntxcarFkSMCOSSfCT2mO4Lfi5Qv1zLcIiPCP59phMEUeSJHkihHhuOlrFoM6th95nh+hfQadtSzm09vF7MxKcQzIOiLyergyZgwzkzVrNoAYBholE77PlQ0s9vryXpu6bNBSDjL/6Z7MbSUqK7uQENhxipvifx+lXWzT3ILFY+OQ0QNcVR24XHpsvAbpd4npg0rVWNzGgnI0njXmXPAgMBAAECggEACImHchJc56sjN/EtECLKK1t1CShVminMIFry8sq7rxcaFlKsLX0xJuQE1ZfTXLJ8lb3Hin2liKnG+UcspJnozcGHzML7oKz1fIO40N/VaS1Gbu6euIVRMhvpoHw3daG5pA7nM3w9m0UvlItnKlwN1Uc4F5+ietRpnin/ZNATiIjgnC2Kf0L/Trwn5tXxtD5eB070uRuifgLFFy0al7/j6oiN+Oe1QRWHyHik4dJkYigxfqN7eTMy8rGRFValQUwHcHlUw9/Yl3Xgr/UXxrbe5AP53ES1eOatZ5nbYTXmXia8OOU+CS6EhNaWEfJJxqEVFxhllJuY3Yv+oGYZxapJcQKBgQDQUMtCN0LdwuYXNNxZXhkxQibQfontlOu2voKkFLJLSl69lZbRp150O7Md29z4UDtiimeLuyHsHMYGhvpk6fABTvFgVlfX71v2/Yi/bP+ncAZAtAdA84ndU+39Gp14hmh1D4lN77y3p3XMCFQ4g6PflNjjYCeoNyuue1Xnu7fTdwKBgQCmM6vYKwcCDxNn+pCqNIyRaVSm+pcc04Ri9VAR4QrW5uzHyjBs7xI9nbObtEhx/IEB2sGdO5eXUEDj4Y/6pVmc/j8XTEA49+IhHaxCkMlChvz0ML8pmpikkxREvlXKk1FA7zrrl76QVdyqbg0ZtYeYYatNrMiqHJeuEzi8VFMmaQKBgQC/0mYYp0JPanTt0aNGN7wC++M6AguIVqVnNa6e4N/9LJJpCSJEFFaJuZ+KUzb7AQZuCvymUr896JEA2bIg0rpKuiLSjy98i9Cnc3dErl4MFL/tPNmhGaFNyUdQ1f1DSqFNiezpc2TXyMBUDSdgkveHnkzJs3VRFNyIYtIL/XOcqQKBgEC743Tg3Wvp308igvIoYY/JjNU0yWLK58d7cOJl2sj1TMhMciwbuekR4YEF6SmshbrpL3xEV7jx4zRfCKtBd/Pz+zLh2inWMtdfLVcH+bvVw/SAgBR+SHHhb4WO9O9gDcfS5goZInopVzdygdu/nr61W/l3EPlhBZshlXmVBoXxAoGBAI5jSnorNOJ5VOTZhApH1DMYWqSd2z4CSW0daPfk680jb3095XhR+MbXYd0IWs5cXi0Hek7usAaKhGnRscppLTXJ3aVA5AgZlzneMCZMJdAa1e0/yngicsJ2EFIUauB3sybnZLfsZo0jTGGyIBFBmFQDNIU7P4v+ZDC3nhT/WC7G
    alipayPublicKey: MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAjrEVFMOSiNJXaRNKicQuQdsREraftDA9Tua3WNZwcpeXeh8Wrt+V9JilLqSa7N7sVqwpvv8zWChgXhX/A96hEg97Oxe6GKUmzaZRNh0cZZ88vpkn5tlgL4mH/dhSr3Ip00kvM4rHq9PwuT4k7z1DpZAf1eghK8Q5BgxL88d0X07m9X96Ijd0yMkXArzD7jg+noqfbztEKoH3kPMRJC2w4ByVdweWUT2PwrlATpZZtYLmtDvUKG/sOkNAIKEMg3Rut1oKWpjyYanzDgS7Cg3awr1KPTl9rHCazk15aNYowmYtVabKwbGVToCAGK+qQ1gT3ELhkGnf3+h53fukNqRH+wIDAQAB
    returnUrl: http://dhg8mu.natappfree.cc/alipay/return
    notifyUrl: http://dhg8mu.natappfree.cc/alipay/notify
```
3.支付代码  
3.1properties配置代码
```
@Data
@Slf4j
@ConfigurationProperties(prefix = "pay.alipay")
public class AlipayProperties {
    /** 支付宝网关 */
    private String gatewayUrl;
    /** 商户应用id */
    private String appid;
    /** RSA私钥,用于对商户请求报文加签 */
    private String privateKey;
    /** 支付宝RSA公钥,用于对支付宝应答的验签 */
    private String alipayPublicKey;

    /** 签名类型 */
    private String signType = "RSA2";

    /** 格式 */
    private String formate = "json";
    /** 编码 */
    private String charset = "UTF-8";

    /** 同步地址 */
    private String returnUrl;
    /** 异步地址 */
    private String notifyUrl;

    /** 最大查询次数 */
    private static int maxQueryRetry = 5;
    /** 查询间隔 */
    private static long queryDuration = 5000;
    /** 最大撤销次数 */
    private static int maxCancelRetry = 3;
    /** 撤销间隔(毫秒) */
    private static long cancelDuration = 3000;
}

@Configuration
@EnableConfigurationProperties(AlipayProperties.class)
public class AlipayConfiguration {
    @Autowired
    private AlipayProperties properties;

    @Bean
    public AlipayTradeService alipayTradeService() {
        return new AlipayTradeServiceImpl.ClientBuilder()
                .setGatewayUrl(properties.getGatewayUrl())
                .setAppid(properties.getAppid())
                .setPrivateKey(properties.getPrivateKey())
                .setAlipayPublicKey(properties.getAlipayPublicKey())
                .setSignType(properties.getSignType())
                .build();
    }
}
```
3.2具体业务代码
```
@Slf4j
@RestController
@RequestMapping("/alipay/f2fpay")
public class AlipayF2FPayController {
    @Autowired
    private AlipayTradeService alipayTradeService;

    @Autowired
    private AlipayProperties properties;

    /**
     * 当面付 -- 条码支付
     * 适用于 用户出示付款码,商家使用扫码工具获取到付款码信息
     * @param authCode
     * @return
     */
    @PatchMapping("/barCodePay")
    public String barCodePay(String authCode) {
        // (必填) 商户网站订单系统中唯一订单号，64个字符以内，只能包含字母、数字、下划线
        String outTradeNo = UUID.randomUUID().toString();

        // (必填) 订单标题，粗略描述用户的支付目的。如"xxx品牌xxx门店消费"
        String subject = "xxx品牌xxx门店当面付消费";

        // (必填) 订单总金额，单位为元，不能超过1亿元
        // 如果同时传入了【打折金额】,【不可打折金额】,【订单总金额】三者,则必须满足如下条件:【订单总金额】=【打折金额】+【不可打折金额】
        String totalAmount = "0.01";

        // (必填) 商户门店编号，通过门店号和商家后台可以配置精准到门店的折扣信息，详询支付宝技术支持
        String storeId = "test_store_id";

        // 订单描述，可以对交易或商品进行一个详细地描述，比如填写"购买商品3件共20.00元"
        String body = "购买商品3件共20.00元";

        // 商户操作员编号，添加此参数可以为商户操作员做销售统计
        String operatorId = "test_operator_id";

        // 支付超时，线下扫码交易定义为5分钟
        String timeoutExpress = "5m";

        // 商品明细列表,需填写购买商品详细信息
        List<GoodsDetail> goodsDetailList = new ArrayList<>();
        // 创建一个商品信息，参数含义分别为商品id（使用国标）、名称、单价（单位为分）、数量，如果需要添加商品类别，详见GoodsDetail
        GoodsDetail goods1 = GoodsDetail.newInstance("goods_id001", "xxx面包", 1000, 1);
        // 创建好一个商品后添加至商品明细列表
        goodsDetailList.add(goods1);

        // 继续创建并添加第一条商品信息，用户购买的产品为“黑人牙刷”，单价为5.00元，购买了两件
        GoodsDetail goods2 = GoodsDetail.newInstance("goods_id002", "xxx牙刷", 500, 2);
        goodsDetailList.add(goods2);

        // 创建条码支付请求builder,设置请求参数
        AlipayTradePayRequestBuilder builder = new AlipayTradePayRequestBuilder().setOutTradeNo(outTradeNo)
                .setSubject(subject).setAuthCode(authCode).setTotalAmount(totalAmount)
                .setStoreId(storeId).setBody(body).setOperatorId(operatorId)
                .setGoodsDetailList(goodsDetailList).setTimeoutExpress(timeoutExpress);

        // 调用tradePay方法获取当面付应答
        AlipayF2FPayResult result = alipayTradeService.tradePay(builder);
        // 此处必须加,因为不是每次支付都会成功
        switch (result.getTradeStatus()) {
            case SUCCESS:
                log.info("支付宝支付成功:");
                break;

            case FAILED:
                log.error("支付宝支付失败!!!");
                break;

            case UNKNOWN:
                log.error("系统异常，订单状态未知!!!");
                break;

            default:
                log.error("不支持的交易状态，交易返回异常!!!");
                break;
        }

        return result.getResponse().getBody();
    }
}
```

### 扫码支付示例
#### 场景介绍
&emsp;&emsp;扫码支付指用户打开支付宝钱包中的"扫一扫"功能,扫描商户针对每个订单实时生成的订单二维码,并在手机端确认支付.
#### 调用流程
![扫码支付调用流程](/img/in-post/SpringBoot/Alipay/扫码支付调用流程.jpg)  
1.商户系统调用支付宝预下单接口alipay.trade.precreate,获得该订单二维码图片地址  
2.发起轮询获得支付结果：等待5秒后调用[交易查询接口alipay.trade.query](https://docs.open.alipay.com/api_1/alipay.trade.query)
通过支付时传入的商户订单号(out_trade_no)查询支付结果(返回支付查询TRADE_STATUS),如果仍然返回等待用户付款(WAIT_BUYER_PAY),则再次等待5秒后继续查询,
直到返回确切的支付结果(成功TRADE_SUCCESS 或 已撤销关闭TRADE_CLOSED),或是超出轮询时间.在最后一次查询仍然返回等待用户付款的情况下,必须立即调用[交易撤销接口alipay.trade.cancel](https://docs.open.alipay.com/api_1/alipay.trade.cancel)
将这笔交易撤销,避免用户继续支付  
3.除了主动轮询,也可以通过接受异步通知获得支付结果,详见[扫码异步通知](https://docs.open.alipay.com/194/103296),注意一定要对异步通知做验签,确保通知是支付宝发出的
#### 扫码示例代码
##### 使用SDK快速接入
```
AlipayClient alipayClient = new DefaultAlipayClient("https://openapi.alipay.com/gateway.do", APP_ID, APP_PRIVATE_KEY, "json", CHARSET, ALIPAY_PUBLIC_KEY, "RSA2"); //获得初始化的AlipayClient
AlipayTradePrecreateRequest request = new AlipayTradePrecreateRequest();//创建API对应的request类
request.setBizContent("{" +
"    \"out_trade_no\":\"20150320010101002\"," +
"    \"total_amount\":\"88.88\"," +
"    \"subject\":\"Iphone6 16G\"," +
"    \"store_id\":\"NJ_001\"," +
"    \"timeout_express\":\"90m\"}");//设置业务参数
AlipayTradePrecreateResponse response = alipayClient.execute(request);
System.out.print(response.getBody());
//根据response中的结果继续业务逻辑处理
```
##### 利用官方Demo使用SpringBoot方式接入
依赖及配置同上方的条码支付
具体业务代码
```
/**
 * 当面付 -- 扫码付
 * 适用于商家根据每笔订单产生二维码,用户使用"扫一扫"扫码付
 * @param request
 * @param response
 */
@PostMapping("/precreate")
public void precreate(HttpServletRequest request, HttpServletResponse response) {
	// (必填) 商户网站订单系统中唯一订单号，64个字符以内，只能包含字母、数字、下划线，
	String outTradeNo = UUID.randomUUID().toString();

	// (必填) 订单标题，粗略描述用户的支付目的。如"xxx品牌xxx门店消费"
	String subject = "xxx品牌xxx门店当面付消费";

	// (必填) 订单总金额，单位为元，不能超过1亿元
	// 如果同时传入了【打折金额】,【不可打折金额】,【订单总金额】三者,则必须满足如下条件:【订单总金额】=【打折金额】+【不可打折金额】
	String totalAmount = "0.01";

	// (必填) 商户门店编号，通过门店号和商家后台可以配置精准到门店的折扣信息，详询支付宝技术支持
	String storeId = "test_store_id";

	// 订单描述，可以对交易或商品进行一个详细地描述，比如填写"购买商品3件共20.00元"
	String body = "购买商品3件共20.00元";

	// 商户操作员编号，添加此参数可以为商户操作员做销售统计
	String operatorId = "test_operator_id";

	// 支付超时，线下扫码交易定义为5分钟
	String timeoutExpress = "5m";

	// 商品明细列表,需填写购买商品详细信息
	List<GoodsDetail> goodsDetailList = new ArrayList<>();
	// 创建一个商品信息，参数含义分别为商品id（使用国标）、名称、单价（单位为分）、数量，如果需要添加商品类别，详见GoodsDetail
	GoodsDetail goods1 = GoodsDetail.newInstance("goods_id001", "xxx面包", 1000, 1);
	// 创建好一个商品后添加至商品明细列表
	goodsDetailList.add(goods1);

	// 继续创建并添加第一条商品信息，用户购买的产品为“黑人牙刷”，单价为5.00元，购买了两件
	GoodsDetail goods2 = GoodsDetail.newInstance("goods_id002", "xxx牙刷", 500, 2);
	goodsDetailList.add(goods2);

	AlipayTradePrecreateRequestBuilder builder = new AlipayTradePrecreateRequestBuilder()
			.setSubject(subject).setTotalAmount(totalAmount).setOutTradeNo(outTradeNo)
			.setBody(body).setOperatorId(operatorId).setStoreId(storeId)
			// 支付宝主动通知商户服务器指定页面http
			.setTimeoutExpress(timeoutExpress).setNotifyUrl(properties.getNotifyUrl())
			.setGoodsDetailList(goodsDetailList);

	AlipayF2FPrecreateResult result = alipayTradeService.tradePrecreate(builder);
	switch (result.getTradeStatus()) {
		case SUCCESS:
			log.info("支付宝预下单成功: )");
			break;

		case FAILED:
			log.error("支付宝预下单失败!!!");
			break;

		case UNKNOWN:
			log.error("系统异常，预下单状态未知!!!");
			break;

		default:
			log.error("不支持的交易状态，交易返回异常!!!");
			break;
	}
}
```
